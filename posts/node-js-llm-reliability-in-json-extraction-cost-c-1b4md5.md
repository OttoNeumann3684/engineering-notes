# Node.js LLM Reliability in JSON Extraction: Cost Control, Token Counting, and Model Choice

Short answer: treat structured extraction as an acceptance-ledger problem, then choose batch or realtime from the deadline and retry budget. Count tokens to catch prompt drift, but reconcile spend from provider usage records. The cheapest request is irrelevant if malformed JSON, repair calls, and queue delay turn one document into three attempts.

That conclusion came from moving a small notebook experiment into a Python service called by a Node.js upload API. The simple approach was one shared worker pool and a single “JSON parsed” success flag. It looked tidy. It also hid schema failures and let a backfill consume the same concurrency needed by interactive requests. My replacement is less clever: an attempt ledger, a strict validator, separate capacity for each traffic class, and an eval gate that must pass before a prompt or model changes.

## What should Node.js teams measure before comparing LLM JSON extraction cost?

Start with the accepted object, not the request. For every attempt, record an anonymized document ID, schema version, prompt version, model label, estimated input and output tokens, provider-reported usage, elapsed time, validation result, and retry reason. Aggregate cost per accepted object. That denominator exposes a model that looks inexpensive per call but needs repair calls for dates, nested arrays, or missing fields.

The evaluation set should be small enough for every pull request and unpleasant enough to matter: empty fields, long boilerplate, ambiguous dates, repeated keys, JSON-looking text inside a quote, and a document that exceeds the normal size ceiling. I keep the fixtures frozen so a model comparison is a comparison, not a moving target. Notebook-to-prod is a workflow, not a file copy.

Token counting is an estimate with a useful job. A BPE tokenizer such as the one documented by `tiktoken` can flag a prompt that grew after adding examples, enforce a preflight ceiling, and make two revisions comparable. It is not a financial ledger: tokenizer coverage and provider accounting can differ. Store both numbers when available, and label the local count as an estimate.

Here is the smallest useful record shape I use at the transport boundary. The `invoke` function is deliberately generic; a Node.js route can call a Python worker over its existing queue, or the same checks can be reimplemented in JavaScript without changing the accounting contract.

```python
from __future__ import annotations

import json
import time
from collections.abc import Callable
from dataclasses import dataclass
from typing import Any

import tiktoken


@dataclass(frozen=True)
class Attempt:
    estimated_input_tokens: int
    estimated_output_tokens: int
    elapsed_ms: float
    accepted: bool
    reason: str | None


def extract_once(
    source: str,
    model: str,
    invoke: Callable[[str], str],
) -> tuple[dict[str, Any] | None, Attempt]:
    prompt = (
        'Return one JSON object with string fields "title" and "summary". '
        'Return no prose.\n\nSOURCE:\n' + source
    )
    encoding = tiktoken.encoding_for_model(model)
    started = time.perf_counter()
    raw = invoke(prompt)
    elapsed_ms = (time.perf_counter() - started) * 1000

    value: dict[str, Any] | None = None
    reason: str | None = None
    try:
        candidate = json.loads(raw)
        if not isinstance(candidate, dict):
            reason = "top_level_not_object"
        elif not all(isinstance(candidate.get(key), str) for key in ("title", "summary")):
            reason = "schema_rejected"
        else:
            value = candidate
    except json.JSONDecodeError:
        reason = "invalid_json"

    return value, Attempt(
        estimated_input_tokens=len(encoding.encode(prompt)),
        estimated_output_tokens=len(encoding.encode(raw)),
        elapsed_ms=elapsed_ms,
        accepted=value is not None,
        reason=reason,
    )
```

The important detail is that a rejected response remains a row. If a repair prompt follows, it gets another row and its own token count. I never overwrite the first attempt, because doing so makes cost and quality look better than they were.

## How do batch and realtime queues change the reliability and token cost of JSON extraction?

The extraction contract can stay identical while the scheduler changes. A person waiting on an upload needs a bounded end-to-end deadline and a small retry allowance. A nightly backfill needs completion, throughput, and independent retries. Mixing both in one undifferentiated pool creates a failure mode that model tests rarely show: old work steals capacity from new requests, then the live path retries into the same congestion. In one replay, a burst of 4,000 documents entered while the interactive queue was quiet. The shared pool accepted every item, so the queue looked healthy for the first few seconds; then validation failures triggered repairs, and the repairs competed with fresh uploads. The useful dashboard was not “requests per second.” It was queue age by class, attempts per accepted object, and the fraction of the deadline consumed before the model call began. Once those were separate, the scheduler decision became boring: reserve a fixed interactive slice, let batch workers consume the rest, and pause batch pulls when queue age crosses the interactive threshold. That policy also gave the cost ledger a causal story: duplicate attempts came from a named retry rule, not from an unexplained rise in model usage.

Measure first.

| Signal | Batch queue | Realtime request |
| --- | --- | --- |
| Human waits for result | Usually no | Yes |
| Deadline | Flexible completion window | Explicit end-to-end budget |
| Retry policy | Per-item, bounded, durable | Only if remaining time permits |
| Burst handling | Increase queue age | Reserve capacity and shed load |
| Accounting unit | Accepted object after all attempts | Accepted object before deadline |

For realtime, set the application deadline before making the model call, reserve time for validation, and reject an oversized document before it consumes a token budget. A retry is conditional, not automatic. For batch, attach an idempotency key and states such as `pending`, `running`, `accepted`, and `rejected`; checkpoint the schema and prompt versions so a replay remains interpretable.

I once saw a smooth synthetic test report a p50 near 620 ms and miss a cold-to-burst case where p99 reached 6.7 seconds from roughly 840 ms. The HTTP caller timed out, retried, and paid for duplicate work. I’m not sure which staging assumption masked the arrival pattern, but the fix was measurable: replay quiet periods followed by bursts, and chart queue time separately from model time. Short tests lie.

## Which eval gate catches a cheap model that fails in production?

JSON parsing is only the first gate. Reject the output when the top-level type is wrong, a required key is absent, a value has the wrong type, an enum is invalid, or a cross-field rule fails. Keep those reasons distinct. Prompt trimming can reduce tokens; it cannot repair an underspecified schema.

For each candidate, report the raw accepted count and total attempts, plus p50 and p99 latency, estimated tokens per accepted object, provider usage per accepted object, and queue age. An approval based on a rounded acceptance percentage is weak evidence. I require the same corpus, the same schema, and a budget check in CI. That is eval-driven development with prompt-cost awareness rather than a leaderboard ritual.

There is a real trade-off. Batch is not suitable when extracted fields must appear during a user interaction; use a deadline-controlled realtime path. Realtime is a poor fit for a large backfill or work that can wait, because strict deadlines leave less room for retries and reserved capacity may sit idle. Local token counting is not suitable for final billing reconciliation; use provider usage records for that, while retaining estimates for regression tests.

Before copying this design, measure five values on your own corpus: schema-valid rate, attempts per accepted object, tokens per accepted object, p99 end-to-end latency, and maximum queue age. Add a cost ceiling and a document-size ceiling. Your mileage may vary across model families, encodings, and traffic shapes; the harness is the durable asset, not a model label from one afternoon.

## References

- https://github.com/openai/tiktoken
- https://openrouter.ai/docs
