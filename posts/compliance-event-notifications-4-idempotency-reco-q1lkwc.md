# Compliance Event Notifications: 4 Idempotency Records for Exactly-Once Email Retries

TL;DR: Own the approved template and the delivery ledger in the application, then treat each compliance notice as one immutable intent per recipient and channel. A retry may reclaim unfinished work, but it must reuse the same intent key, rendered content, and message identity. This does not create a magical exactly-once network operation. It creates an auditable state machine that prevents ordinary duplicate sends and makes the unavoidable crash window visible instead of guessing about it.

For a fintech notice, the useful unit is not “an email job.” It is a specific disclosure obligation: account, notice type, effective event, recipient, channel, and approved template version. Persist that unit before contacting an email or SMS transport. The flow is straightforward: accept the business event, derive a stable notice key, freeze the approved content, reserve one channel delivery, send with a stable downstream idempotency identifier where supported, and record the receipt. Four durable records carry the proof: the notice intent, the template version, the channel delivery, and the provider receipt.

## How can retries avoid duplicate event notifications and send email exactly once?

The dangerous interval sits between an external side effect and a local commit. A worker can submit an email, lose its database connection before saving the receipt, and then be retried. Its database still says “sending,” while the recipient may already have the message. No local transaction can atomically commit work inside an unrelated delivery service.

Tiny window. Real risk.

Queue redelivery, overlapping schedulers, impatient API clients, and a worker killed after a timeout all reach the same branch. The fix starts before the worker: assign the notice a stable business identity and reject a second intent with that identity. A random key generated inside every attempt is useless because each retry looks new.

For example, derive the key from immutable facts such as `account_id`, `notice_kind`, `effective_event_id`, `recipient_id`, and `channel`. Do not include an attempt number or current timestamp. Store the full tuple under a unique constraint as well as its digest; that makes hash inputs reviewable during an audit. If the recipient address changes, policy must decide whether that creates a new intent. Hiding that decision inside a hash function produces tidy code and poor evidence.

Email's `Message-ID` identifies a message according to the Internet Message Format, but it is not a universal receiver-side deduplication contract. SMS likewise has no channel-wide exactly-once primitive. A transport-specific idempotency feature can narrow the crash window, yet the application ledger remains the system of record because channel acknowledgements and delivery receipts describe different stages.

## Build the ledger before the sender

The example below uses Python's standard library and SQLite so the state transition can run in a notebook or a test without infrastructure. The same constraints belong in the production database. `reserve_delivery` is the critical operation: concurrent workers race on a unique `(notice_id, channel)` constraint, and only one obtains the durable row.

```python
import hashlib
import json
import sqlite3
from dataclasses import dataclass
from typing import Protocol


SCHEMA = """
PRAGMA foreign_keys = ON;

CREATE TABLE IF NOT EXISTS notice_intent (
    id INTEGER PRIMARY KEY,
    intent_key TEXT NOT NULL UNIQUE,
    account_id TEXT NOT NULL,
    notice_kind TEXT NOT NULL,
    effective_event_id TEXT NOT NULL,
    recipient_id TEXT NOT NULL,
    template_version TEXT NOT NULL,
    rendered_body TEXT NOT NULL,
    created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS channel_delivery (
    id INTEGER PRIMARY KEY,
    notice_id INTEGER NOT NULL REFERENCES notice_intent(id),
    channel TEXT NOT NULL CHECK (channel IN ('email', 'sms')),
    status TEXT NOT NULL CHECK (status IN ('reserved', 'accepted', 'unknown')),
    attempt_count INTEGER NOT NULL DEFAULT 0,
    transport_key TEXT NOT NULL UNIQUE,
    accepted_at TEXT,
    UNIQUE (notice_id, channel)
);

CREATE TABLE IF NOT EXISTS delivery_receipt (
    id INTEGER PRIMARY KEY,
    delivery_id INTEGER NOT NULL REFERENCES channel_delivery(id),
    receipt_kind TEXT NOT NULL,
    transport_reference TEXT NOT NULL,
    payload_json TEXT NOT NULL,
    observed_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP,
    UNIQUE (delivery_id, receipt_kind, transport_reference)
);
"""


class Transport(Protocol):
    def send(self, *, channel: str, body: str, idempotency_key: str) -> str:
        """Return the transport's acceptance reference."""


@dataclass(frozen=True)
class Notice:
    account_id: str
    notice_kind: str
    effective_event_id: str
    recipient_id: str
    template_version: str
    rendered_body: str


def stable_key(parts: list[str]) -> str:
    encoded = json.dumps(parts, separators=(",", ":"), ensure_ascii=True)
    return hashlib.sha256(encoded.encode("utf-8")).hexdigest()


def create_intent(db: sqlite3.Connection, notice: Notice) -> int:
    key = stable_key([
        notice.account_id,
        notice.notice_kind,
        notice.effective_event_id,
        notice.recipient_id,
    ])
    db.execute(
        """INSERT OR IGNORE INTO notice_intent
           (intent_key, account_id, notice_kind, effective_event_id,
            recipient_id, template_version, rendered_body)
           VALUES (?, ?, ?, ?, ?, ?, ?)""",
        (key, notice.account_id, notice.notice_kind,
         notice.effective_event_id, notice.recipient_id,
         notice.template_version, notice.rendered_body),
    )
    row = db.execute(
        "SELECT id FROM notice_intent WHERE intent_key = ?", (key,)
    ).fetchone()
    return int(row[0])


def reserve_delivery(db: sqlite3.Connection, notice_id: int, channel: str) -> tuple[int, str]:
    transport_key = stable_key([str(notice_id), channel])
    db.execute(
        """INSERT OR IGNORE INTO channel_delivery
           (notice_id, channel, status, transport_key)
           VALUES (?, ?, 'reserved', ?)""",
        (notice_id, channel, transport_key),
    )
    row = db.execute(
        """SELECT id, transport_key FROM channel_delivery
           WHERE notice_id = ? AND channel = ?""",
        (notice_id, channel),
    ).fetchone()
    return int(row[0]), str(row[1])


def deliver(db: sqlite3.Connection, delivery_id: int, transport: Transport) -> None:
    row = db.execute(
        """SELECT d.channel, d.status, d.transport_key, n.rendered_body
           FROM channel_delivery AS d
           JOIN notice_intent AS n ON n.id = d.notice_id
           WHERE d.id = ?""",
        (delivery_id,),
    ).fetchone()
    if row is None or row[1] == "accepted":
        return

    channel, _, transport_key, body = row
    db.execute(
        "UPDATE channel_delivery SET attempt_count = attempt_count + 1 WHERE id = ?",
        (delivery_id,),
    )
    db.commit()

    try:
        reference = transport.send(
            channel=channel, body=body, idempotency_key=transport_key
        )
    except TimeoutError:
        db.execute(
            "UPDATE channel_delivery SET status = 'unknown' WHERE id = ?",
            (delivery_id,),
        )
        db.commit()
        raise

    db.execute(
        """UPDATE channel_delivery
           SET status = 'accepted', accepted_at = CURRENT_TIMESTAMP
           WHERE id = ?""",
        (delivery_id,),
    )
    db.execute(
        """INSERT OR IGNORE INTO delivery_receipt
           (delivery_id, receipt_kind, transport_reference, payload_json)
           VALUES (?, 'accepted', ?, ?)""",
        (delivery_id, reference, json.dumps({"reference": reference})),
    )
    db.commit()


if __name__ == "__main__":
    connection = sqlite3.connect(":memory:")
    connection.executescript(SCHEMA)
    sample = Notice(
        account_id="acct_1042",
        notice_kind="fee_schedule_change",
        effective_event_id="evt_88017",
        recipient_id="customer_391",
        template_version="fee-change-v7",
        rendered_body="Your account fee schedule changes on the stated effective date.",
    )
    with connection:
        notice_id = create_intent(connection, sample)
        delivery_id, _ = reserve_delivery(connection, notice_id, "email")
        assert reserve_delivery(connection, notice_id, "email")[0] == delivery_id
```

The sample intentionally does not auto-resend an `unknown` delivery. A timeout does not prove failure; it means the outcome needs reconciliation. Query the transport by the stable key or acceptance reference if that capability exists. Otherwise, route the record to an explicit policy: wait for a late receipt, review it, or authorize a resend while retaining both attempts. Calling an uncertain outcome “failed” is the classic duplicate-notice bug.

Do not retry yet.

The code also commits the incremented attempt count before the network call. That is an audit choice, not a delivery guarantee. The ledger can now show that an attempt began even if the process disappears. In a larger system, an outbox row written in the same transaction as the business event closes the earlier gap between creating an obligation and enqueueing its work. This design has limitations: it adds durable state, reconciliation work, retention decisions, and an operator path. It is a poor trade-off for disposable, low-consequence notifications where an occasional duplicate is explicitly acceptable; a short-lived deduplication cache may be enough there. It is also insufficient by itself when the transport cannot accept a stable idempotency key or expose a lookup operation, because the post-send crash remains ambiguous. The documented policy must then specify manual review, a delayed resend, or reconciliation of accepted requests at the transport boundary. None can prove a negative from a timeout. The policy depends on the harm of a missed statutory notice versus the harm of sending it twice, and it belongs in a documented decision rather than a worker's exception handler.

## Template ownership is part of idempotency

A retry must not render “the latest template.” Compliance wording can change between attempts, and dynamic account data can drift too. If the first email carried version 7 while a retry renders version 8, those are materially different messages even when their database key matches. Store an immutable template version plus the rendered subject and body, or store the complete immutable inputs and a deterministic renderer whose artifact is retained. For audit work, the frozen output is easier to inspect.

This matters even more when an AI system helps draft or classify content. Generation belongs before approval, not inside the retry loop. Pin the approved artifact, record its version, and run an evaluation fixture that checks required clauses, forbidden placeholders, locale, and length boundaries. A retry should spend zero model tokens because it should perform zero new generation.

Application ownership also makes channel adaptation explicit. An email and an SMS may represent the same legal obligation, but they are separate delivery rows with separate content artifacts and receipts. Decide whether SMS is a parallel requirement or a fallback. If it is a fallback, transition to it through policy after the email outcome is known; do not let two independent workers infer fallback from silence.

**The invariant is precise:** one approved notice intent may have at most one active delivery per required channel, while every network attempt and receipt remains append-only evidence. “One row forever” is too restrictive when policy authorizes a corrected notice. “One job at a time” is too weak because jobs are ephemeral.

## Test the crash windows, not only the happy path

An eval harness for delivery code should inject failures at boundaries. Run two workers against the same event and verify that the unique constraint produces one delivery row. Raise a timeout after the fake transport records acceptance, then confirm the state becomes `unknown` and the next worker does not blindly send. Kill processing before the transport call and verify that a lease or recovery process can reclaim the reservation. Deliver the same receipt twice and expect one stored receipt.

Use fixed fixtures. One useful matrix varies event duplication, worker concurrency, transport outcome, and template version. The assertions should cover database state, transport call count, rendered-content digest, and the evidence returned to an auditor. A test that checks only the final `accepted` status can pass while two messages leave the system.

There is a subtle correction worth making in many retry designs: exponential backoff controls pressure, not uniqueness. It may reduce simultaneous attempts, but it can't distinguish “the request failed” from “the response was lost.” Jitter solves synchronized load. The ledger solves identity. Reconciliation handles uncertainty.

Observability should use the intent key and delivery ID as correlation fields, while keeping account addresses and message bodies out of routine logs. Count transitions into `unknown`, reservation conflicts, receipt latency, and manually authorized resends. Alert on a growing age of unresolved records rather than raw retry volume alone. The former points to evidence that the workflow cannot currently settle.

## Operate the proof as carefully as the message

Before release, walk one notice from originating event to rendered artifact, delivery reservation, acceptance record, and later receipt. Confirm that the template repository has named owners and an approval history; database access cannot compensate for an untraceable wording change. Verify retention and access controls against the organization's legal and privacy requirements, because message content and receipts can contain sensitive data. There is no universal retention period to copy from an example.

Then exercise recovery with the network disconnected after acceptance. An operator should see `unknown`, the stable transport key, attempt timestamps, and the permitted next actions. The operator should not need to edit a row or manufacture a new key. If reconciliation is impossible, a resend decision needs an actor, reason, and link to the original delivery. Keep that decision append-only.

Finally, review cost at the correct boundary. Duplicate suppression avoids unnecessary channel calls, but cost is not a sound reason to weaken evidence or combine distinct recipients under one key. For AI-assisted workflows, keep prompt evaluation and generation upstream of approved templates. Delivery workers stay deterministic, cheap to replay, and boring to debug.

The practical promise is narrower than “exactly once,” and stronger because it can be demonstrated: one durable business intent, one frozen template artifact, one channel reservation, and a preserved chain of attempts and receipts. Retries reuse identity. Unknown outcomes wait for evidence.

## References

- RFC 5322, Internet Message Format: https://www.rfc-editor.org/rfc/rfc5322
- RFC 9110, HTTP Semantics: https://www.rfc-editor.org/rfc/rfc9110
- Transactional Outbox pattern: https://microservices.io/patterns/data/transactional-outbox.html
- SQLite uniqueness constraints: https://www.sqlite.org/lang_createtable.html#uniqueconst
- NIST Secure Software Development Framework, SP 800-218: https://csrc.nist.gov/pubs/sp/800/218/final

## Sources

- https://www.rfc-editor.org/rfc/rfc5322
- https://www.rfc-editor.org/rfc/rfc9110
- https://microservices.io/patterns/data/transactional-outbox.html
- https://www.sqlite.org/lang_createtable.html#uniqueconst
- https://csrc.nist.gov/pubs/sp/800/218/final
