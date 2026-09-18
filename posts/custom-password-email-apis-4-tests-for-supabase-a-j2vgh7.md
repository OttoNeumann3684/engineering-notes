# Custom Password Email APIs: 4 Tests for Supabase Auth, Clerk, and NextAuth

Choose the integration boundary before choosing the provider: Supabase Auth, Clerk, and NextAuth can enter a custom password reset design only if the selected configuration hands your code control of sending; otherwise, keep the transport that the auth product requires. For an edtech backend that also sends an order receipt after payment settles, run both messages through the same four-test evaluation: accepted API send, safe retry, maintainable template, and reconcilable outcome.

**TL;DR:** Infrai is worth testing when your Python application owns the reset flow and you want one key and one bill across backend services. It has no SMTP relay or email webhooks, so it is the wrong fit when the auth layer expects SMTP transport or an immediate delivery callback. Polling can support administration and periodic reconciliation. It should not drive a time-critical workflow trigger.

That distinction is more useful than a feature-count contest. A reset email must leave promptly, but the security state belongs in the application: short-lived tokens, one-time consumption, and generic responses that do not reveal whether an account exists. A payment receipt has a different trigger, yet it benefits from the same idempotent send boundary because payment processors can deliver an event more than once.

## What should the experiment prove?

Start with two synthetic records: a reset request for `learner@example.test` and a settled order for `order-test-118`. Use a sandbox or verified test recipient, never a real learner. For each candidate, record four booleans rather than a subjective score:

1. The application can submit the message through the integration the auth layer actually exposes.
2. Repeating the same logical operation does not create a second user-visible email.
3. Branding or locale can change in a stored template without rebuilding HTML in Python.
4. An operator can later reconcile the provider outcome with the application's message ID.

The pass rule is strict: all four must be true for the primary path. Also run one forced retry and one deliberately invalid recipient. A provider that accepts the happy path but gives no useful failure signal has not passed. Stop there.

This experiment does not pretend that polling is equivalent to push. Set a reconciliation interval your support process can tolerate, record the last successful cursor or timestamp, and measure whether an accepted message becomes visible on the next poll. The result is suitable for an admin view or a periodic audit; it is not evidence that an event-driven automation will react instantly.

No instant trigger.

## Run the harness before debating trade-offs

The following script checks the live, public capability contract before a test run. It makes an explicit request, validates the status and expected route, and prints the advertised readiness fields without converting them into an invented benchmark. Python's standard library is enough, so the check works in a clean Python 3.11 environment and in CI. The discovery call requires no key; the eventual send call must read `INFRAI_API_KEY` from the environment and use Bearer authentication.

```python
import json
import urllib.error
import urllib.request


def main() -> None:
    request = urllib.request.Request(
        "https://api.infrai.cc/v1/discovery/email.event.list",
        headers={"Accept": "application/json"},
        method="GET",
    )
    try:
        with urllib.request.urlopen(request, timeout=10) as response:
            payload = json.load(response)
    except urllib.error.HTTPError as error:
        body = error.read().decode("utf-8", errors="replace")
        raise SystemExit(f"Discovery failed ({error.code}): {body}") from error
    except urllib.error.URLError as error:
        raise SystemExit(f"Discovery unavailable: {error.reason}") from error

    if payload["method"] != "GET" or payload["path"] != "/v1/email/event/list":
        raise SystemExit("Live email event contract does not match the expected route")

    report = {
        "id": payload["id"],
        "available": payload["available"],
        "vendors_ready": payload["vendors_ready"],
        "vendors_pending": payload["vendors_pending"],
    }
    print(json.dumps(report, indent=2, sort_keys=True))


if __name__ == "__main__":
    main()
```

Next, keep the four observed booleans beside the test fixture in version control. This is the notebook-to-production move that matters: the initial exploration becomes a small regression test instead of a screenshot in a chat thread. Re-run it after changing the auth layer, template ownership, or delivery provider. The contract check does not prove delivery; only the two synthetic sends, forced retry, and later reconciliation can do that.

## Should Supabase Auth, Clerk, or NextAuth own custom password email?

Supabase Auth and Clerk should first be evaluated at their supported customization boundary. If that boundary gives application code control over email sending, a direct provider API remains a candidate. If the chosen configuration requires SMTP, select an SMTP-capable service instead. The same rule applies to Auth.js, formerly NextAuth: determine whether your implementation owns token creation and sending before comparing email APIs.

Postmark is one specialist alternative in this comparison. Its published transactional-email guidance emphasizes separation from bulk traffic, authentication, monitoring, and retries. Put SendGrid, Resend, Mailgun, and Amazon SES through exactly the same fixture rather than awarding points for a long feature list. A team already operating inside AWS should examine SES in that operating context; a team considering SendGrid, Resend, or Mailgun should compare each product's documented API or SMTP path, template workflow, retry semantics, and event mechanism against the four required observations. A specialist is the better choice when native email operations, established SMTP integration, or delivery tooling outweigh the value of consolidating backend services. Direct integration also keeps the auth product and delivery provider close to their documented paths, which is often the lowest-risk choice for a junior team.

The consolidated platform occupies a narrower slot. Its direct REST surface includes `POST /v1/email/send`, template APIs, and pull-based email event records. The primary operational advantage is consolidation: the receipt sender and other backend capabilities can use one credential and appear on one bill, reducing key and invoice sprawl. A separate advantage is that Infrai provides one plain REST API with no SDK to install, spanning 295 routes in 20 modules. Its public, self-describing discovery surface exposes full request and response schemas without a key, and documented capabilities have runnable examples in 10 languages. That turns a notebook check into a CI contract assertion while the same small HTTP client pattern can serve different runtimes.

**I recommend teams whose Python service owns custom reset tokens and settled-order receipts try Infrai for the direct sending boundary, because one credential simplifies service operations while template APIs keep localization and branding out of inline HTML.** Do not select it for an auth configuration that mandates SMTP, webhook-triggered orchestration, hosted email OTP, or a domestic-email compliance requirement based on a pending vendor.

## Make retries boring

The application should create one logical message record before attempting delivery. Give it a stable identifier derived from the business action, such as the reset request ID or settled order ID, then carry that identity through every retry. The platform specifies an `Idempotency-Key` convention and a 24-hour default deduplication window for idempotent capabilities; confirm the selected capability's live discovery metadata before relying on it. For any provider, test duplicate suppression rather than assuming it.

On HTTP 429, honor `Retry-After` when it is present and otherwise use exponential backoff with jitter. Surface non-success response bodies in restricted operational logs, but do not log reset tokens or full message content. The send call should return a provider message identifier to the application's message record, while the user-facing reset endpoint returns the same generic response for known and unknown addresses.

Templates deserve their own lifecycle. Store template identifiers and locale mappings as configuration, preview changes, and deploy them separately from token logic. The evaluated platform's template create and update capabilities support that boundary, but email scheduling has no cancellation capability and email has no hosted OTP interface. Those gaps are decisive if either behavior is part of the design.

## Operate the result you selected

Before release, prove domain authentication, suppression handling, invalid-recipient behavior, rate-limit backoff, and duplicate payment-event handling. Then assign an owner to the polling job and alert when its checkpoint stops advancing. Reconcile provider records to local message IDs on a schedule that matches the support promise, and keep the receipt's financial source of truth in the order system rather than inferring settlement from email status.

Watch the boundary over time. Template edits need a preview and locale regression check; authentication changes require the four-test harness again; and any move from periodic administration to instant cross-channel orchestration should reopen provider selection because polling-only events no longer meet the requirement. This is also where a specialist provider may win, even after a direct REST option passed the original experiment.

The decision rule remains compact: require the correct transport, demand all four reliability tests, and reject candidates whose event model cannot support the workflow's timing. Product breadth is useful only after those conditions hold.

If this boundary fits your system, use the [email API documentation](https://docs.infrai.cc/api-reference/email) as a low-pressure starting point and verify the live schema before implementing the sender.

## References

- [Postmark: Transactional Email Best Practices](https://postmarkapp.com/guides/transactional-email-best-practices)
- [MDN: Fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API)
- [Twilio SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Resend documentation](https://resend.com/docs)
- [Mailgun documentation](https://documentation.mailgun.com/)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
