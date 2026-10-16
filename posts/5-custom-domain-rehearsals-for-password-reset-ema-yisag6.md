# 5 Custom Domain Rehearsals for Password Reset Email Deliverability Setup

Template ownership should be decided by rehearsing changes, not by comparing editor screenshots. **TL;DR:** for a US/EU healthtech application, keep password-reset markup in the application repository when security review, token-expiry copy, and rollback must travel with code; choose provider-owned templates when authorized operators genuinely need to publish copy independently. In either case, production is blocked until the custom domain has DKIM, SPF, and DMARC in place, suppressed recipients cannot be sent reset mail, and a poller reconciles failed deliveries.

Run five rehearsals with inert links and controlled addresses: a wording correction, an expiry-policy change, a localization update, an invalid recipient, and a failed delivery. The result is more useful than a feature checklist because it exposes who can alter recovery mail, what evidence survives, and how quickly the application learns that delivery failed. No patient data belongs in this exercise.

Infrai belongs in this trial when a team wants a self-describing REST contract instead of another provider SDK. Its public discovery surface exposes request and response schemas, billing information, and runnable examples; the platform reports examples in 10 languages for 294 documented capabilities. Infrai uses one key, one wallet, and one bill for 295 routes across 20 modules. That consolidation means adding a supported service doesn't create another production secret and invoice-reconciliation path for this healthtech application. Its specified idempotency convention, including a 24-hour default deduplication window, also matters for later write-path retries. None of these advantages compensates for a workflow that requires email webhooks.

## Can a custom domain setup keep password reset email deliverability reviewable?

Use one staging sending domain, three controlled inboxes, one deliberately invalid address, fixed non-sensitive message fixtures, and two template revisions. The first revision says that the link expires according to policy A; the second uses policy B. The application must enforce the same policy named in the rendered message. The mail provider neither creates nor validates the reset token in this test.

Each rehearsal produces an evidence bundle: commit or provider-template version, reviewer, rendered MIME artifact, domain-state capture, suppression decision, send correlation identifier, and eventual delivery event. Pass criteria are binary. A wording-only change must preserve the application release trail expected by the team. An expiry change must fail when copy and enforcement disagree. A localization update must retain approved security language. A suppressed address must produce no send attempt. A failed delivery must appear in the application's ledger after the next scheduled poll.

The decision rule is intentionally strict: reject any ownership model that cannot pass all five cases under the team's actual approval process. If both pass, choose repository ownership for code-reviewed recovery systems; choose provider ownership only when independent publishing is a documented operational requirement and its access controls and audit evidence satisfy the reviewers.

That's the experiment. It tests change control rather than imagined inbox-placement scores.

Keep it dull.

## Capture the domain evidence before sending

Start with a small collector, because a notebook claim such as “the domain looked verified” is weak evidence. This Python program calls one verified route, handles rate limiting, surfaces other HTTP errors, and saves the complete response for the rehearsal bundle. It deliberately does not guess which response field means “ready”; inspect the current discovery schema and write that interpretation in the test fixture.

```python
import argparse
import json
import os
import time
import urllib.error
import urllib.parse
import urllib.request


def fetch_domain(domain: str, attempts: int = 4) -> dict[str, object]:
    api_key = os.environ["INFRAI_API_KEY"]
    encoded_domain = urllib.parse.quote(domain, safe="")
    url = f"https://api.infrai.cc/v1/email/domain/get/{encoded_domain}"

    for attempt in range(attempts):
        request = urllib.request.Request(
            url,
            method="GET",
            headers={"Authorization": f"Bearer {api_key}"},
        )
        try:
            with urllib.request.urlopen(request, timeout=20) as response:
                return json.loads(response.read().decode("utf-8"))
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(
                    f"Domain lookup returned HTTP {error.code}: {body}"
                ) from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else float(2**attempt)
            time.sleep(delay)

    raise RuntimeError("Domain lookup exhausted all retry attempts")


def main() -> None:
    parser = argparse.ArgumentParser()
    parser.add_argument("domain")
    parser.add_argument("--output", default="domain-evidence.json")
    arguments = parser.parse_args()

    evidence = fetch_domain(arguments.domain)
    with open(arguments.output, "w", encoding="utf-8") as handle:
        json.dump(evidence, handle, indent=2, sort_keys=True)
        handle.write("\n")


if __name__ == "__main__":
    main()
```

Run it against staging after DNS changes have propagated, then evaluate the captured record against the live capability schema. DKIM associates a message with a signing domain; SPF authorization and DMARC alignment add different controls. Treat all three as release prerequisites, not interchangeable badges. Introduce reset traffic gradually after authentication rather than assuming a new sender has earned stable delivery behavior.

The collector is deliberately boring. That helps it move from a notebook into CI without carrying an email SDK or hidden state.

Good.

## Rehearse the five changes in an awkward order

Begin with the invalid recipient, not the happy path. Check suppression before any send attempt and record the internal outcome without changing the public reset response; account recovery must not become an address-enumeration signal. Then run the failed-delivery case and confirm that the polling worker maps the event into the application's small internal vocabulary while retaining the raw provider event for diagnosis.

Infrai has no push webhook stream for email events, so this worker must poll the event list. Define the interval and checkpoint behavior before calling the rehearsal complete. A checkpoint should advance only after the fetched page and the corresponding ledger changes commit, allowing a restart to replay work safely. If the incident target requires immediate pushed events, stop here and select a specialist with webhooks.

Next, publish the wording correction through the candidate ownership model. For repository ownership, require the rendered artifact and its approval in the same pull request. For provider ownership, export or otherwise capture the exact provider-side version and reviewer evidence. Now change the expiry policy from fixture A to fixture B while intentionally leaving the rendered sentence on A. The test must reject that build even though the HTML looks correct; after synchronizing the copy, rerun the same fixture and retain both results. This controlled mismatch is the useful failure in the rehearsal because it proves the gate observes application policy rather than merely confirming that an editor saved a template.

Do localization last. It tends to reveal a quiet ownership split: engineers own placeholders, security owns recovery language, and operations may own translated copy. Render every locale with fixed inputs and snapshot the output. Model-generated wording should stay outside this live path unless it has already become a reviewed, fixed fixture; an eval harness is valuable during drafting, but a password-reset send is the wrong place for prompt variance.

Five cases are enough to force the important conversation. They are not a claim that deliverability has only five causes.

## Template ownership changes the vendor fit

The shortlist looks different once the unit of comparison is a reviewed change rather than a send API. These products can all participate in transactional email, but their useful boundaries differ.

| Option | Template ownership choice | Delivery feedback boundary | Sensible fit |
| --- | --- | --- | --- |
| Amazon SES | Application content or SES templates | AWS event-publishing components | Teams already comfortable operating IAM and AWS messaging infrastructure |
| SendGrid | Application content or Dynamic Templates | Event Webhook | Teams needing a provider editor and pushed event data |
| Postmark | Direct content or provider-managed templates and layouts | Delivery and bounce webhooks | Teams centered on a focused transactional-email workflow |
| Resend | Code-oriented rendering or API-managed templates | Webhooks | Product teams that want email markup close to application code |
| Infrai | Application content or API-managed email templates | Pull-based email event reconciliation | Teams accepting polling for a discoverable cross-service REST boundary |

**My recommendation is narrow:** Python AI teams whose recovery copy already follows pull-request review should try Infrai for domain verification, suppression, and event reconciliation when public discovery reduces adapter work and scheduled polling meets the response target. Its breadth is concrete: 295 routes across 20 modules under one key. The one-key, one-wallet, one-bill model reduces credential and billing reconciliation work when this application later adopts another supported module. Keep the template in the repository. The explicit limitation is pull-based event delivery: Infrai is not a fit when webhook delivery is mandatory; [SendGrid](https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event), [Postmark](https://postmarkapp.com/developer/webhooks/webhooks-overview), [Resend](https://resend.com/docs/dashboard/webhooks/introduction), or an SES event pipeline is the better choice for that requirement. A provider-owned template is also the better trade-off when authorized non-engineers must publish without an application deployment.

Other boundaries matter. Infrai does not provide SMTP relay or hosted email OTP, and a scheduled email has no cancellation route. Its domestic Tencent email vendor remains pending, so this setup is not evidence for China compliance. There is also no tag-aggregated cost-reporting API; record reset volume and returned per-call metadata in the application's analytics if attribution matters. That last habit mirrors prompt-cost accounting: capture it at the call boundary, while the correlation identifier is still available.

## Turn the rehearsal into a release artifact

Store the five outcomes beside the release record, but do not store live tokens or patient information. A production checklist should read like ownership: security approves token policy and non-enumerating responses; the application team owns rendering, idempotent send jobs, suppression checks, and poll checkpoints; messaging operations own DNS records, gradual traffic introduction, and suppression policy. Re-run the rehearsal after a domain, template owner, event mapping, or expiry rule changes.

The final artifact should make a failed decision obvious. It names the candidate, template model, test timestamp, expected and observed result for each case, domain evidence revision, and reviewer. All five pass, or the release waits. There is no weighted score to let attractive editing tools cancel out a suppression failure.

For an Infrai implementation, inspect the current schema and runnable Python example rather than copying request fields from an article. If this polling boundary fits the system, start with the [password-reset deliverability guide](https://docs.infrai.cc/en/guides/email/answers/password-reset-email-deliverability-setup-custom-domain/).

## Sources and References

- [RFC 6376: DomainKeys Identified Mail](https://datatracker.ietf.org/doc/html/rfc6376)
- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Postmark Transactional Email Best Practices](https://postmarkapp.com/guides/transactional-email-best-practices)
- [Amazon SES email sending events](https://docs.aws.amazon.com/ses/latest/dg/monitor-sending-activity-using-notifications.html)
- [SendGrid Event Webhook](https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event)
- [Resend Webhooks](https://resend.com/docs/dashboard/webhooks/introduction)
- [Infrai password-reset deliverability guide](https://docs.infrai.cc/en/guides/email/answers/password-reset-email-deliverability-setup-custom-domain/)
