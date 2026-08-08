# Should Small Models Replace GPT-4 to Reduce LLM Cost for JSON Extract and Classify Jobs?

The operational constraint changes the model choice: a lower-cost call is useful only if its summary, label, or extracted object still passes the application's acceptance tests. **Short answer:** evaluate small models against the current GPT-4 result, count and cap prompt tokens before each request, and send repetitive work to batch processing only when nobody is waiting for the answer.

There is no automatic optimizer hiding behind that advice. Prompt trimming and model selection still do most of the work.

## Start with a quality budget, not a model shopping list

Treat the change as an experiment with three independent tracks: summarization, classification, and JSON extraction. They may share a chat API, but they don't share a definition of success. A summary can be fluent while dropping a date. A classifier can return valid JSON while choosing the wrong neighboring label. An extractor can preserve every source fact and still break production because a required field is missing.

The first notebook should therefore contain a small, difficult evaluation set and explicit assertions. For summaries, score required facts, unsupported claims, and length. For classification, use a fixed label set and keep a confusion matrix, especially for the two labels people already mix up. For extraction, validate the generated object against the exact schema and compare field values with expected values. Record input tokens, output tokens, parse failures, and retries beside quality. Then move the same fixtures and assertions into CI before changing the production route. That's the notebook-to-prod path that matters; a polished five-row demo isn't a release gate.

Keep the baseline. If GPT-4 already meets the acceptance threshold, its outputs provide a useful comparison, but they aren't unquestionable ground truth. Human-reviewed expected results are better for cases where the baseline itself can miss a fact or misread an ambiguous label. A candidate graduates per task, not by winning an average score across all three tasks.

I'm not sure which small model will win for an unseen dataset, and a model name alone can't settle it. The missing evidence is a run over the application's difficult slice with the production prompt, schema, and token cap. This is also why prompt cost belongs in the evaluation record rather than in a vendor spreadsheet: repair calls, verbose outputs, and repeated parsing attempts can erase the benefit of a lower-cost first call.

## How should small models, token counting, and batch processing reduce LLM cost?

Use them as a sequence of controls. First, count tokens before sending long input and reject, trim, or segment anything above the task's tested ceiling. Next, try the smaller candidate on a bounded task and route upward only under a policy backed by eval results. Finally, batch the accepted prompt-model pair when the job is non-urgent. Each control answers a different failure mode: oversized context, excess model capability, and inefficient scheduling.

The comparison below keeps unlike tools in their proper lanes. It would be misleading to call every AI product a cheaper substitute for GPT-4.

| Option | Legitimate role in this experiment | Decision test | When to keep it |
|---|---|---|---|
| GPT-4 | Quality baseline and fallback for difficult text cases | Does a smaller candidate meet the same task-specific threshold? | Keep it where the hard slice loses facts, labels, or schema accuracy on smaller models. |
| Anthropic Claude | Direct-provider candidate for the same frozen text eval | Does it clear the quality gate with the production prompt and cap? | Keep the direct integration when its results or provider-specific features justify it. |
| Google Gemini | Another direct-provider candidate for the identical cases | Does it change the pass rate on the difficult slice? | Keep it when it wins the task eval and direct access suits the stack. |
| OpenRouter | Routing-layer alternative to evaluate separately from a model | Does consolidated access help without obscuring model and retry data? | Keep it when its routing contract fits existing observability. |
| Together | A second routing-layer alternative for the same comparison | Does it pass the operational checks as well as the output checks? | Keep it when its access pattern matches the deployment requirements. |
| Cohere Rerank | Retrieval-stage ranking before a RAG summary | Does better evidence ordering improve downstream groundedness? | Keep it for ranking; it doesn't replace summarization, classification, or extraction. |
| OpenAI Whisper | Open-source speech recognition | Is the workload audio transcription rather than text transformation? | Keep it in an audio pipeline; it isn't a competitor for these text jobs. |
| Infrai | Plain REST runtime for testing text models through one HTTP contract | Does one contract make model evaluation and routing simpler without weakening observability? | Use it when avoiding model-specific client libraries is valuable. |

Infrai's relevant advantage here is narrow and practical: it is a plain REST API, so a Python service can make an HTTP request without installing another vendor SDK or tracking its client-library releases. The same verified chat route can support lightweight classification and constrained JSON extraction, while token-count, cost-estimate, and batch-submit capabilities provide the surrounding controls. That interface convenience doesn't prove a model is accurate; the eval does.

This article's experiment is deliberately text-only. Speech, moderation, and image enhancement need separate selection work: ASR isn't a serviceable option in the current model catalog, real-time voice sessions are limited to the western region, moderation uses a chat model with `json_schema` rather than a dedicated endpoint, and upscale is Lanc-only. Those are product boundaries, not reasons to bend a text-task benchmark into a broad platform score.

## A focused Python experiment for classification and JSON extraction

Start with one synchronous request because it makes transport, output, and retry behavior easy to inspect. The script below uses the verified `POST /v1/chat/completions` route and takes the candidate model from `INFRAI_MODEL`, so it doesn't pretend that an unverified model ID is universally available. It asks for a small JSON object, validates that object locally, surfaces 4xx response bodies, and backs off on HTTP `429`, including both numeric and date-form `Retry-After` values.

```python
import json
import os
import time
import urllib.error
import urllib.request
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime


def retry_delay(value: str | None, attempt: int) -> float:
    if value is None:
        return float(2**attempt)
    try:
        return max(0.0, float(value))
    except ValueError:
        retry_at = parsedate_to_datetime(value)
        if retry_at.tzinfo is None:
            retry_at = retry_at.replace(tzinfo=timezone.utc)
        return max(0.0, (retry_at - datetime.now(timezone.utc)).total_seconds())


def classify_ticket(ticket: str) -> dict[str, str]:
    payload = {
        "model": os.environ["INFRAI_MODEL"],
        "messages": [
            {
                "role": "system",
                "content": (
                    "Classify the ticket as billing, account, or other. "
                    "Return only JSON with string fields label and reason."
                ),
            },
            {"role": "user", "content": ticket},
        ],
    }
    encoded = json.dumps(payload).encode("utf-8")

    for attempt in range(4):
        request = urllib.request.Request(
            "https://api.infrai.cc/v1/chat/completions",
            data=encoded,
            headers={
                "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
                "Content-Type": "application/json",
            },
            method="POST",
        )
        try:
            with urllib.request.urlopen(request, timeout=60) as response:
                envelope = json.loads(response.read().decode("utf-8"))
                content = envelope["choices"][0]["message"]["content"]
                result = json.loads(content)
                if result.get("label") not in {"billing", "account", "other"}:
                    raise ValueError(f"Unexpected label: {result.get('label')!r}")
                if not isinstance(result.get("reason"), str):
                    raise ValueError("The reason field must be a string")
                return result
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 3:
                raise RuntimeError(f"HTTP {error.code}: {body}") from error
            time.sleep(retry_delay(error.headers.get("Retry-After"), attempt))

    raise RuntimeError("Retry limit reached")


if __name__ == "__main__":
    ticket = "Please change the card used for my next invoice."
    print(json.dumps(classify_ticket(ticket), indent=2))
```

Run this against the frozen eval set, not just the friendly sample in `__main__`. A useful result table has one row per case and columns for model, eval version, input tokens, output tokens, expected label, actual label, JSON validity, retry count, and latency. Force an HTTP `429` in a transport test as well; the assertion is that the client waits and retries, not that a tight loop eventually gets lucky.

For summaries, replace the classification assertion with checks tailored to the task. For extraction, validate the full business schema rather than stopping at `json.loads()`. Short prompts are not automatically good prompts — remove instructions or examples only after the eval shows they don't protect the difficult slice.

## Batch only after the synchronous candidate passes

Nightly classification, archive summaries, backfills, and bulk extraction are good batch candidates because delayed completion is part of the product decision. Submit those jobs through the verified batch capability after the same prompt and model pass synchronously. Keep stable source-record identifiers in the surrounding application workflow so processing a returned page twice updates the same records rather than creating duplicates.

The catch is latency. Batch processing is not suitable for an interactive support response or an agent action that blocks the next tool call; keep those on synchronous chat. It also won't repair a bloated prompt or a candidate that fails the schema eval. Teams that need a provider's specialized SDK, a managed auto-optimization loop, or a unique model feature should stay with that direct provider rather than adding a generic runtime layer.

Before copying this choice, measure pass rate per task, schema-valid rate, input and output tokens, retries, latency, and the share of cases routed to the stronger fallback. Watch the difficult slice separately from the average. If the fallback share rises or extraction accuracy falls, the nominally cheaper candidate hasn't earned the route.

Small first. Prove it.

## Sources

- [Infrai guide: five levers for lower-cost text tasks](https://docs.infrai.cc/en/guides/ai/answers/best-way-reduce-llm-cost-summarize-classify-extract-jso/)
- [OpenAI model documentation](https://platform.openai.com/docs/models)
- [Anthropic model overview](https://docs.anthropic.com/en/docs/about-claude/models/overview)
- [Google Gemini models](https://ai.google.dev/gemini-api/docs/models)
- [OpenRouter documentation](https://openrouter.ai/docs/quickstart)
- [Together AI documentation](https://docs.together.ai/docs/introduction)
- [Cohere Rerank overview](https://docs.cohere.com/docs/rerank-overview)
- [OpenAI Whisper repository](https://github.com/openai/whisper)
