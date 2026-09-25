# How to Design Clinical Image Previews with Minimal Sensitive-Data Handling

Short answer: render a disposable, low-resolution derivative for the portal, keep the diagnostic original immutable, and make every transformation an explicit, logged step. For a fintech team borrowing patterns from clinical imaging, this is also the right way to preview identity documents before OCR: inspect less data, for less time, with a reversible path to the source.

I start in a notebook with representative images, then move the exact same checks into production. It's easy to pass a demo while quietly stripping orientation metadata, changing color profiles, or retaining a sensitive intermediate in a cache. The goal is not a prettier thumbnail. It is a bounded transformation whose failure mode is obvious.

Start small.

## How should clinical image previews minimize transformations around sensitive originals?

Treat the original as an evidence object, not an input buffer. Store it once with a content hash and immutable retention policy. Generate a preview in memory or in a short-lived isolated workspace, and send only that derivative to the browser. Never resize the original in place, and do not let a convenience image library write over the source path.

A practical flow has four stages: validate the upload, read pixels without trusting embedded instructions, transform to a conservative display format, and attach provenance. In a medical image portal that means checking the declared MIME type against magic bytes, dimensions, and a decompression limit before a worker touches pixels; applying orientation exactly once; removing metadata that can identify a patient or account while preserving enough visual information for a human to decide whether OCR is worth running; and recording the source hash, transform policy version, and expiry time without placing original pixels in an application log. The same sequence works for a fintech receipt: the preview is a review aid, while the immutable upload remains the evidence object that downstream extraction can revisit under authorization.

Here is a small Python reference implementation for a preview worker. It uses Pillow because the interface is easy to reproduce in a notebook; the policy decisions are the important part, not the library brand.

```python
from __future__ import annotations

import hashlib
import io
from dataclasses import dataclass

from PIL import Image, ImageOps


@dataclass(frozen=True)
class Preview:
    source_sha256: str
    jpeg_bytes: bytes
    width: int
    height: int
    policy: str


def make_preview(raw: bytes, max_edge: int = 1600) -> Preview:
    source_sha256 = hashlib.sha256(raw).hexdigest()
    with Image.open(io.BytesIO(raw)) as opened:
        opened.verify()

    with Image.open(io.BytesIO(raw)) as opened:
        image = ImageOps.exif_transpose(opened).convert("RGB")
        image.thumbnail((max_edge, max_edge), Image.Resampling.LANCZOS)
        output = io.BytesIO()
        image.save(output, format="JPEG", quality=85, optimize=True, exif=b"")
        return Preview(
            source_sha256=source_sha256,
            jpeg_bytes=output.getvalue(),
            width=image.width,
            height=image.height,
            policy="preview-v1",
        )
```

The `verify()` pass catches truncated files before the worker allocates a large pixel buffer. In production I also set Pillow's decompression-bomb limit and enforce a byte quota at the upload boundary. The returned hash lets an OCR job refer to the source without copying it into a queue payload. A short-lived object key can point to the preview, while the original stays behind an authorization check.

One sentence of code review: if a function accepts a path to the source and returns a path with the same name, reject it. That's it.

## What can go wrong when a preview is treated like a harmless thumbnail?

The common failure is accidental mutation. A rotation fix saves back to the upload, and the next audit discovers that the hash no longer matches the file that was consented to. Another is metadata leakage: EXIF GPS is obvious, but software tags, embedded thumbnails, and PDF-to-image comments can reveal more than expected. A third is visual misinterpretation. Aggressive JPEG compression can erase a decimal point on a scanned amount; a sharpen filter can make a faint mark look authoritative. For OCR, that is a correctness problem and a moderation problem at the same time. It gets worse when a queue retries a job: if the worker stores an intermediate under a predictable name, a later request can expose yesterday's derivative before authorization runs, so I bind object keys to the source hash and an expiry record instead of to a user-visible filename.

I once saw an eval notebook score a pipeline higher after a resize change because the sample set contained clean, front-facing photos. A held-out set with glare and rotated receipts reversed the result. The lesson was uncomfortable: preview quality and extraction quality are different metrics. I now keep a tiny fixture matrix with orientation, glare, handwriting, and low contrast, and I compare character error rate plus a human review flag. Your mileage may vary on the exact threshold; the threshold should be decided with the compliance owner, not guessed from a screenshot.

## A notebook-to-prod evaluation loop

Start with a corpus that has been explicitly cleared for testing. Record expected text and a binary decision such as `needs_manual_review`. Run the preview transform and OCR separately so a regression in one cannot hide behind the other. Prompt-cost awareness matters when a vision model performs a second pass: send the smallest derivative that still preserves the characters under review, and avoid resending the same image in every agent turn.

```python
from dataclasses import dataclass


@dataclass
class Case:
    image: bytes
    expected: str
    review: bool


def score_case(case: Case, ocr) -> dict:
    preview = make_preview(case.image)
    text = ocr(preview.jpeg_bytes)
    normalized = " ".join(text.split()).lower()
    target = " ".join(case.expected.split()).lower()
    return {
        "exact": normalized == target,
        "review_expected": case.review,
        "preview_bytes": len(preview.jpeg_bytes),
    }
```

Keep the output small and boring: counts, error categories, and links to a case identifier. Do not log the base64 image, OCR prompt, or model response verbatim when those fields can contain personal data. Set alerts on preview retention and failed deletion, not only on latency. A 200 response from an image service says almost nothing about whether the right artifact was displayed.

## Choosing the boundary that fits your portal

A browser-side transform reduces server handling but gives you less control over device memory, color management, and evidence capture. A server-side worker centralizes policy and audit, at the cost of one more trusted component. A hybrid approach can show a local, immediate placeholder and replace it with a policy-approved derivative after upload. The choice depends on threat modeling and clinical workflow, not on which option has the shortest code sample.

The catch is that this design is not suitable when clinicians must inspect pixel-level detail, multi-frame studies, or lossless measurements in the preview itself. Keep a standards-aware viewer and serve the original under a separate permission path in that case. It is also a poor fit for offline capture devices that cannot guarantee deletion; use encrypted local storage and an explicit sync lifecycle there.

My operational checklist is intentionally prose: pin the transform policy, hash before decoding, cap dimensions, transpose orientation once, strip metadata, expire derivatives, test ugly fixtures, and have a reviewer sign off on the escalation rule. If any one of those is implicit, the preview is doing more than your architecture can explain.

## References

- https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats
- https://www.dicomstandard.org/current
- https://www.w3.org/TR/exif/
- https://www.nist.gov/privacy-framework
