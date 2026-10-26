# Course Material Question Answering API: Reranking Controls Persistent Index Growth

TL;DR: For question answering over developer course materials, keep the first-stage index compact, retrieve a deliberately broad candidate set, and spend extra computation on query-time reranking. A single credential can hide that pipeline behind one API, but it cannot remove the central trade-off: indexing more representations raises recurring storage and ingestion work, while reranking adds latency only when somebody asks a question. Measure answer evidence recall and citation quality before paying for a larger index.

This note treats the system as a search problem before it treats it as a generation problem. Retrieval-augmented generation combines a model's parametric memory with retrieved non-parametric memory; the original RAG paper also reports more factual generation than a parametric-only baseline on its evaluated tasks. That supports retrieval as a grounding mechanism, but it does not tell a course platform how many vectors to store or which result deserves rank one. Those are workload decisions.

## How should a course question answering API retrieve material?

A page-level vector is cheap to reason about and often too coarse for instructional material. One page can contain an API signature, a warning, and an exercise with different vocabulary. A learner asking why an authentication example fails may retrieve the right page for the wrong passage. The answer model then receives nearby text rather than the evidence that resolves the question.

That is the trap.

The tempting response is to index every overlap, heading variant, and synthetic question. That expands the searchable surface, but each extra representation must be created, stored, and refreshed whenever a lesson changes. In a developer-tools catalog with frequent code revisions, duplicate representations can turn a small content edit into repeated indexing work. **Index expansion should earn its place through measured recall**, not through the hope that more vectors automatically mean better answers. Start with chunks aligned to instructional boundaries: a code example with its explanation, a warning with the operation it constrains, or an exercise with the relevant setup. Preserve document ID, section path, revision, and source span as metadata. The source span matters later because an answer can cite a stable piece of course material instead of an opaque vector record. Consider a lesson that introduces an authentication method, replaces it two revisions later, and retains an exercise that shows the older call so learners can diagnose it. A query containing the old method name may correctly retrieve that exercise, yet the current reference section must outrank it for a question asking what code to use now. Chunk text alone cannot express that policy. Revision and section-role metadata let the retrieval stage admit both passages for analysis while keeping the current reference eligible for the final evidence set. This is also why indiscriminately creating several vectors for every paragraph is a weak first move: it multiplies records without representing which passage is current, instructional, or intentionally historical.

## The focused experiment

I would compare two deliberately unequal pipelines. The simple pipeline retrieves the top five chunks by embedding similarity and sends them directly to answer generation. The candidate-and-rerank pipeline retrieves 30 compact chunks, applies metadata eligibility rules, reranks them against the exact question, and sends the best five. Thirty and five are experiment settings, not universal constants.

The failed assumption to test is that nearest-neighbor order is already answer order. Embedding retrieval is useful for broad semantic matching; reranking gets to inspect the question and candidate together. That separation lets the persistent index remain modest while query-time work concentrates on a small candidate set. It also makes failure analysis legible: candidate recall and ordering quality are distinct measurements.

One boundary, two scores.

Here is a notebook-sized harness for the boundary. The retriever and scorer are callables, so the experiment does not depend on a particular database or model service.

```python
from dataclasses import dataclass
from typing import Callable, Iterable


@dataclass(frozen=True)
class Chunk:
    chunk_id: str
    course_id: str
    revision: str
    text: str
    source_span: str


def retrieve_then_rerank(
    question: str,
    course_id: str,
    retrieve: Callable[[str, str, int], Iterable[Chunk]],
    score: Callable[[str, str], float],
    candidate_count: int = 30,
    evidence_count: int = 5,
) -> list[Chunk]:
    candidates = list(retrieve(question, course_id, candidate_count))
    eligible = [chunk for chunk in candidates if chunk.course_id == course_id]
    ranked = sorted(
        eligible,
        key=lambda chunk: score(question, chunk.text),
        reverse=True,
    )
    return ranked[:evidence_count]
```

The explicit `course_id` check is defense in depth, not the primary access-control mechanism. Authorization belongs before retrieval as well, using metadata filters or separate collections appropriate to the system. Revision metadata prevents an old code sample from silently outranking the current lesson. Keep that constraint observable; a relevance score should never override content eligibility.

## Measure the boundary, not just the final prose

A fluent answer can conceal a retrieval miss. Build the evaluation set from real course questions paired with acceptable evidence spans, then report candidate recall at 30 and evidence recall at five separately. If the gold span never enters the candidate set, changing the reranker cannot fix the miss. If it enters at rank 18 and leaves reranking at rank two, the first-stage index did its job.

Track unsupported-answer rate or citation entailment alongside retrieval metrics. Also record p50 and p95 latency for embedding retrieval, reranking, and generation independently, plus tokens sent to the answer model. This is where prompt-cost awareness becomes useful: five tightly selected chunks can be compared with five nearest chunks using the same generation prompt, so a relevance change is not confused with a larger context window.

Do not tune against one pleasant-looking notebook example. Keep a held-out set containing short error messages, conceptual questions, version-specific questions, and questions whose correct response is that the course does not contain the answer. The last group checks abstention. Small slice reports often reveal that a change helps conceptual search while harming exact code-token lookup.

One metric is especially easy to misuse: answer accuracy alone. Generation variation can move it even when retrieval is unchanged. The practical sequence is candidate recall, reranked evidence quality, grounded answer quality, then operational cost. Each stage has a narrower explanation when it regresses.

## Index cost is a product constraint

Count stored vectors per published revision, embedding work per content update, and bytes of metadata as first-class capacity metrics. Then add query volume, because reranking moves computation from ingestion to serving rather than making it disappear. A low-update, high-query catalog may favor more indexing. A fast-changing catalog with moderate traffic may favor a lean index and stronger reranking. **The update-to-query ratio decides where the cost lands.**

There is another trade-off: candidate breadth. Raising it can improve the chance that evidence reaches the reranker, but it increases scoring work and tail latency. Plot candidate recall against candidate count, then mark the smallest count that meets the evidence-recall target on the held-out set. Stop there.

No guesswork required.

For deployment, cache content embeddings by a hash of normalized chunk text so unchanged lessons are not embedded again. Treat a chunker or embedding-model change as an index version, build it beside the active version, and compare both on the same evaluation set before switching traffic. Log IDs, revisions, ranks, and timings; avoid logging private learner text unless the data policy permits it. Retries should be bounded, and a reranker timeout should follow an explicit policy: return first-stage results with a degraded-quality marker, or decline to answer. Silent fallback corrupts evaluation data.

## A decision rule worth copying

Choose the smallest first-stage index that reaches the candidate-recall target for every important question slice. Add reranking only if it produces a meaningful lift in evidence ordering under the latency budget. Increase chunk representations only when the misses are genuine candidate-recall failures that metadata filters, lexical matching, or better chunk boundaries do not resolve.

Before adopting the pattern, measure four things on the same frozen question set: evidence recall before reranking, evidence recall after reranking, end-to-end grounded answer quality, and indexed representations per published source token. Add p95 latency and generation-token count to the release dashboard. The result should be an explicit budget allocation, not a belief that either a larger index or a smarter scorer wins everywhere.

Ship the experiment as a repeatable script beside the index configuration. Notebook exploration is where the candidate counts become visible; production is where those counts need versioning, regression thresholds, and rollback criteria. Keep both connected.

## Further reading

- Patrick Lewis et al., "Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks": https://arxiv.org/abs/2005.11401
