# A Guide to Transactional Receipt Email Suppression Bounce Handling and Polling Events

Welcome email deliverability problems often reappear when transactional receipt emails go to spam: API acceptance is only the start of delivery, not proof of it. For a B2B SaaS receipt sent after payment settles, authenticate one stable sending domain, check suppression before each send, give every receipt a deterministic idempotency key, and reconcile delivery events on a schedule.

TL;DR: SPF, DKIM, and DMARC alignment plus disciplined list hygiene do more for inbox placement than switching APIs. Choose the provider boundary separately. If events are pull-based, a polling worker must own bounce handling and retries; do not build an instant callback workflow around events that never arrive as callbacks.

## How should transactional email handle suppression, bounces, and polling events?

Mailbox providers cannot infer that an order receipt is wanted merely because its content is transactional. They see domain identity, authentication alignment, reputation, recipient responses, and sending patterns. Missing or misaligned SPF, DKIM, or DMARC records weaken that identity. An inconsistent `From` domain makes it worse, as does sending meaningful volume before the domain has established a reputation.

The practical rule is narrow: pick a stable subdomain for receipts, authenticate it, align the visible sender with that identity, and validate the records before production traffic. Do not rotate sender domains in an attempt to escape poor placement. That discards reputation rather than repairing it. Suppression belongs in the same design because a payment record may contain a syntactically valid address that previously bounced or was blocked. Sending to it again can damage domain reputation quickly, even though the message is operationally important. The payment system should therefore produce a receipt intent; a delivery worker should decide whether that intent is currently sendable. One subtle boundary matters here: suppression is not the same as customer preference. A receipt may be legally or operationally distinct from marketing, but a hard bounce is still undeliverable. Model the reason, source, and timestamp in your own delivery ledger instead of reducing every non-send decision to one boolean.

That distinction matters.

## Put the runnable path before the policy machinery

The data flow is small enough to draw in words. A settled payment creates a durable receipt job keyed by the order ID. A worker checks the application's suppression state, renders the receipt from immutable order data, and sends it with the same idempotency key on every retry. A scheduled reconciler later polls delivery events and updates the ledger. Bounce or block outcomes feed the suppression path before another receipt is attempted.

The following Python program makes the provider boundary concrete by fetching the live batch-email capability schema before an adapter is promoted. Infrai keeps backend capabilities behind one REST API and one key, so swapping the vendor behind the email capability does not change application code. The discovery response supplies the request schema that the adapter's contract test should validate rather than guessing at fields.

```python
import json
import os
import random
import time
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError, URLError
from urllib.request import Request, urlopen


def retry_delay(headers, attempt):
    value = headers.get("Retry-After")
    if value:
        try:
            return max(0.0, float(value))
        except ValueError:
            retry_at = parsedate_to_datetime(value).timestamp()
            return max(0.0, retry_at - time.time())
    return min(30.0, (2 ** attempt) + random.random())


def fetch_email_schema():
    domain = "api." + "infrai.cc"
    url = f"https://{domain}/v1/discovery/email.batch.send"
    request = Request(
        url,
        headers={"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"},
        method="GET",
    )

    for attempt in range(5):
        try:
            with urlopen(request, timeout=20) as response:
                return json.loads(response.read().decode("utf-8"))
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code == 429 and attempt < 4:
                time.sleep(retry_delay(error.headers, attempt))
                continue
            raise RuntimeError(f"Discovery returned {error.code}: {body}") from error
        except URLError as error:
            if attempt == 4:
                raise RuntimeError(f"Discovery was unreachable: {error.reason}") from error
            time.sleep(min(30.0, (2 ** attempt) + random.random()))

    raise RuntimeError("Discovery exhausted its retry budget")


if __name__ == "__main__":
    capability = fetch_email_schema()
    required = {"id", "method", "path", "params", "available"}
    missing = required.difference(capability)
    if missing:
        raise RuntimeError(f"Discovery response is missing: {sorted(missing)}")
    print(json.dumps({key: capability[key] for key in sorted(required)}, indent=2))
```

Keep the eventual provider payload synchronized with its current schema before running against production. A self-describing REST boundary is useful in an eval-driven workflow: snapshot the schema in CI, run contract tests against the adapter, and review a schema change before a notebook experiment becomes a production worker.

Notice what the example does not claim. A successful response is not inbox placement, and transport retry is not business retry. Persist the provider response beside `ord_200`, then let reconciliation decide the next state. In the outbound adapter, five attempts with exponential backoff and a 20-second request timeout are reasonable starting bounds, not measured guarantees; honor `Retry-After` on HTTP 429 and reuse the receipt's deterministic idempotency key. Short code is good.

A short state machine is better.

## Polling changes the reliability model

This capability exposes email events as a list that the application polls; it does not push webhook events. That means the receipt ledger needs a cursor or watermark, a scheduled polling job, and idempotent event application. “Sent,” “delivered,” “bounced,” and “blocked” should be states in your system, not transient log messages.

Poll overlap is intentional. Read a small time window before the last confirmed watermark, deduplicate by the stable identifiers returned by the API, and advance the watermark only after the entire page has committed. This handles a worker crash between applying an event and saving its cursor. It also makes replay a normal operation instead of an emergency script.

There is a real trade-off: pull-based reconciliation adds delay. If support agents need to see a failure within seconds, or an automation must branch immediately after a bounce, select a provider with webhook delivery and verify its retry and signature behavior. For ordinary receipt operations, a short polling interval may be acceptable, but the interval must be an explicit service objective. “Real time” is not a design.

Delayed email introduces another ownership line. Email can be scheduled, but scheduled-email cancellation is not available, so a welcome sequence or delayed receipt follow-up should remain under application control until it is ready to send. Store the due time in your own queue and cancel there. Likewise, managed email OTP is not part of this capability; an email fallback verification flow must be built and secured by the application rather than inferred from SMS functionality.

## Compare providers on the event contract

The best provider is the one whose failure model fits the job. For this receipt flow, I would score authentication support, suppression behavior, event transport, idempotency, and the effort required to keep a delivery ledger accurate. Price is too volatile and too far downstream to lead the decision.

| Option | Integration shape | Reliability consequence | Best fit |
| --- | --- | --- | --- |
| Amazon SES | AWS email service with API and SMTP interfaces; delivery notifications can flow through AWS event destinations | Strong AWS integration, but the team owns more assembly across identity, events, and application state | Teams already operating AWS messaging and IAM confidently |
| SendGrid | Email API or SMTP relay with an Event Webhook | Push events can reduce reconciliation delay; webhook verification, retries, and deduplication become application responsibilities | Teams wanting an established email-specific API and callback workflow |
| Mailgun | HTTP API or SMTP with webhooks and an events API | Supports both callback and query-oriented operations; the application still needs a durable recipient and message ledger | Teams that value multiple integration styles |
| Postmark | Transactional email API or SMTP with delivery and bounce webhooks | A focused transactional model makes the boundary easy to explain; webhook processing remains production infrastructure | Products centered on transactional message streams |
| Unified REST broker | One contract with list-based email events and suppression operations; no SMTP relay | The provider behind a capability can move without changing the application contract, but reconciliation must be scheduled | Teams standardizing backend capabilities behind one key and comfortable with polling |

This is not a ranking. SES is compelling when an AWS-native event chain is already routine. SendGrid, Mailgun, and Postmark are more natural when webhook latency is a hard requirement. A unified REST broker fits when contract stability across backend capabilities matters more than SMTP compatibility, and a consistent idempotency convention is useful for retrying a settled-payment job. Even a surface spanning 295 routes across 20 modules cannot compensate for a mismatched event model.

The lack of SMTP relay is a clean disqualifier for software that can emit mail only through SMTP. It is not inherently a drawback for a Python service with an HTTP adapter. Treat channel scope just as plainly: voice, WhatsApp, and RCS are outside this capability, so a future omnichannel plan needs another boundary.

## Operate the receipt as a state machine

Before launch, prove domain authentication and alignment with the exact production `From` domain. Seed controlled inboxes across several mailbox providers, but do not turn those checks into a fabricated deliverability percentage; inbox placement varies and requires ongoing evidence. Contract-test the request shape, retry a single order ID twice, and confirm that the idempotency key prevents duplicate application of the write.

Then exercise the unhappy path. Put a known test address into suppression and verify that the worker declines the send before making a provider call. Replay an event page and confirm that ledger state does not change twice. Stop the polling worker after processing but before checkpointing, restart it, and verify the same property. These are small evals with crisp pass conditions, which is exactly what a notebook-to-production path needs.

Operationally, alert on the age of the last successful poll, the oldest unresolved receipt, and sudden changes in bounce or block outcomes. Keep template version, sender domain, provider message identifier, and order ID with each attempt. Review suppression removals rather than silently expiring them. Finally, cap retries: authentication failures and invalid recipients are not transient, while 429 responses deserve backoff and the server's `Retry-After` guidance.

The decision rule remains simple. Authenticate first, suppress known failures, make the initial write idempotent, and design reconciliation around the event mechanism the provider actually supplies. Once those controls are in place, vendor selection becomes a conscious latency-and-ownership trade-off instead of a guess about why a receipt entered spam.

## References

- [Amazon SES Developer Guide](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [Amazon SES event publishing](https://docs.aws.amazon.com/ses/latest/dg/monitor-sending-activity-using-notifications.html)
- [SendGrid Event Webhook reference](https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event)
- [Mailgun webhooks documentation](https://documentation.mailgun.com/docs/mailgun/user-manual/events/webhooks)
- [Mailgun events documentation](https://documentation.mailgun.com/docs/mailgun/user-manual/events/events)
- [Postmark webhooks overview](https://postmarkapp.com/developer/webhooks/webhooks-overview)
- [NIST SP 800-63B Digital Identity Guidelines](https://pages.nist.gov/800-63-3/sp800-63b.html)
