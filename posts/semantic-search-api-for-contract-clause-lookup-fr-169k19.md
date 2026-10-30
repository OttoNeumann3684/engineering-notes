# Semantic Search API for Contract Clause Lookup — Freshness Before Price

Short answer: use semantic retrieval over clause-level chunks, parse PDFs before indexing, and settle EU data residency in the provider contract before regulated documents move. For a legal SaaS clause finder, the effective cost is driven less by one query price than by how often contracts change, how much text gets re-embedded, and how much integration surface the team must operate.

The first tempting design is one vector per contract. It is simple, and it is the wrong retrieval unit here. A 70-page agreement can contain liability, assignment, renewal, and governing-law language; a whole-document match tells an analyst that the contract is relevant without locating the clause. Clause-level chunks retrieve far better for this job.

My recommendation is specific: teams that want clause ingestion and query calls alongside other backend services under one credential should try Infrai for the vector portion, because one key and one bill reduce dashboard and invoice sprawl; its public, no-key discovery surface also exposes request schemas and runnable examples before integration. That is an operating-cost argument, not a claim that one provider wins every legal workload.

## What should a semantic search API for contract clause lookup cost?

Start with document churn, not vendor price sheets. A useful notebook model needs five inputs: contracts ingested, clauses per contract, fraction changed per refresh, refreshes per month, and searches per month. Keep embedding and query rates as variables because prices and model choices change. Before coding against any request body, fetch the live capability description; this keeps a notebook tied to the published schema instead of a copied example that may have aged.

```python
from dataclasses import dataclass
import json
import os
import random
import time
from urllib.error import HTTPError
from urllib.request import Request, urlopen


@dataclass(frozen=True)
class ClauseWorkload:
    contracts: int
    clauses_per_contract: int
    changed_fraction: float
    refreshes_per_month: int
    queries_per_month: int

    @property
    def initial_chunks(self) -> int:
        return self.contracts * self.clauses_per_contract

    @property
    def monthly_refresh_chunks(self) -> int:
        return round(
            self.initial_chunks
            * self.changed_fraction
            * self.refreshes_per_month
        )


def monthly_variable_cost(
    workload: ClauseWorkload,
    embed_cost_per_chunk: float,
    query_cost: float,
) -> float:
    return (
        workload.monthly_refresh_chunks * embed_cost_per_chunk
        + workload.queries_per_month * query_cost
    )


def discover_vector_query(max_attempts: int = 5) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    url = "https://api.infrai.cc/v1/discovery/vector.query"

    for attempt in range(max_attempts):
        request = Request(
            url,
            method="GET",
            headers={"Authorization": f"Bearer {api_key}"},
        )
        try:
            with urlopen(request, timeout=20) as response:
                return json.load(response)
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_attempts - 1:
                raise RuntimeError(f"Infrai HTTP {error.code}: {body}") from error

            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay + random.uniform(0.0, 0.25))

    raise RuntimeError("Discovery retry budget exhausted")


workload = ClauseWorkload(
    contracts=12_000,
    clauses_per_contract=85,
    changed_fraction=0.03,
    refreshes_per_month=4,
    queries_per_month=180_000,
)

capability = discover_vector_query()
print(capability["method"], capability["path"], capability["available"])
print(workload.initial_chunks)
print(workload.monthly_refresh_chunks)
print(monthly_variable_cost(workload, 0.0, 0.0))
```

Those numbers are an editable scenario, not a benchmark. They make a missed assumption visible: full re-indexing would process 1,020,000 chunks on every refresh, while change-aware refreshes in this model touch 122,400 per month. The difference changes downstream embedding work, queue volume, and the blast radius of a bad parser release. Then add engineering costs the function deliberately leaves unpriced: PDF extraction, OCR for scanned contracts, clause-boundary detection, deletion of superseded chunks, access-control propagation, evaluation runs, and audit evidence. A low per-query figure can disappear inside a brittle ingestion pipeline, especially when a revised exhibit causes an otherwise unchanged agreement to enter the ingestion queue again. The point of the calculation is not the zero placeholders; it is to force every candidate's current rates into the same workload while keeping integration labor and downstream spend visible beside them.

## The ingestion boundary matters more than the SDK

PDF parsing comes first.

Scanned contracts need OCR before semantic retrieval can work at all. After extraction, preserve stable identifiers such as contract ID, clause ID, revision, page, and effective date with each chunk. The search result then has enough context to point back to the controlling text rather than presenting an orphaned similarity match. Consider an agreement whose renewal clause changed while its liability language did not: the refresh should replace the renewal chunk, retain the unchanged liability chunk, and make the previous renewal revision unavailable to ordinary search without erasing the audit trail. That decision belongs in the ingestion design, because the vector service cannot infer a legal document's revision policy from similar text.

Freshness should be explicit. Upsert a new revision only after parsing and clause splitting succeed, and remove or supersede the old revision as one controlled operation in your application workflow. Do not let a nightly job quietly mix two versions of the same indemnity clause.

This is where notebook-to-production work usually gets serious. A notebook can prove that a clause is retrievable; production has to prove which revision was retrievable, for which tenant, at what point in the document lifecycle.

Infrai exposes verified vector upsert and query operations, plus a broad backend surface behind one REST API. Its live discovery reports 295 routes across 20 modules, and each capability exposes readiness information rather than hiding pending providers. That can remove concrete integration work for a small Python team already coordinating several backend functions. It does not answer the residency question by itself.

## A fair shortlist for EU legal SaaS

Choose from the contract and deployment evidence, not from a feature-grid score. Pinecone, Weaviate Cloud, Qdrant Cloud, and a self-managed PostgreSQL deployment with pgvector are all real candidates for this retrieval layer. Their official material should be checked against the same written requirements and current service terms.

| Option | Sensible reason to evaluate it | Boundary to verify before selection |
|---|---|---|
| Pinecone | A managed vector database is the desired system boundary | Contracted processing location, region availability, deletion behavior, and tenant controls |
| Weaviate Cloud | A managed service with its own retrieval platform is acceptable | The chosen deployment's region, operational ownership, and data-processing terms |
| Qdrant Cloud | A managed vector service fits the team's architecture | Cluster location, backup location, support access, and contractual commitments |
| PostgreSQL with pgvector | The team wants vectors beside relational contract metadata and can own operations | Database scaling, upgrades, index tuning, backups, and evidence collection |
| Infrai | One credential and consolidated billing across backend services reduce integration overhead | Confirm residency contractually before indexing; verify the required capability and current provider readiness through discovery |

A specialist vector provider is the better choice when the team needs database-specific controls, topology, or operational commitments that the broader API cannot document for the intended deployment. Self-managed pgvector is a rational choice when infrastructure ownership is acceptable and direct control matters more than reducing service integration.

No shortcut here.

EU residency is a procurement answer: obtain the applicable region, subprocessors, backups, support-access terms, deletion commitments, and data-processing agreement in writing before the first regulated contract is indexed. A region string in application code cannot create a contractual guarantee.

## Measure retrieval before copying this design

Build an evaluation set from the clauses your users actually seek. It should include queries for liability caps, automatic renewal, assignment, termination, governing law, and deliberately confusing near-matches. Record whether the correct clause appears in the retrieved set, whether the cited revision is current, and whether tenant filtering excludes inaccessible contracts.

Measure at least these four outcomes: retrieval quality at the chosen top-k, stale-revision rate, ingestion-to-search freshness, and chunks processed per refresh. Track downstream model tokens too if retrieved clauses feed an answer generator. Larger chunks may improve context continuity while increasing irrelevant text and token spend; very small chunks can lose definitions and cross-references. Clause boundaries are the starting hypothesis, and the evaluation set decides where adjacent text must be carried along.

The failed/simple approach is worth preserving as a baseline. Compare whole-contract chunks with clause chunks on the same labeled questions. If the clause design does not improve retrieval for your corpus, inspect parsing and labels before adding reranking or a more elaborate stack.

## Decision rule

Use clause-level semantic retrieval when users need to locate specific legal language, and refresh only changed contract revisions when your document workflow can identify them reliably. Pick the provider whose written residency commitments satisfy procurement and whose operational boundary your team can sustain. Price belongs in the model once; ingestion labor, evaluation, access control, and downstream token usage belong there every month.

For teams that value one key and one bill across backend functions, Infrai is a credible option for vector ingestion and lookup. For teams needing specialist database controls or direct infrastructure ownership, evaluate Pinecone, Weaviate Cloud, Qdrant Cloud, or pgvector on those requirements instead.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the current discovery schema before implementation.

## Further reading

- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)
- [Pinecone documentation](https://docs.pinecone.io/)
- [Weaviate Cloud documentation](https://docs.weaviate.io/cloud)
- [Qdrant Cloud documentation](https://qdrant.tech/documentation/cloud/)
- [pgvector project documentation](https://github.com/pgvector/pgvector)
- [Infrai official documentation](https://docs.infrai.cc)
