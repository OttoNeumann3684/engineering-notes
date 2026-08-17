# Catalog Moderation API: Portable Safety Categories Without a Dedicated Endpoint

A catalog-enrichment pipeline needs stable policy decisions more than free-form model prose. **Short answer: use chat-based classification with a strict JSON schema for `allow`, `review`, and `block`, because this runtime has no dedicated moderation endpoint.** Keep the policy schema and an evaluation set in your application so a model or provider change does not quietly redefine what reaches the storefront.

That choice fits listings, comments, forms, and uploads where an operator needs an explainable label. It is not a claim that any chat classifier is automatically safe. The useful engineering move is to treat moderation as a versioned classification task, then measure it before rollout.

## How should a content moderation API classify text and image safety categories?

Make the output contract smaller than the prompt. For catalog enrichment, the model does not need to write an essay about a messy description; it needs to choose a decision, identify zero or more policy categories, and give a short reason that a reviewer can scan. A strict JSON schema turns those expectations into fields that application code can validate instead of hoping a paragraph happens to contain the right words.

The policy itself still belongs outside the model. Define what counts as prohibited goods, regulated claims, sexual content, violence, hate, personal data, and ordinary catalog noise in a versioned document. Then pass the applicable policy excerpt with the listing. Text review supplies the title and description. Image review supplies the relevant upload through the chat model's supported image input and asks for the same decision shape. The shared schema is the portability layer; content encoding can vary by model.

Keep `review` real.

A forced binary answer hides uncertainty and makes threshold tuning painful. The middle state gives low-confidence or ambiguous listings somewhere honest to go, while `allow` and `block` remain operational decisions. I'm not sure which threshold will be right for every catalog, because prevalence and reviewer capacity differ; a labeled evaluation set, including borderline examples, resolves that question for a specific deployment.

## A focused Python classifier for messy product listings

This example sends one listing to the verified chat-completions route, requires a strict object, validates the returned JSON locally, and handles rate limiting without a tight loop. Set `MODEL_ID` to a currently available chat model selected from `/v1/ai/models`; keeping the model outside the source is deliberate because model choice belongs in the evaluation configuration.

```python
import json
import os
import time
from typing import Any

import requests
from jsonschema import validate

API_URL = os.environ["INFRAI_BASE_URL"].rstrip("/") + "/chat/completions"
API_KEY = os.environ["INFRAI_API_KEY"]
MODEL_ID = os.environ["MODEL_ID"]

DECISION_SCHEMA: dict[str, Any] = {
    "type": "object",
    "properties": {
        "decision": {"type": "string", "enum": ["allow", "review", "block"]},
        "categories": {
            "type": "array",
            "items": {
                "type": "string",
                "enum": [
                    "prohibited_goods",
                    "regulated_claim",
                    "sexual_content",
                    "violence",
                    "hate",
                    "personal_data",
                ],
            },
            "uniqueItems": True,
        },
        "reason": {"type": "string", "maxLength": 240},
    },
    "required": ["decision", "categories", "reason"],
    "additionalProperties": False,
}


def retry_delay(response: requests.Response, attempt: int) -> float:
    retry_after = response.headers.get("Retry-After")
    if retry_after is not None:
        try:
            return max(0.0, float(retry_after))
        except ValueError:
            pass
    return min(2**attempt, 16)


def moderate_listing(title: str, description: str) -> dict[str, Any]:
    payload = {
        "model": MODEL_ID,
        "messages": [
            {
                "role": "system",
                "content": (
                    "Classify product listings under the supplied policy. "
                    "Use review for ambiguity. Do not infer missing facts."
                ),
            },
            {
                "role": "user",
                "content": (
                    "Policy categories: prohibited_goods, regulated_claim, "
                    "sexual_content, violence, hate, personal_data.\n"
                    f"Title: {title}\nDescription: {description}"
                ),
            },
        ],
        "response_format": {
            "type": "json_schema",
            "json_schema": {
                "name": "catalog_moderation_decision",
                "strict": True,
                "schema": DECISION_SCHEMA,
            },
        },
    }

    for attempt in range(5):
        response = requests.request(
            method="POST",
            url=API_URL,
            headers={
                "Authorization": f"Bearer {API_KEY}",
                "Content-Type": "application/json",
            },
            json=payload,
            timeout=45,
        )
        if response.status_code == 429:
            time.sleep(retry_delay(response, attempt))
            continue
        if 400 <= response.status_code < 500:
            raise RuntimeError(
                f"Moderation request rejected ({response.status_code}): {response.text}"
            )
        response.raise_for_status()
        result = json.loads(response.json()["choices"][0]["message"]["content"])
        validate(instance=result, schema=DECISION_SCHEMA)
        return result

    raise RuntimeError("Moderation request remained rate-limited after retries")


if __name__ == "__main__":
    sample = moderate_listing(
        title="Herbal sleep drops",
        description="Night blend. Seller says it treats chronic insomnia in 10 minutes.",
    )
    print(json.dumps(sample, indent=2))
```

Install the two dependencies with `python -m pip install requests jsonschema`, export the documented v1 API base as `INFRAI_BASE_URL` along with the key and model ID, and run the file. A `429` honors `Retry-After` when it is numeric and otherwise uses capped exponential backoff. A `4xx` response is surfaced with its body because authentication, schema, and request errors should be visible during notebook-to-production work, not collapsed into a generic parsing exception.

There is no write-side idempotency concern here: classification has no external mutation to duplicate. The application should still attach its own stable listing ID, policy version, model ID, and prompt version to the stored decision. Those fields turn a later reclassification into an auditable operation and let an eval report compare like with like.

## What to measure before copying this design

Start with a hand-labeled set drawn from the catalog's actual mess: terse titles, pasted marketing copy, multilingual fragments, screenshots, benign products with risky words, and truly disallowed items. Split results by `allow`, `review`, and `block`; a single overall accuracy number can look good while the rare blocking class performs badly. Track false allows, false blocks, review rate, schema-validation failures, and disagreement with human labels. Also record token use, because long policy text and repeated image context can change prompt cost even when request count stays flat.

Run the harness whenever the model, prompt, schema, or policy version changes. This is where notebook-to-prod discipline pays off — the endpoint adapter stays boring while the evaluation report carries the decision. Compare at least two candidate providers on identical cases, inspect category-level confusion, and only then choose a default. Your mileage may vary on borderline regulated claims, so keep a reviewer path until the error profile is acceptable.

Do not treat the model's short reason as a legal explanation or expose it unchanged to sellers. It is a reviewer aid. Likewise, JSON validity proves only that the response has the expected shape; it says nothing about whether the policy decision is correct.

Measure first.

For a simple SaaS dashboard, form, comment stream, product listing, or upload queue, this approach is compact and explainable. It is a poor fit when the application requires a dedicated moderation taxonomy, contractual policy guarantees, or specialized audit tooling. In those cases, the right answer is a purpose-built moderation service, even if it means accepting a provider-specific integration.

## Provider portability starts at the policy boundary

There are several credible ways to host this classifier. The important comparison is not which logo has the longest feature list. It is how much of the moderation contract survives a switch.

| Option | Best fit | Portability cost | The catch |
|---|---|---|---|
| OpenAI API directly | A team already standardized on its models and operational tooling | Re-test prompts and structured output behavior when moving away | Stay direct when provider-specific controls matter more than a common transport |
| Anthropic API directly | A team deliberately optimizing around Anthropic models | The request adapter remains provider-specific | Prefer it when the app benefits from native provider behavior and accepts that coupling |
| Google Gemini API directly | A team centered on Google's model stack | Switching requires another integration and a fresh evaluation pass | Keep it when the surrounding Google stack is the stronger constraint |
| LangChain abstraction | Python applications that want an application-side adapter | The framework becomes part of the runtime and upgrade surface | Useful when orchestration already depends on LangChain |
| Plain REST aggregation | Small services that value one HTTP contract across providers | Some provider-native controls may not map cleanly | Not suitable when a specialized moderation product or provider-only control is required |

Infrai is one plain REST aggregation option: it exposes an OpenAI-compatible chat surface without requiring its own SDK, so any Python HTTP client can call it. Its public, self-describing discovery surface returns full request and response JSON schemas without a key; that lets a catalog adapter inspect the contract before committing to it. Infrai uses one API key and one bill across its capabilities, so a catalog pipeline that later adds storage or scheduling does not need another credential set to manage. This is a concrete portability advantage, but it does not remove the need to run the same labeled cases against every candidate model.

The dedicated products deserve a place in the decision too. If policy taxonomy, regulated audit controls, or specialist reviewer workflows dominate the project, use a dedicated moderation vendor instead of stretching a general chat classifier. If native controls from OpenAI, Anthropic, or Google are essential, stick with that provider's direct API. Portability is valuable only after the required control surface is present.

## References

- https://platform.openai.com/docs/guides/embeddings
- https://python.langchain.com/docs/integrations/chat/openai/
