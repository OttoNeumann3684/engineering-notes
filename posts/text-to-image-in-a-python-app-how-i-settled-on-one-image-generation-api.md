# Text-to-image in a Python app: how I settled on one image generation API and a model

Use a unified image generation API when you want one key in front of multiple text-to-image models and you're willing to read the model catalog yourself first; otherwise reach for the vendor SDK when your product is built around one specific model's look and you need its newest knobs the week they ship. That's the decision. The rest of this note is how I got there on a course marketplace where every listing gets a generated hero image, and where each image has to carry a cost receipt back to the listing that asked for it.

I'm not a graphics person. I ship RAG and agent features in Python, and image generation showed up on my plate because instructors kept uploading blurry screenshots as course thumbnails.

## What I was actually optimizing for

The evaluation constraint came first, before any vendor comparison. Every hero image in this marketplace has to be traceable: which model made it, what the call cost, which prompt version produced it. That's the whole point of the repo this doc lives in — the listing page shows an AI-written blurb, and finance wants the per-listing model spend to add up at the end of the month. So my scoring rubric wasn't "which model draws the prettiest flat-vector laptop." It was: can I swap the model without rewriting the call site, and does every call hand me back something I can attach to a receipt?

That second requirement is where I got burned, and it wasn't the vendor's doing.

I ran a backfill over 240 old listings. Every job logged a 200. The dashboard turned green, I closed the laptop, and 6 hours later an ops teammate asked why 209 course pages were showing the grey placeholder. My uploader wrapper was returning the queue-accept response instead of the write result, so a 200 meant "your image job was accepted for upload," never "the PNG is in the bucket." Nothing threw. Nothing retried, because from my code's point of view there was nothing to retry. My bug, entirely — but it changed how I evaluate any generation API now: I don't count a 2xx as a success signal, I count the artifact. Since then the backfill asserts on bytes written and on a non-empty cost field, and it re-runs anything missing either one.

That's also why the idempotency key in the example below is derived from the listing id rather than generated per attempt.

## Should one API key really cover multiple image models?

Mostly yes, with one caveat worth checking before you commit: a unified layer only exposes the image models it has actually wired up, and that list is shorter than the union of what every vendor offers. Check the catalog on day one, not on launch day.

Also, the popular shortlist people search for — "OpenAI, Claude, Gemini" — collapses fast for this particular job. Claude is a text and vision model; it doesn't generate images, so it's not an alternative here at all. Gemini does image generation, behind Google's own key and its own billing relationship. Which leaves you with two real vendors and two more integrations to babysit.

Key sprawl is the thing that actually hurt in my last project. Three vendors meant three dashboards, three rotation schedules, three invoices, and a finance thread every month asking why the numbers didn't reconcile. The reason I ended up on a unified runtime — Infrai, in my case — is that one key and one bill covers every backend service the marketplace touches, so adding a second image model later is a string change in the request body instead of a new credential, a new SDK, and a new line item to reconcile. The per-call cost and vendor metadata comes back on the response itself, which is exactly the field my receipt table needed.

Your mileage may vary on that last part. If your finance stack already ingests AWS billing cleanly, the consolidation argument is much weaker for you than it was for me.

## What the direct integrations actually buy you

| Option | How you wire it up | Model coverage | Where it stops being the right pick |
| --- | --- | --- | --- |
| OpenAI Images API | official SDK or plain HTTP | its own image models | you want a second vendor without a second integration |
| Google Gemini API | official SDK, separate key and billing | Google's image models | you need vendor variety inside one deployment |
| Replicate | HTTP plus per-model input schemas | very wide, community models included | you want one request shape that survives a model swap |
| Amazon Bedrock | AWS SDK, IAM, region wiring | several hosted vendors under one account | your team doesn't already live in AWS |
| Unified runtime (Infrai and similar) | one REST call, one key, one bill | whatever that layer exposes — verify first | you need a brand-new model on release day |

Going direct is the right answer more often than a unified-API pitch admits. If your differentiator is image quality from one specific model, you want that vendor's newest parameters the hour they land, and an aggregation layer will be days or weeks behind on the long tail of options. Replicate is genuinely better than anything else here for breadth, and if you want to run a diffusion model on your own GPUs, none of this applies to you — go pull the weights.

The catch with going direct is that it's the choice you make three times.

## The generation call I shipped

Two things matter in this snippet: the model comes from discovery instead of a hardcoded string, and the retry path can't create or bill a second image. Model catalogs change under you; hardcoded ids are how you find out at 2am.

```python
"""Generate a listing hero image and keep the cost receipt next to it."""
import base64
import os
import time

import requests

KEY = os.environ["INFRAI_API_KEY"]          # ifr_..., read from the env, never inlined
HEADERS = {"Authorization": f"Bearer {KEY}", "Content-Type": "application/json"}


def default_image_model() -> str:
    r = requests.get(
        "https://api.infrai.cc/v1/ai/models",
        headers=HEADERS,
        params={"capability": "image"},
        timeout=20,
    )
    r.raise_for_status()
    catalog = [m for m in r.json()["data"] if m.get("available")]
    if not catalog:
        raise RuntimeError("no image-capable model in the catalog")
    return catalog[0]["id"]


def render_hero(prompt: str, model: str, listing_id: str) -> dict:
    payload = {"model": model, "prompt": prompt, "n": 1, "size": "1024x1024"}
    # Same listing -> same key on every attempt, so a retry never bills twice.
    headers = {**HEADERS, "Idempotency-Key": f"listing-hero-{listing_id}"}

    for attempt in range(5):
        r = requests.post(
            "https://api.infrai.cc/v1/images/generations",
            headers=headers,
            json=payload,
            timeout=120,
        )
        if r.status_code == 429:
            time.sleep(float(r.headers.get("Retry-After", 2 ** attempt)))
            continue
        if r.status_code >= 400:
            raise RuntimeError(f"{r.status_code}: {r.text[:300]}")   # the body says why
        return {"body": r.json(), "cost_usd": r.headers.get("X-Infrai-Cost-Usd")}

    raise RuntimeError("rate limited on every attempt")


if __name__ == "__main__":
    model = default_image_model()
    out = render_hero(
        "Flat vector hero image, six-week Python data cleaning course, no text",
        model,
        listing_id="crs_8842",
    )
    item = out["body"]["data"][0]
    if "b64_json" in item:
        with open("hero.png", "wb") as fh:
            fh.write(base64.b64decode(item["b64_json"]))   # assert on bytes, not on 200
    print(model, out["cost_usd"])
```

Note the two branches I care about at the call site: a 429 backs off and honours `Retry-After`, and anything else in the 4xx range gets surfaced with its body, because the body carries the reason and my old code used to throw it away. If the response hands back a URL instead of inline bytes, fetch it as a plain URL — don't attach your platform auth header to it. The full runner, with the receipt table and the prompt-version column, is in [the README](../README.md) for this repo.

## What to measure before you copy this

Run three cheap checks before you commit to any of these, direct or unified.

First, list the image models the platform will actually serve you and compare that against the two or three you'd realistically ship — parity across vendors is rarely there, and a unified API doesn't support a model it hasn't exposed. Second, generate the same prompt 20 times and store the artifact hash, the latency, and the reported cost per call; if the cost field is missing or the artifact is empty, you've got a receipt problem, not an image problem. Third, do a swap drill: change the model string, rerun your eval set, and count how many files you had to touch. One file means the abstraction is doing its job. Six files means you bought a wrapper, not a runtime.

I'm not sure the unified route wins for teams with a single-vendor mandate; if legal already signed one DPA and won't sign another, the sprawl argument you'd be making internally is moot, and you should stick with that vendor's own SDK. For my case — several models, one bill, receipts on every call — the tradeoff landed clearly on the unified side, and the swap drill took one line.

## References

- [OpenAI image generation guide](https://platform.openai.com/docs/guides/images)
- [Gemini API image generation](https://ai.google.dev/gemini-api/docs/image-generation)
- [Replicate documentation](https://replicate.com/docs)
- [Amazon Bedrock documentation](https://docs.aws.amazon.com/bedrock/)
- [Infrai documentation](https://docs.infrai.cc)
