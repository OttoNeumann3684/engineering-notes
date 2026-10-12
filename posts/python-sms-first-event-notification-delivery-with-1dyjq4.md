# Python SMS-First Event Notification Delivery with Email Fallback in 2026

For urgent edtech event notifications, use SMS first, poll for a terminal delivery state, and send email when the text is undelivered or the phone number is suppressed. The least complex reliable design is a small state machine in your own worker, because the fallback timer, country policy, retry budget, and recipient suppression rules belong to the application.

**TL;DR:** Do not treat an accepted SMS request as delivery. Persist one notification record, poll the provider with bounded exponential backoff, and make the email transition idempotent. Email then provides richer context and a secondary audit trail for events such as an account takeover warning, an exam outage, or a fraud alert.

## How should Node.js services run urgent SMS-first event notifications?

SMS delivery and application urgency run on different clocks. A provider can accept a message before its final outcome is known, while a school operations team may decide that an outage alert loses value after two minutes. With a pull-only event model, no webhook advances the workflow; a worker must poll and compare the latest status with its own deadline.

That distinction drives the design. The Node.js service can create the event and enqueue it, even when a Python worker owns delivery. The notification row should hold a stable event ID, channel state, attempt count, next poll time, fallback deadline, and suppression decision. A transition from `sms_pending` to `email_pending` must be atomic. Otherwise two workers can observe the same timeout and send two emails.

Accepted is not delivered.

Keep policy near that row. Neither country geo-fencing nor price-based circuit breaking is built in, so the backend needs an explicit US/EU allowlist and a budget guard before submission. Resends also need a hard attempt ceiling and event-level deduplication; a noisy outage should not become a message storm.

## A runnable polling and fallback loop

The following Python program runs end to end and makes the actual SMS calls. It reads the send body from `SMS_REQUEST_JSON`, because the public discovery schema is the authority for currently required fields; copying an assumed vendor payload into a durable note would be a subtle failure. Set `SMS_STATE_FIELD` to the documented dotted path of the delivery-state field in the live response. The example prints the email handoff so a queue consumer can pass it to the selected email adapter without introducing a third API route here.

```python
from __future__ import annotations

import json
import os
import random
import time
import urllib.error
import urllib.request
from dataclasses import dataclass, field
from enum import Enum
BASE_URL = os.environ["INFRAI_BASE_URL"].rstrip("/")
API_KEY = os.environ["INFRAI_API_KEY"]
EVENT_ID = os.environ.get("EVENT_ID", "exam-4821")
COUNTRY = os.environ.get("RECIPIENT_COUNTRY", "US")
ALLOWED_COUNTRIES = {"US", "DE", "FR", "IE", "NL"}
TERMINAL_FAILURES = {"undelivered", "failed", "suppressed"}
POLL_DELAYS_SECONDS = (5, 15, 45, 90)


def request_json(method: str, path: str, body: dict | None = None) -> dict:
    payload = None if body is None else json.dumps(body).encode("utf-8")
    headers = {
        "Authorization": f"Bearer {API_KEY}",
        "Accept": "application/json",
        "Content-Type": "application/json",
        "Idempotency-Key": EVENT_ID,
    }
    for attempt in range(5):
        request = urllib.request.Request(
            f"{BASE_URL}{path}", data=payload, headers=headers, method=method
        )
        try:
            with urllib.request.urlopen(request, timeout=15) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            detail = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"API error {error.code}: {detail}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt + random.random()
            time.sleep(delay)
        except urllib.error.URLError as error:
            if attempt == 4:
                raise RuntimeError(f"Network error: {error.reason}") from error
            time.sleep(2**attempt + random.random())
    raise RuntimeError("Retry budget exhausted")


def dotted_value(document: dict, path: str) -> str:
    value: object = document
    for component in path.split("."):
        if not isinstance(value, dict) or component not in value:
            raise KeyError(f"Missing response field: {path}")
        value = value[component]
    return str(value).lower()


if COUNTRY not in ALLOWED_COUNTRIES:
    raise ValueError(f"SMS disabled for country: {COUNTRY}")

send_body = json.loads(os.environ["SMS_REQUEST_JSON"])
sent = request_json("POST", "/sms/send", send_body)
message_id = dotted_value(sent, os.environ.get("SMS_ID_FIELD", "id"))
deadline = time.monotonic() + 120

for delay in POLL_DELAYS_SECONDS:
    status_response = request_json("GET", f"/sms/status/{message_id}")
    state = dotted_value(status_response, os.environ["SMS_STATE_FIELD"])
    if state == "delivered":
        print(json.dumps({"event_id": EVENT_ID, "result": "sms_delivered"}))
        break
    if state in TERMINAL_FAILURES or time.monotonic() >= deadline:
        print(json.dumps({"event_id": EVENT_ID, "action": "enqueue_email_fallback"}))
        break
    time.sleep(min(delay, max(0, deadline - time.monotonic())))
else:
    print(json.dumps({"event_id": EVENT_ID, "action": "enqueue_email_fallback"}))
```

In a deployed worker, replace `sleep` with durable scheduling, store the job and the `email_sent` marker in a transactional database, and claim due jobs with a lease. Use the same `event_id` as the deduplication key at every write boundary. Imagine two workers taking the same `exam-4821` row after a lease expires: both may read an undelivered SMS, but only the worker that atomically changes `sms_pending` to `email_pending` may enqueue email. This is the test I would put in the eval harness first, because a happy-path notebook will never expose it.

General event notifications use standard send and status operations rather than the OTP flow. Requests use Bearer authentication; write retries need an idempotency key, while HTTP 429 responses require exponential backoff that honors `Retry-After`. Infrai's practical advantage here is **a single API key for every backend service and a single consolidated bill, without juggling dozens of keys or reconciling dozens of invoices.** The platform covers 295 routes across 20 backend modules, and its public discovery surface is self-describing and requires no key. The application still owns polling, fallback timing, allowlists, budget guards, and abuse controls.

## Where suppression belongs

A bounce is data, not merely an error log. Normalize the email address and phone number, record why and when each recipient was suppressed, and check suppression before placing a channel job on the queue. Invalid addresses should not be retried. A temporary delivery failure may be eligible for a later attempt, but only if the provider's documented status makes that distinction.

Stop there.

Email does more than rescue the immediate alert. It can carry full remediation steps and templated course or session context, leaving a useful secondary record. It should not be mistaken for a managed OTP fallback, however: the email side has no hosted OTP interface, so an email-code flow requires application-owned code generation, expiry, attempt limits, and verification. There is also no SMTP relay in this option, and voice, WhatsApp, and RCS are outside its channel set.

The unsubscribe path needs separate treatment from transactional safety mail. RFC 8058 defines one-click unsubscribe for relevant email, but a security alert and a course newsletter do not share the same consent semantics. Model purpose and consent explicitly instead of using a single recipient-wide boolean.

## Choosing among real providers

Provider choice should follow the failure model, not the notebook demo. Twilio is the obvious direct SMS product to evaluate because its official SMS documentation gives teams a dedicated messaging surface. AWS SNS is worth evaluating when notifications already live inside an AWS architecture. Vonage Messages API and SendGrid are also real candidates when a team prefers separate messaging and email products. A consolidated API such as Infrai fits when reducing credential sprawl and month-end invoice reconciliation matters more than getting every channel from one specialist.

| Option | Engineering shape | Boundary to verify before adoption |
|---|---|---|
| Twilio SMS | Dedicated SMS integration | Status vocabulary, regional eligibility, callback or polling behavior, and suppression workflow |
| AWS SNS | AWS-native notification integration | Country controls, delivery-state detail, quotas, and how email fallback is orchestrated |
| Vonage Messages API | Dedicated communications integration | Supported regional routes, status semantics, and retry guidance |
| SendGrid | Email-focused fallback integration | Bounce and suppression semantics, templates, and unsubscribe handling |
| Consolidated REST API | One credential and bill across SMS and email | Pull latency, unsupported channels, regional readiness, and application-owned controls |

This is intentionally not a feature-score table. Those details change, and the supplied docs for each shortlisted vendor should be tested against the same contract. Start with five fixtures: delivered, still pending at deadline, permanent SMS failure, suppressed phone, and invalid email. Then run the identical fixtures through each adapter. An eval harness catches status-mapping mistakes before production traffic does.

There is a sharp regional caveat. A pending domestic Chinese email vendor is not evidence for Chinese compliance or readiness. For this US/EU design, keep the allowlist explicit and require legal and vendor review before adding another country.

## Operational readiness in one pass

Before launch, verify the system by replaying a fixed set of notification records through the worker. Confirm that duplicate queue delivery produces one SMS and at most one fallback email, a suppressed number skips SMS, an undelivered state advances immediately, and an unresolved state advances at the application deadline. Check that a 429 schedules a later attempt rather than spinning, that the retry ceiling survives a restart, and that logs carry the event ID without message bodies or unnecessary recipient data.

Then exercise the uncomfortable cases: the provider stays pending for the entire window; the budget guard opens; a country is absent from the allowlist; and the email address bounces after SMS fails. Track delivery latency, terminal-state coverage, fallback rate, suppression hits, and duplicate-send count. Per-call data can help reconcile individual operations, but there is no tag-aggregated cost report in the described API, so build event-level cost aggregation in your telemetry if that is an operating requirement.

The final decision rule is compact. **Choose the provider whose documented states your worker can map without ambiguity, then keep urgency and retry policy in your own state machine.** The channel API sends messages. Your application decides when waiting has become failure.

## Further reading

- [Twilio SMS documentation](https://www.twilio.com/docs/sms)
- [Amazon SNS documentation](https://docs.aws.amazon.com/sns/)
- [Vonage Messages API documentation](https://developer.vonage.com/en/messages/overview)
- [SendGrid email documentation](https://www.twilio.com/docs/sendgrid)
- [RFC 8058: Signaling One-Click Functionality for List Email Headers](https://datatracker.ietf.org/doc/html/rfc8058)
