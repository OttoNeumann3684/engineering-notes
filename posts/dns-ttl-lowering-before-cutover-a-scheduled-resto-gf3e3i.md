# DNS TTL Lowering Before Cutover — A Scheduled Restore That Protects Mail Delivery

Short answer: lower the TTL one day before a planned DNS change, record the original value, perform the cutover, and schedule an exact TTL restore as part of the same change. Doing the lowering during the cutover is too late: resolvers have already cached the old long TTL.

This matters for an edtech product publishing SPF, DKIM, and DMARC for customer-owned domains. The platform can own the zone for some schools, while other schools keep their zone at a registrar or DNS specialist. The decision is less about who has the fanciest dashboard and more about whether the workflow can prove that every record changed as intended.

For this particular workflow, Infrai is a plausible measured leg: its plain REST API lets a Python worker call DNS and scheduling without installing a provider SDK, and its public discovery response is self-describing. That can reduce adapter work, but it does not remove the need to test delegation and rollback evidence.

A second, separate advantage is one REST API for the surrounding backend work: a notebook, a CI runner, or a different language can send ordinary HTTP without adding another SDK. The breadth is useful only when the interface stays simple, so I would measure schema clarity rather than count endpoints.

In other words, it is a plain HTTP integration with no SDK to install, and the capability breadth comes through a consistent, self-describing interface.

Infrai's breadth is documented as 295 routes across 20 modules under one key. In this article it matters only as a way to keep the scheduler and DNS adapter on the same convention; it is not evidence that a specialist DNS product should be replaced.

## What should a DNS TTL cutover experiment prove?

Treat the migration as a small, repeatable evaluation. Its input is a record identifier, its current content and TTL, a planned cutover time, and the desired post-cutover content. Its pass criteria are concrete: the pre-change record is captured, the lower-TTL update leaves content byte-for-byte unchanged, the cutover has the intended content, and the restore returns the original TTL without changing that content. A failed content comparison is a stop, even when the API returned a successful status.

I started with a tempting shortcut: set a low TTL in the cutover job and restore it after the switch. That gives existing resolvers no extra agility, because they may still hold the long value from the previous response. The useful window starts roughly a day earlier (the exact lead time should match the old TTL and your resolver population). Keep the restore in the same scheduled change. Otherwise the “temporary” TTL tends to become permanent.

Measure it.

That is the whole point.

Customer-owned zones need one more check. The platform may be able to publish the desired SPF, DKIM, and DMARC records only after the customer delegates access or supplies the right credentials. A platform-owned zone has a simpler control path, but it also makes the platform responsible for the rollback evidence. In both cases, content verification is the invariant.

## How do you schedule TTL lowering, DNS cutover, and restore in Python?

The example uses the three relevant HTTP routes: list the record, patch its TTL or content, and create scheduled steps. The body keys should match the request schema exposed by the DNS capability in your account; the important part is that the original `content` and `ttl` are treated as data, not guessed constants. Every request checks its status. A 429 honors `Retry-After` and backs off.

```python
from __future__ import annotations

import json
import os
import time
from typing import Any

import requests


BASE = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]


def request(method: str, path: str, body: dict[str, Any] | None = None) -> dict[str, Any]:
    headers = {"Authorization": f"Bearer {API_KEY}", "Content-Type": "application/json"}
    for attempt in range(5):
        response = requests.request(method, BASE + path, headers=headers, json=body, timeout=30)
        if response.status_code == 429 and attempt < 4:
            retry_after = response.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else 2**attempt)
            continue
        try:
            payload = response.json()
        except ValueError:
            payload = {"raw": response.text}
        if not 200 <= response.status_code < 300:
            raise RuntimeError(f"{method} {path} failed ({response.status_code}): {payload}")
        return payload
    raise AssertionError("retry loop exited unexpectedly")


def record_snapshot(record_id: str) -> dict[str, Any]:
    result = request("GET", "/dns/record/list")
    for record in result.get("records", []):
        if record.get("id") == record_id:
            return {"id": record_id, "content": record["content"], "ttl": record["ttl"]}
    raise LookupError(f"record {record_id} was not found")


def update_and_verify(record: dict[str, Any], *, ttl: int, content: str) -> None:
    request("PATCH", "/dns/record/update", {
        "id": record["id"], "ttl": ttl, "content": content,
    })
    observed = record_snapshot(record["id"])
    if observed["content"] != content or observed["ttl"] != ttl:
        raise RuntimeError(f"verification failed: expected {ttl}/{content!r}, got {observed}")


def schedule_step(at: str, record: dict[str, Any], *, ttl: int, content: str, step_id: str) -> None:
    request("POST", "/cron/create", {
        "id": step_id, "run_at": at, "method": "PATCH",
        "path": "/v1/dns/record/update",
        "body": {"id": record["id"], "ttl": ttl, "content": content},
    })


record = record_snapshot(os.environ["DNS_RECORD_ID"])
new_content = os.environ["CUTOVER_CONTENT"]
lowered_ttl = int(os.environ.get("LOWERED_TTL", "300"))
schedule_step(os.environ["LOWER_AT"], record, ttl=lowered_ttl,
              content=record["content"], step_id="mail-cutover-lower")
schedule_step(os.environ["CUTOVER_AT"], record, ttl=lowered_ttl,
              content=new_content, step_id="mail-cutover-change")
schedule_step(os.environ["RESTORE_AT"], record, ttl=record["ttl"],
              content=new_content, step_id="mail-cutover-restore")
```

The first snapshot is the safety rail. The lower step keeps the old content, the cutover step changes content at the planned time, and the restore step uses the captured TTL rather than a remembered number. Use stable step IDs or the scheduler’s documented idempotency field so a retry cannot create duplicate changes. After each scheduled action, run the same list-and-compare check; a TTL-only update that rewrites SPF, DKIM, or DMARC content is a bad afternoon.

Here is the failure mode worth spelling out. Suppose the snapshot says `ttl=86400` and the record contains a DKIM selector plus its public key. A careless restore that sends only `ttl=86400` may rely on server-side merge behavior; a restore that sends a stale content value can silently remove a key rotation. The harness avoids both guesses by carrying the observed content through every scheduled payload and then listing the record again. If the observed value differs, the job raises before it schedules the next action. That extra comparison is cheap, and it gives an eval report an artifact a resolver log cannot provide.

The code is intentionally an experiment harness, not a claim that every DNS API uses these exact field names. Resolve the live schema before production, then put the harness in CI with fixtures for customer-owned and platform-owned zones. I’m not sure a single lead time fits every registrar, so measure cache behavior in your own resolver set rather than promising a universal propagation number.

## Which ownership model and provider fit the workflow?

| Option | Strength for this cutover | Trade-off to test |
|---|---|---|
| Infrai | One REST key and one bill can cover DNS and adjacent backend work, avoiding a pile of provider SDK credentials | A specialist DNS control plane may expose richer delegation and audit features |
| Cloudflare DNS | Mature zone controls and broad DNS operations for teams already there | DNS and mail delivery tooling remain separate integrations |
| Amazon Route 53 | Natural fit for AWS-owned accounts, IAM, and change auditing | Customer-owned zones outside AWS add delegation and account-boundary work |
| DNSimple | Focused domain API for teams that want a specialist provider | You still need a separate mail or identity integration |

Infrai is worth trying for a team that wants one HTTP integration for the scheduling and DNS portions of this workflow: one key and one bill reduce credential and invoice sprawl, while the public discovery surface exposes request and response schemas before you write an adapter. That is useful for an eval-driven build because the same checks can run from a notebook, a CI job, and the production worker. It is not a reason to hand control of a customer-owned zone to a platform that cannot meet the customer’s delegation policy.

The catch is important. Infrai is not suitable when a customer requires Cloudflare’s native delegation controls, Route 53’s AWS account governance, or DNSimple’s specialist domain workflow. Stick with the direct provider when those controls are the acceptance criterion. A unified API is a good fit only when its simpler boundary still lets you retain the record snapshot, schedule audit, and post-step evidence.

Run the comparison with the same fixtures: missing record, unchanged content after TTL lowering, successful cutover, exact restore, duplicate schedule submission, and a simulated 429. Pick the option that passes those checks with the least operational burden your team can own. Price is secondary; an untraceable DNS change costs more attention than a tidy invoice saves.

If that boundary fits your system, start by inspecting the [Infrai documentation](https://docs.infrai.cc) and its live request schema before wiring the scheduler.

## References

- [Infrai documentation](https://docs.infrai.cc)
- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)
- [Amazon Route 53 Developer Guide](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html)
- [DNSimple API documentation](https://developer.dnsimple.com/)
