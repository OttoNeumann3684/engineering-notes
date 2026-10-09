# FastAPI SMS 2FA Login: 6 Evidence Events for Polling Failed Delivery

Short answer: treat an SMS send receipt as the start of an authentication attempt, not proof that a seller received a code. For a logistics marketplace, the backend should bind each OTP attempt to the seller and new order, persist six evidence records, poll delivery through a provider-neutral adapter, and permit verification independently of a late delivery callback. This costs a little more storage and code than a `send()` call. It produces a defensible account of what the system requested, observed, and allowed without storing the OTP or full phone number.

The six records are `otp_issued`, `send_accepted`, `delivery_observed`, `verification_failed`, `verification_succeeded`, and `attempt_closed`. They are events, not six mutable columns. A current-state projection makes the API quick; the event sequence is the compliance evidence.

## How should a backend poll failed SMS 2FA login delivery?

An upstream message identifier proves that a gateway accepted a request. It does not prove handset delivery, and it certainly does not prove that the intended human read the message. Keep those meanings separate. Map provider-specific responses into a small internal vocabulary such as `accepted`, `delivered`, `failed`, and `unknown`, while retaining the original status in a protected evidence record for later investigation.

The distinction matters in the new-order flow. A seller may enter a valid code while the carrier status still says `accepted`; rejecting that code would turn messaging telemetry into an authentication dependency. Conversely, a `delivered` status must never authenticate anyone. **Delivery state informs recovery; possession of the unexpired code controls verification.**

That separation is the hinge.

Use an attempt identifier as the correlation key. The record can contain a seller ID, order ID, purpose, timestamps, a provider message ID, normalized delivery state, and a redacted destination. Keep message text and raw phone numbers out of routine logs. Access to the order should still be authorized after OTP verification; knowing a code for one attempt must not grant access to a different seller's order.

## Implement the evidence loop first

This compact FastAPI example keeps data in memory so the control flow is visible. Its `SmsPort` is the only provider-dependent boundary. The example deliberately makes the fake adapter return `accepted` once and `delivered` on the next poll, which lets a local test exercise the transition without a network account.

```python
from __future__ import annotations

import hashlib
import hmac
import secrets
import time
import uuid
from dataclasses import dataclass, field
from typing import Literal, Protocol

from fastapi import FastAPI, HTTPException
from pydantic import BaseModel


Delivery = Literal["accepted", "delivered", "failed", "unknown"]
SECRET = secrets.token_bytes(32)
OTP_TTL_SECONDS = 300
MAX_VERIFY_TRIES = 5


class SmsPort(Protocol):
    def send(self, destination: str, body: str) -> str: ...
    def status(self, message_id: str) -> tuple[Delivery, str]: ...


class LocalSms:
    def __init__(self) -> None:
        self.polls: dict[str, int] = {}

    def send(self, destination: str, body: str) -> str:
        message_id = f"msg_{uuid.uuid4().hex}"
        self.polls[message_id] = 0
        return message_id

    def status(self, message_id: str) -> tuple[Delivery, str]:
        if message_id not in self.polls:
            return "unknown", "not_found"
        self.polls[message_id] += 1
        if self.polls[message_id] == 1:
            return "accepted", "queued"
        return "delivered", "delivered"


@dataclass
class Attempt:
    id: str
    seller_id: str
    order_id: str
    destination_hint: str
    code_digest: str
    expires_at: float
    message_id: str | None = None
    delivery: Delivery = "unknown"
    verify_tries: int = 0
    verified: bool = False
    closed: bool = False
    events: list[dict[str, object]] = field(default_factory=list)


class StartRequest(BaseModel):
    seller_id: str
    order_id: str
    phone_e164: str


class VerifyRequest(BaseModel):
    code: str


app = FastAPI()
sms: SmsPort = LocalSms()
attempts: dict[str, Attempt] = {}


def digest(attempt_id: str, code: str) -> str:
    payload = f"{attempt_id}:{code}".encode()
    return hmac.new(SECRET, payload, hashlib.sha256).hexdigest()


def record(attempt: Attempt, kind: str, **details: object) -> None:
    attempt.events.append(
        {"kind": kind, "at": int(time.time()), "details": details}
    )


def hint(phone: str) -> str:
    return f"***{phone[-4:]}" if len(phone) >= 4 else "***"


@app.post("/otp-attempts", status_code=202)
def start_otp(request: StartRequest) -> dict[str, object]:
    attempt_id = uuid.uuid4().hex
    code = f"{secrets.randbelow(1_000_000):06d}"
    attempt = Attempt(
        id=attempt_id,
        seller_id=request.seller_id,
        order_id=request.order_id,
        destination_hint=hint(request.phone_e164),
        code_digest=digest(attempt_id, code),
        expires_at=time.time() + OTP_TTL_SECONDS,
    )
    attempts[attempt_id] = attempt
    record(attempt, "otp_issued", ttl_seconds=OTP_TTL_SECONDS)

    # Never log this body: it contains both order context and the live secret.
    body = f"Order {request.order_id}: your sign-in code is {code}"
    attempt.message_id = sms.send(request.phone_e164, body)
    attempt.delivery = "accepted"
    record(attempt, "send_accepted", message_id=attempt.message_id)
    return {"attempt_id": attempt_id, "expires_in": OTP_TTL_SECONDS}


@app.get("/otp-attempts/{attempt_id}")
def get_status(attempt_id: str) -> dict[str, object]:
    attempt = attempts.get(attempt_id)
    if attempt is None:
        raise HTTPException(404, "Attempt not found")
    if attempt.message_id and not attempt.closed:
        normalized, raw = sms.status(attempt.message_id)
        if normalized != attempt.delivery:
            attempt.delivery = normalized
            record(attempt, "delivery_observed", status=normalized, raw=raw)
    return {
        "attempt_id": attempt.id,
        "delivery": attempt.delivery,
        "verified": attempt.verified,
        "closed": attempt.closed,
    }


@app.post("/otp-attempts/{attempt_id}/verify")
def verify(attempt_id: str, request: VerifyRequest) -> dict[str, bool]:
    attempt = attempts.get(attempt_id)
    if attempt is None:
        raise HTTPException(404, "Attempt not found")
    if attempt.closed or time.time() >= attempt.expires_at:
        if not attempt.closed:
            attempt.closed = True
            record(attempt, "attempt_closed", reason="expired")
        raise HTTPException(410, "Attempt expired")

    attempt.verify_tries += 1
    valid = hmac.compare_digest(
        attempt.code_digest, digest(attempt.id, request.code)
    )
    if not valid:
        record(attempt, "verification_failed", try_number=attempt.verify_tries)
        if attempt.verify_tries >= MAX_VERIFY_TRIES:
            attempt.closed = True
            record(attempt, "attempt_closed", reason="try_limit")
        raise HTTPException(400, "Invalid or expired code")

    attempt.verified = True
    attempt.closed = True
    record(attempt, "verification_succeeded")
    record(attempt, "attempt_closed", reason="verified")
    return {"verified": True}
```

Install FastAPI and an ASGI server, save the example as `app.py`, then run it locally:

```bash
python -m pip install fastapi uvicorn
python -m uvicorn app:app --reload
```

The in-memory dictionary is instructional, not a deployment store. In production, put attempts and events in a transactional database, enforce uniqueness for event identities, and encrypt sensitive fields according to the system's threat model. Generate the server secret outside the process and rotate it through a managed secret lifecycle. Also validate E.164 input instead of accepting any string.

## Polling is a bounded job, not a request habit

The GET handler polls inline to keep the example short. A production service should schedule status checks after acceptance and stop on a terminal result or a fixed deadline. Persist the next-check time so a process restart does not reset the schedule. Add jitter to spread bursts, and make each observation idempotent because workers, queues, and callbacks can all deliver duplicates.

Do not poll forever. For example, a policy might schedule checks at increasing intervals until the OTP's 300-second validity window ends, then close the attempt even if the messaging network still reports an intermediate state. That interval sequence is a local design choice, not a universal delivery guarantee. Tune it with observed latency distributions and the provider's published rate limits.

Callbacks and polling can coexist behind the same normalizer. Whichever arrives first writes a delivery observation; a later duplicate with the same normalized and raw state changes nothing. A later terminal state may still be useful after expiry for delivery analysis, but it cannot reopen authentication.

Keep failure classes precise. A provider rejection, an unreachable destination, an expired code, a wrong code, and an internal timeout require different evidence and different user recovery. Collapsing all five into `failed` creates misleading support data and makes an audit trail hard to interpret.

## How should failed sends change the login path?

They should change recovery, not weaken authentication. Return a consistent response when an account or phone lookup fails so the endpoint does not reveal registered sellers. OWASP recommends consistent messages and timing for account-recovery flows, plus protections against excessive submissions; the same controls are useful around OTP issuance.

Rate-limit by several dimensions, including account, destination, IP risk, and active attempt. Invalidate older codes when policy permits only one live attempt, but preserve their close events. A reissued code should create a new attempt rather than overwrite message identifiers on the old one. This makes the sequence reviewable and prevents a delayed code from being mistaken for the current challenge.

SMS also has a security boundary that the UI should not disguise. NIST's digital identity guidance treats use of the public switched telephone network for out-of-band authentication as restricted and calls for consideration of risks such as device swap or number porting. For accounts with material operational impact, offer a stronger authenticator and use SMS as one policy-controlled option, not silent proof of identity.

One sharp rule helps: **never convert delivery confidence into identity confidence.** A carrier event can decide whether to offer another code or another channel. It cannot decide whether a seller may view or acknowledge an order.

There is a real trade-off here. This polling pattern is not suitable when the team cannot operate durable jobs, protect an evidence store, or review delivery semantics as upstream mappings change; a managed identity service or an authenticator based on public-key credentials is the better boundary in that situation. It is also excessive for a notification that grants no access and carries no compliance requirement. On the other side, a callback-only design is a poor fit when callbacks can be delayed or blocked and the business still needs a bounded account of each order-access attempt. The six-event model buys traceability at the cost of another stateful workflow, retention decisions, and adapter contract tests. Choose it because that evidence is required, not because six states sound tidy.

## Evaluate evidence, latency, and prompt cost together

This backend deserves an eval harness even though it contains no model call. Build deterministic cases for accepted-then-delivered, immediate rejection, unknown message ID, duplicate observation, status regression, expiry before delivery, five wrong codes, valid code before delivery, and concurrent verification. Assert both the API result and the exact event sequence. The event assertion catches compliance regressions that a happy-path status check misses.

If an agent or language model later summarizes delivery evidence for support, keep it downstream of the authoritative records. Feed it a minimal structured projection, require event IDs in the output, and evaluate unsupported claims. Do not send phone numbers, OTPs, or message bodies into a prompt merely to explain a state transition. That reduces exposure and token cost at the same time.

Deployment gates should include migration tests, clock-boundary tests, adapter contract tests, and log-scrubbing checks. Track acceptance-to-terminal latency by normalized outcome, attempts per seller, verification failures, expirations, and observations that never become terminal. Alert on changes from an established baseline rather than inventing a universal threshold.

The operational checklist is short enough to remain prose. Confirm that credentials and HMAC keys come from managed secrets, database writes are transactional, raw provider payloads have a retention policy, clocks are synchronized, and dashboards use redacted identifiers. Exercise provider timeout and database restart paths before release. Finally, have security and legal owners approve the authenticator policy, message wording, retention period, and jurisdiction-specific consent obligations. The architecture can preserve evidence; it cannot choose those policies for the business.

## Sources

References used for the security and messaging assertions:

- OWASP, Forgot Password Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- NIST, Digital Identity Guidelines, Authentication and Authenticator Management: https://pages.nist.gov/800-63-4/sp800-63b.html
- RFC 6238, TOTP: Time-Based One-Time Password Algorithm: https://www.rfc-editor.org/rfc/rfc6238
- Yahoo, Sender Best Practices and Requirements: https://senders.yahooinc.com/best-practices/
