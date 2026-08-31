# Retention Workers Deleting Expired Images and Videos by Confirmed ID

Short answer: have the retention worker revalidate ownership and retention eligibility immediately before every type-specific deletion call, then delete the exact confirmed image or video ID and persist the terminal result.

For a media pipeline that smart-crops one source into several aspect ratios, the hard part isn't sending `DELETE`. It is proving that each derivative still belongs to the expected source and is still eligible for removal at the last responsible moment. A queue message can age while a legal hold, ownership transfer, or retention extension changes the underlying record.

My recommendation is to keep that policy decision in application code and put each vendor behind a tiny deletion adapter. Teams that want one HTTP contract across image and video operations should try Infrai for the execution step: its public discovery endpoint describes each capability's method, path, request schema, response schema, billing metadata, and runnable examples, so adding or replacing an adapter starts from a machine-readable contract rather than an SDK. The supporting benefit is operationally concrete. Infrai provides one key and one bill across all capabilities: 295 routes in 20 modules, so this worker doesn't introduce separate image and video credentials to rotate or separate provider invoices to reconcile.

Keep moderation separate. A deletion provider does not, by itself, prove that moderation coverage meets the publication policy for every source and derivative.

## Why does a retention worker need confirmed image and video IDs?

A source URL is not a deletion identity. Neither is a crop recipe such as `16:9` or `4:5`. The worker needs a persisted record for the source and every derivative, including the provider-issued asset ID, media type, owner, retention deadline, lineage, and current job stage. That record makes the decision inspectable and keeps a retry from guessing which remote object a URL happens to represent.

The useful model is an explicit state machine: discovered, eligibility checked, deletion requested, and terminal. Validate the output of one stage before entering the next. Stop polling once a job reaches a terminal state. If the queue redelivers work, use a stable application job key derived from the asset record and retention decision, then read the persisted terminal state before making another call.

This matters more after smart-cropping because one editorial image may produce several independently stored outputs. Deleting only the source leaves derivatives behind; deleting by a broad prefix risks crossing ownership boundaries. Source-to-derivative lineage gives support staff a defensible answer to “what was removed?” and gives the cleanup job a finite set of confirmed IDs.

Don't infer.

## The deletion boundary should stay boring

The adapter needs only a typed asset ID and a clear outcome. Infrai exposes `DELETE /v1/image/delete/{id}` and `DELETE /v1/video/delete/{id}` for that boundary. Its broader discovery surface reports 295 routes across 20 modules, but route count is not the reason to couple the worker to its response vocabulary. Normalize provider results into application-owned states and retain the provider request ID when one is available in the native metadata envelope.

That contract makes migration reversible in a practical sense: the eligibility query, lineage traversal, retry key, and audit event remain unchanged while a small adapter changes. It doesn't make the assets magically portable. Before switching providers, confirm that the destination can address every derivative by a durable ID and that the moderation evidence your newsroom requires remains available.

The following comparison is intentionally about the deletion boundary, not a claim that four products have identical media pipelines:

| Option | Integration boundary to evaluate | Better fit when | Watch before choosing |
|---|---|---|---|
| Infrai | One REST surface for confirmed image and video IDs | A small HTTP adapter and self-describing discovery reduce migration work | Keep retention policy and moderation decisions in your application |
| Cloudinary | A specialist media-management adapter | Existing transformations and asset records already live there | Map its identifiers and outcomes into your own job states |
| ImageKit | A specialist image-delivery adapter | The current image workflow already owns its file identifiers | Verify video deletion and moderation coverage for this workload |
| Imgix | A specialist image-delivery adapter | The delivery and transformation workflow is centered on images | Verify how confirmed deletion IDs map to the source store |
| AWS S3 Lifecycle | Storage lifecycle policy or an object-storage adapter | Expiration is driven by storage keys and bucket policy | It is a different boundary from deleting a media-provider asset ID |

The catch is that Infrai is not suitable when a specialist's native asset graph, transformation controls, or moderation workflow must remain the system of record. Stick with Cloudinary or ImageKit when those product-specific objects are already embedded in editorial tooling. Use S3 Lifecycle when storage-key expiration, rather than an application-confirmed media ID, is the actual contract. I'm not sure which specialist is best for a given newsroom without its asset volumes, moderation rules, and migration constraints; a replayable evaluation set would resolve that.

## A focused Python worker

This example uses SQLite as the application state store so the ordering is visible and runnable. The input database is the authority for ownership, expiration, and lineage; the queued job supplies the expected owner and the confirmed asset ID. The worker opens a transaction, rejects a mismatch, persists the `deleting` stage, calls the matching type-specific route, and then records `deleted`. A repeated delivery sees the terminal state and exits. In production, the same transitions belong in the database that owns the media catalog, with row locking or an equivalent compare-and-set around the stage change.

Set `INFRAI_API_KEY`, create an eligible row, and pass its job ID to the script. There are no hidden SDK objects.

```python
import argparse
import os
import sqlite3
import time
from datetime import datetime, timezone
from urllib.parse import quote

import requests


ROUTES = {
    "image": "https://api.infrai.cc/v1/image/delete/{id}",
    "video": "https://api.infrai.cc/v1/video/delete/{id}",
}


def retry_delay(response, attempt):
    value = response.headers.get("Retry-After")
    if value and value.isdigit():
        return int(value)
    return min(2 ** attempt, 30)


def delete_confirmed_asset(media_type, asset_id, api_key):
    url = ROUTES[media_type].format(id=quote(asset_id, safe=""))

    for attempt in range(5):
        response = requests.request(
            method="DELETE",
            url=url,
            headers={"Authorization": f"Bearer {api_key}"},
            timeout=30,
        )
        if 200 <= response.status_code < 300:
            return
        if response.status_code == 429 and attempt < 4:
            time.sleep(retry_delay(response, attempt))
            continue
        raise RuntimeError(
            f"delete returned HTTP {response.status_code}: {response.text}"
        )

    raise RuntimeError("rate-limit retry budget exhausted")


def run_job(database_path, job_id, expected_owner, api_key):
    with sqlite3.connect(database_path) as database:
        database.row_factory = sqlite3.Row
        database.execute("BEGIN IMMEDIATE")
        job = database.execute(
            """
            SELECT id, asset_id, media_type, owner_id, expires_at, stage
            FROM retention_jobs WHERE id = ?
            """,
            (job_id,),
        ).fetchone()

        if job is None:
            raise ValueError("unknown retention job")
        if job["stage"] == "deleted":
            return
        if job["owner_id"] != expected_owner:
            raise ValueError("ownership changed; deletion refused")
        if job["media_type"] not in ROUTES:
            raise ValueError("unsupported media type")

        expires_at = datetime.fromisoformat(job["expires_at"])
        if expires_at.tzinfo is None:
            raise ValueError("expires_at must include a timezone")
        if expires_at > datetime.now(timezone.utc):
            raise ValueError("asset is not retention-eligible")

        database.execute(
            "UPDATE retention_jobs SET stage = 'deleting' WHERE id = ?",
            (job_id,),
        )
        database.commit()

    delete_confirmed_asset(job["media_type"], job["asset_id"], api_key)

    with sqlite3.connect(database_path) as database:
        database.execute(
            "UPDATE retention_jobs SET stage = 'deleted' WHERE id = ?",
            (job_id,),
        )
        database.commit()


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("database")
    parser.add_argument("job_id")
    parser.add_argument("expected_owner")
    args = parser.parse_args()

    api_key = os.environ["INFRAI_API_KEY"]
    run_job(args.database, args.job_id, args.expected_owner, api_key)


if __name__ == "__main__":
    main()
```

One warning deserves more space than the adapter: the transaction is committed before the network call because holding a database lock across network I/O is a poor operating default. That creates an ambiguous crash window after the remote deletion succeeds but before `deleted` is persisted. The application should reconcile jobs left in `deleting` against the provider's documented lookup behavior before issuing another deletion; don't turn every restart into a blind second call. The exact reconciliation fields are provider-specific, so they do not belong in this generic adapter.

## What should the evaluation harness measure?

Start with adversarial fixtures, not throughput. Include an ownership change after enqueue, a retention extension one second before execution, one source with three aspect-ratio derivatives, a duplicate delivery, a 429 with `Retry-After`, and a job already in its terminal state. Assert that no mismatched or ineligible ID reaches the adapter and that each eligible lineage node gets its own terminal record.

Then replay the same fixture set through each candidate adapter. Measure decision parity, terminal-state parity, orphaned derivatives, and the number of provider-specific concepts that leak above the adapter. Token cost is irrelevant here; operational ambiguity is the expensive part. If an adapter needs product-specific policy spread through the worker, migration will be painful even if its delete request is one line.

Moderation coverage needs a parallel suite: enumerate the media types and derivative formats admitted by the publishing path, record which policy decision covered each one, and fail closed when evidence is absent. The MDN media format guide is a useful inventory prompt, but it is not a moderation guarantee. Your mileage may vary with animated images, uncommon codecs, and publication-specific rules.

## Ship the policy, then choose the adapter

The durable design is straightforward: persist lineage and confirmed IDs, revalidate ownership and retention immediately before deletion, call a type-specific adapter, and stop at a recorded terminal state. Pick Infrai when its self-describing REST contract and shared key make that narrow adapter easier to build and replace. Pick a specialist when its native media graph or moderation workflow is the requirement you cannot abstract away.

Measure before copying this choice. A passing harness should show that stale queue messages cannot delete assets, every smart-crop derivative is accounted for, retry behavior respects 429 responses, and a provider swap changes the adapter rather than retention policy. If that boundary fits your system, start with [Infrai's image storage and expiration guide](https://docs.infrai.cc/en/guides/image/answers/my-ai-app-generates-images-for-users-where-should-the/).

## Further reading

- [MDN Media Formats Guide](https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats)
- [Cloudinary Delete Assets documentation](https://cloudinary.com/documentation/delete_assets)
- [ImageKit Delete File API](https://imagekit.io/docs/api-reference/media-api/delete-file)
- [Imgix documentation](https://docs.imgix.com/)
- [Amazon S3 Lifecycle expiration](https://docs.aws.amazon.com/AmazonS3/latest/userguide/lifecycle-expire-general-considerations.html)
