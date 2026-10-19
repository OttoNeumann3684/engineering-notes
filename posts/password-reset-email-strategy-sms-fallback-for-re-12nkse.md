# Password Reset Email Strategy: SMS Fallback for Reliable Financial Report Access

TL;DR: Keep email as the default recovery channel for a service that sends generated financial reports as attachments. Add SMS only for users who already verified a phone number and explicitly enabled it, and treat it as a separate recovery attempt rather than an automatic reaction to a slow email. The least complex reliable design is a small state machine: create one short-lived reset challenge, enqueue email once, accept delivery evidence asynchronously, and expose a user-initiated SMS option after a fixed waiting state. Never put the reset secret in logs, attachment metadata, or delivery analytics.

This choice is less about comparing message prices than limiting ambiguity. An email-only path has fewer identity attributes, consent records, templates, queues, and failure states. SMS may improve reach when a mailbox is inaccessible, but automatic fallback can send two usable links, leak account existence through timing, or route a credential-recovery message to a reassigned phone number. For a financial-report workflow, reliability means that an authorized user can recover access without creating a second, poorly observed authentication path.

## Should a password reset email strategy use SMS fallback?

Start with the data flow. A user requests recovery without the API revealing whether the account exists. The application creates a challenge record, stores only a digest of the random token, and emits an email job with an idempotency key. A worker renders the recovery message and sends it. Provider events may update delivery evidence, but they do not decide whether SMS is sent. After the waiting state, the same generic screen can offer SMS if the account has a previously verified, eligible number and the user enabled that route. Either channel redeems the same challenge family, so the first successful reset invalidates every outstanding attempt.

That distinction matters. A temporary SMTP response, a delayed event, and a message in a spam folder are different observations, yet none proves the user cannot receive email. Triggering SMS from any one of them couples authentication behavior to noisy transport telemetry. User-initiated escalation is slower by one click, but its behavior is explainable and testable. **Delivery events are evidence, not authorization.**

Wait for intent.

The report email itself should remain a different message class and job. It carries the generated attachment; recovery email carries a reset link and no report. Separate templates, queues, retention settings, and metrics prevent a retry of a large report attachment from accidentally becoming a recovery signal. They also make a simple question answerable during an audit: which system sent access credentials, and which system sent customer data?

## A runnable recovery state machine

The following example is intentionally compact. It omits the HTTP framework and transport adapters, which makes the decision logic runnable in a notebook and portable into a Node.js service. The production boundary is a queue-backed `send` implementation, not a direct network call inside the request handler.

```python
from dataclasses import dataclass, field
from datetime import datetime, timedelta, timezone
from enum import Enum
from hashlib import sha256
import secrets
from typing import Protocol


class Channel(str, Enum):
    EMAIL = "email"
    SMS = "sms"


class Sender(Protocol):
    def send(self, *, destination_ref: str, body: str, key: str) -> str: ...


@dataclass
class Recovery:
    challenge_id: str
    token_digest: str
    expires_at: datetime
    email_job_id: str | None = None
    sms_job_id: str | None = None
    redeemed_at: datetime | None = None
    events: list[dict[str, str]] = field(default_factory=list)

    def active(self, now: datetime) -> bool:
        return self.redeemed_at is None and now < self.expires_at


def begin_recovery(account_ref: str, email_sender: Sender) -> tuple[Recovery, str]:
    now = datetime.now(timezone.utc)
    token = secrets.token_urlsafe(32)
    challenge = Recovery(
        challenge_id=secrets.token_hex(16),
        token_digest=sha256(token.encode()).hexdigest(),
        expires_at=now + timedelta(minutes=20),
    )
    key = f"recovery:{challenge.challenge_id}:email:v1"
    challenge.email_job_id = email_sender.send(
        destination_ref=account_ref,
        body=f"Use the recovery token: {token}",
        key=key,
    )
    challenge.events.append({"kind": "email_queued", "at": now.isoformat()})
    return challenge, token


def request_sms_backup(
    challenge: Recovery,
    *,
    account_ref: str,
    phone_verified: bool,
    sms_enabled: bool,
    sms_sender: Sender,
    now: datetime,
) -> None:
    if not challenge.active(now):
        raise ValueError("Recovery challenge is not active")
    if not (phone_verified and sms_enabled):
        raise PermissionError("SMS recovery is not eligible")
    if challenge.sms_job_id is not None:
        return

    key = f"recovery:{challenge.challenge_id}:sms:v1"
    challenge.sms_job_id = sms_sender.send(
        destination_ref=account_ref,
        body="A recovery attempt was requested. Open the app to continue.",
        key=key,
    )
    challenge.events.append({"kind": "sms_queued", "at": now.isoformat()})
```

The SMS body deliberately avoids embedding the token. A better mobile flow opens the first-party application, where the server can continue the challenge after another check. If the product must send a link, keep the same one-time redemption and expiry rules across channels; do not mint an independent password-reset capability just because the transport changed.

There are three identifiers with different jobs. `challenge_id` correlates safe operational events. The token is shown only to the intended recipient, while the database retains its SHA-256 digest. The idempotency key stops a worker retry from creating a second logical send, assuming the transport adapter persists and honors that key. The code does not claim that queue acceptance means delivery. Good.

In a real service, the public response should be identical for known and unknown accounts, and rate limits should cover account, network, device, and challenge dimensions. The example uses a 20-minute expiry as an explicit test fixture, not a universal security standard. Choose the production duration through a threat review, record the decision, and test both sides of the boundary.

## Threat modeling beats channel comparison

Email-only is the sensible baseline when users have stable mailbox access, the support team can handle rare recovery exceptions, and adding a phone number would collect identity data solely for fallback. SMS backup becomes defensible when loss of mailbox access is a demonstrated recovery problem and the product can maintain verified-number state, user choice, revocation, abuse controls, and comparable audit evidence.

The dangerous design is “email failed, therefore text.” What does failed mean? Queue rejection is immediate and actionable. A bounce event can arrive later. No open event proves almost nothing because image loading may be blocked, and an open event does not prove the recipient can complete recovery. A timeout proves only that the application waited. None of those observations establishes control of the registered phone. Picture the awkward sequence: the email worker receives a temporary rejection, the application starts its timer, the mailbox accepts the retry, and the delivery event is delayed in transit. If the timer independently launches SMS, the user now receives two recovery prompts even though every component behaved within its contract. A second worker retry can multiply the problem unless both transports share durable idempotency records. This is why the state transition belongs to the recovery service and requires a user action; transport callbacks may annotate that state, but they cannot silently expand the user's authentication surface.

A threat model should cover at least duplicate sends, account enumeration, mailbox compromise, recycled phone numbers, SIM-swap exposure, replay, delayed webhooks, and support-agent override. The controls interact. Masking the destination in the UI reduces disclosure, but an overly precise mask can still identify a person. A generic response reduces enumeration, but materially different response times can restore the signal. Hashing tokens protects them at rest, but analytics that capture full reset URLs can put the secret back into a less protected system.

One correction often changes the architecture: the fallback channel is not a backup queue. It is another authenticator-adjacent route. That makes eligibility a policy decision made before an incident, not an improvisation made after a delivery event.

That is the boundary.

For generated financial reports, include attachment behavior in the same model without mixing the credentials. Report jobs should record document version, recipient reference, attachment checksum, attempt number, and final disposition. Recovery jobs should record challenge reference, selected channel, policy decision, and redemption outcome. Neither record needs the raw message body or reset token. This narrower telemetry is cheaper to retain and safer to inspect during an incident.

## Template ownership and compliance evidence

Keep message templates in source control, render them with fixtures, and snapshot the important variants. Mustache is useful for logic-light templates: variables are HTML-escaped by default, while triple braces or an ampersand produce unescaped output. That boundary deserves a test because a display name, support label, or generated report title may contain characters that should not become markup. Recovery URLs should be constructed and validated by application code, then passed to the template as data.

```python
def recovery_template_data(*, display_name: str, recovery_url: str) -> dict[str, str]:
    if not recovery_url.startswith("https://accounts.example.invalid/"):
        raise ValueError("Unexpected recovery origin")
    return {
        "display_name": display_name,
        "recovery_url": recovery_url,
        "purpose": "account recovery",
    }
```

Compliance should be represented as evidence, not a label pasted onto the transport. The FTC explains that CAN-SPAM applies according to a message's primary purpose and distinguishes commercial content from transactional or relationship content; it also states that messages must not contain false or misleading routing information. A team should document why each template is classified as it is, who approved that classification, and which tests ensure later edits do not blend promotional copy into a recovery or report-delivery message.

For EU recipients, do not infer that the US classification settles the analysis. Record the purpose of processing, data fields, retention, processors, recipient region, user preference, and approval owner, then have the applicable policy reviewed by qualified counsel. The engineering requirement is concrete even when legal conclusions depend on context: every send needs a traceable policy version, and every stored phone number needs a defined purpose and deletion path.

Avoid claiming “compliant” because a provider exposes a checkbox. Providers can carry messages and emit events; the application still decides the purpose, content, eligibility, retention, and response to a rights request. This is also why vendor pricing should not lead the design. The material cost is the second path's policy surface, abuse monitoring, support training, test matrix, and on-call burden.

The trade-off is explicit. Email-only is a poor fit when mailbox loss is common and support cannot verify users safely; SMS fallback is a poor fit when phone ownership is not maintained, users cannot pre-enroll, or the team cannot monitor abuse. In the latter case, choose a reviewed support recovery process or another previously enrolled factor rather than pretending a text is automatically safer.

No checkbox settles it.

## Evaluate the path before shipping it

Notebook-to-production work goes better when the state machine has an eval set before transport credentials exist. Build fixtures for a known account, unknown account, expired challenge, already redeemed challenge, duplicate email job, ineligible SMS request, repeated SMS request, late bounce, and a report attachment retry. Assertions should cover public response equivalence, one active challenge family, one logical send per channel, secret-free logs, and deterministic policy reasons.

Prompt-generated report content adds another boundary. The model should never choose recipients, recovery eligibility, or fallback timing. Validate the generated report, freeze the attachment bytes, calculate a checksum, and hand a reference to the delivery worker. This keeps model retries from becoming email retries and makes token cost measurable independently of transport behavior. An eval failure can stop document release without changing account recovery state.

Measure queue age, accepted jobs, transport rejections, delayed and bounced messages, user-requested escalations, successful redemptions, expired challenges, duplicate suppression, and support-assisted recoveries. Segment by channel and policy version. Do not call an email “delivered” merely because the worker handed it to the next server; name the observed state precisely.

The release gate should exercise concurrency. Two workers may claim the same job. An email event may arrive after SMS was requested. A reset may be redeemed while another event is being processed. Use a transaction or compare-and-set operation around redemption and escalation, then test those races repeatedly. Tiny happy-path tests miss the expensive failures.

## Operational acceptance

Before launch, an operator should be able to trace a report job and a recovery challenge by opaque identifiers without seeing report contents, addresses, phone numbers, or tokens. Retry policy must cap attempts, distinguish permanent from temporary transport errors, and route exhausted work to a reviewed queue. Alerts should describe growing queue age or rejection rate rather than announce that users “did not receive” a message the system cannot observe end to end.

Runbook drills should cover revoking an active challenge, disabling one channel, rotating signing material, replaying a provider event safely, and answering a user who reports an unexpected text. Template changes need fixture renders and a policy review when purpose or content changes. Phone verification and opt-out state need tests independent of the send adapter.

**Choose email-only until the evidence justifies another recovery path.** When SMS is justified, make it pre-enrolled, user-initiated, idempotent, and subordinate to the same one-time challenge. The result is not the channel with the fewest theoretical outages. It is a recovery system whose decisions remain understandable when queues lag, events arrive out of order, and a financial report is waiting.

## Further reading

References used:

- Mustache template syntax manual: https://mustache.github.io/mustache.5.html
- FTC, CAN-SPAM Act compliance guide for business: https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business
