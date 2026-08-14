# FastAPI RAG Hallucination: Evidence Gates for Embeddings, Chunking, and Context Windows

Use a decision rule before tuning anything: if the retrieved passages do not directly support a healthtech support answer, abstain and route the ticket to a person. Better embeddings, smaller chunks, and a larger context window cannot repair missing evidence.

That rule changes the build from “search, then always answer” into “retrieve, verify, then answer or defer.” For incoming tickets, the data flow is plain: classify the request, apply access and metadata filters, retrieve candidate passages, test whether those passages cover the requested facts, and let the model draft only from the accepted evidence. Keep the provider boundary behind Python interfaces so an eval can compare implementations without changing the triage policy.

Short answer: wrong RAG answers usually need an evidence gate and an abstention path, not another round of blind embedding, chunking, or context-window tuning.

## Choose the failure budget before writing Python

Start by separating three outcomes that look identical in a chat transcript. Retrieval can miss the relevant policy. Retrieval can return a related policy that does not answer the ticket. Or the generator can ignore adequate evidence and add an unsupported detail. Calling all three “hallucination” hides the component that needs work.

| Trace stage | Observable result | Change to test | What will not prove the fix |
| --- | --- | --- | --- |
| Retrieval | Required passage is absent | Filters, query construction, or indexing | A fluent final answer |
| Evidence gate | Passages are related but required facts are absent | Coverage rules and abstention labels | A higher similarity score alone |
| Generation | Accepted evidence is complete but the draft adds a claim | Prompt constraint and claim-level citation check | More retrieved chunks |
| Input preparation | The ticket text does not preserve the customer's request | The upstream parser or transcription evaluation | A larger context window |

This matters in healthtech support because a semantically close passage may still be operationally wrong. A ticket asking how to correct an insurance identifier is not answered by a passage about changing a mailing address, even if both live in an “account updates” section. Chunk overlap does not establish that the retrieved text contains the required action, actor, prerequisite, and exception. A long context can actually make inspection harder because the useful sentence competes with adjacent material.

Treat the retrieved set as untrusted input — OWASP's LLM application guidance is a useful reason to preserve that boundary — and make support content carry explicit metadata. Useful fields include document revision, audience, workflow, jurisdiction, and effective status. Apply deterministic filters before semantic ranking. A retired procedure should not win because its wording resembles the ticket.

Then require coverage, not mere similarity. For each answerable intent, define the facts that evidence must contain. An address-change question might require an authorized actor and the actual procedure; an eligibility question may require scope and effective status. If one required fact is absent, the correct output is a deferral with the ticket and retrieved evidence attached.

Stop there.

## How can a RAG chatbot answer wrong despite embeddings and chunking?

The smallest useful implementation sits between retrieval and generation. It should not know which vector database, embedding provider, or language model produced its inputs. That keeps notebook experiments honest: the same gate runs in a batch eval, a FastAPI handler, and production triage.

The example below is deliberately narrow. A retriever supplies scored passages; a coverage checker supplies the required facts found in each passage. In a real system, that checker can begin as deterministic metadata and phrase rules, then graduate to a separately evaluated model. The important part is that “no supported answer” is a normal result rather than an exception.

```python
from dataclasses import dataclass
from typing import Protocol, Sequence


@dataclass(frozen=True)
class Passage:
    document_id: str
    revision: str
    text: str
    score: float
    covered_facts: frozenset[str]


@dataclass(frozen=True)
class RetrievalQuery:
    text: str
    audience: str
    active_revision: str


class Retriever(Protocol):
    def search(self, query: RetrievalQuery, limit: int) -> Sequence[Passage]: ...


@dataclass(frozen=True)
class EvidenceDecision:
    answerable: bool
    passages: tuple[Passage, ...]
    missing_facts: frozenset[str]


def gate_evidence(
    passages: Sequence[Passage],
    required_facts: frozenset[str],
    active_revision: str,
) -> EvidenceDecision:
    eligible = tuple(
        passage
        for passage in passages
        if passage.revision == active_revision
    )
    covered = frozenset().union(
        *(passage.covered_facts for passage in eligible)
    )
    missing = required_facts - covered
    return EvidenceDecision(
        answerable=bool(eligible) and not missing,
        passages=eligible,
        missing_facts=missing,
    )


def triage_ticket(
    ticket: str,
    retriever: Retriever,
    active_revision: str,
) -> EvidenceDecision:
    query = RetrievalQuery(
        text=ticket,
        audience="customer-support",
        active_revision=active_revision,
    )
    candidates = retriever.search(query, limit=8)
    return gate_evidence(
        passages=candidates,
        required_facts=frozenset({"authorized_actor", "procedure"}),
        active_revision=active_revision,
    )
```

Notice what is absent: there is no universal similarity threshold. A score has meaning only relative to a particular embedding model, index, corpus, and query distribution. Copying a threshold from a demo would create false confidence. Calibrate it on labeled tickets, record it with the retriever configuration, and still retain the fact-coverage test; similarity is a ranking signal, while coverage is the release condition.

The generator receives only `decision.passages` when `answerable` is true. Its prompt should require citations to passage identifiers and prohibit claims outside those passages. When the decision is false, the application returns a stable handoff state that names `missing_facts` for internal observability without pretending to answer the customer.

Don't turn the generator into the gate.

If the same model both invents an answer and judges whether that answer is supported, correlated failures are hard to see. Keep the retrieval record, gate decision, final response, and cited passage IDs as separate artifacts. This costs a little more storage and engineering effort, but it makes a bad response traceable.

## Reliability test: separate unsupported answers from avoidable deferrals

Build the eval set from the job, not from convenient documentation questions. For support triage, include answerable tickets, requests whose answer exists only in a retired revision, ambiguous requests, and unanswerable requests. Include paraphrases and compact ticket language. If voice messages are transcribed before indexing or querying, preserve the transcript boundary too; the open-source Whisper repository describes speech recognition and related language tasks, but transcription quality and RAG evidence quality remain different stages to test.

Label the minimum evidence for every answerable case. Then calculate retrieval recall against those passage labels, gate precision against answerability labels, citation support against the final claims, and abstention behavior on unsupported tickets. The exact metric set can vary, but one blended score cannot tell the team whether to change the index, the gate, or the prompt. I'm not sure a model-based support judge is reliable for a given corpus until its decisions have been checked against human labels; disagreement review is what would resolve that uncertainty.

A practical eval runner should freeze the corpus revision, chunker configuration, embedding identifier, retrieval parameters, prompt version, and model identifier with every result. Compare one change at a time. If a new chunker raises retrieval recall but lowers evidence precision, that is a real trade-off rather than an automatic win. If a larger context increases prompt cost without improving citation support, keep the smaller context. The notebook can explore these variants quickly, but the saved cases and assertions are what make the result portable into CI.

One trap deserves extra space. Teams often inspect only tickets that received an answer, which makes a strict gate look excellent while hiding excessive deferral. Measure both sides: unsupported answers are dangerous, and unnecessary handoffs make the system unhelpful. Slice results by intent, document revision, source type, and access tier. A global average can conceal that one rare workflow always fails retrieval. No invented benchmark can choose the acceptable balance; the support owner and risk owner need to set it for the actual queue.

## Compare provider behavior with contract tests

Portability is more than swapping an SDK import. Embedding dimensions, score distributions, filtering syntax, ranking behavior, tokenization, and structured-output handling can differ across providers. The application contract should therefore expose domain inputs and outputs — query, eligible passages, evidence decision, cited draft — rather than provider response objects. Adapters can translate those contracts, while the eval harness decides whether a replacement preserves behavior.

This has a catch. A generic interface can hide provider-specific controls that materially improve retrieval, and maintaining adapters plus conformance tests takes work. It is not suitable when the team has a stable single-provider deployment, no migration requirement, and a specialized feature that the common contract would erase. In that case, keep the direct integration, but preserve the labeled eval set and evidence gate so a later migration has a behavioral baseline. Use the portable boundary when regulatory constraints, deployment regions, or procurement make replacement a real requirement — not as architecture theater.

FastAPI belongs at the edge, not in the policy. The route should authenticate the caller, validate the ticket, invoke the pipeline, and serialize either an evidence-backed draft or a handoff. Keeping the gate in ordinary Python makes batch evaluation possible without an HTTP server and prevents transport details from leaking into retrieval tests.

## Ship from trace evidence

Before release, read a sample of source documents and confirm that revision and audience metadata match how support actually works. Run the frozen eval suite, inspect every newly unsupported answer, and compare deferral slices rather than trusting the aggregate. Record corpus, retriever, gate, prompt, and generator versions together. In production, watch the rates of answer, abstention, missing-fact categories, citation presence, latency, and prompt usage; investigate distribution changes before loosening the gate. Re-run the suite whenever documentation, chunking, embeddings, retrieval parameters, prompts, or models change.

The operational decision stays simple: ship a change only when it preserves evidence support and keeps deferral within the queue's agreed boundary. Context windows are capacity. Embeddings are candidates. The gate is what turns retrieved text into permission to answer.

## References

- https://owasp.org/www-project-top-10-for-large-language-model-applications/
- https://github.com/openai/whisper
