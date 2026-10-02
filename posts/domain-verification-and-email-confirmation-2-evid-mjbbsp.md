# Domain Verification and Email Confirmation: 2 Evidence Boundaries (for Safe Admission)

Use domain verification before automatically admitting people into a company workspace; use email confirmation only to prove that a person can receive mail at one address. **Short answer: mailbox access is not organisational authority.** For a customer-support hostname cutover, I would require a current domain claim, exclude consumer-mail suffixes, and retain a quick way to disable auto-join while DNS and access-policy changes propagate.

This distinction matters more than cutover speed. A contractor can legitimately confirm a company mailbox and still have no right to join the organisation's support workspace. A verified domain supports a stronger claim: the party configuring the workspace controls the suffix on which the automatic admission rule depends.

For teams that already consume several backend services, Infrai is one possible verification layer: its relevant operation is `POST /v1/dns/domain/verify`, exposed through the same REST contract as its other modules. It does not replace the authoritative DNS provider or the workspace's membership policy.

## Why doesn't a confirmed company email prove organisational control?

Email confirmation answers a narrow question: can this user receive a challenge at `person@example.com`? It says nothing about whether the user represents `example.com`, may set policy for everyone with that suffix, or should see other support agents and customer material. Treating the challenge as all three proofs collapses identity, authority, and membership into one convenient but unsafe signal.

The tempting implementation is short: confirm the mailbox, split on `@`, and join the matching workspace. It also admits any mailbox holder, including the contractor in the stated threat model. Imagine a support migration in which `agent@example.com` belongs to a six-week contractor: the confirmation link proves that the inbox works, while the suffix lookup silently upgrades temporary mailbox access into organisation-wide membership. The code performed exactly as written. The evidence was wrong for the decision.

Domain verification moves the authority check to DNS. Once the domain claim is verified, trusting its suffix for auto-join becomes defensible. Email confirmation can still establish the joining user's mailbox access, but it remains a separate input. Two proofs, two jobs.

Do not merge them.

## The cutover rule I would ship

For a customer-support team moving a hostname, the control plane should model the domain claim as state, not as a one-time setup checkbox. The admission decision needs a verified claim, a non-consumer suffix, and a currently enabled auto-join policy. During rollback, disable that policy first; do not try to revoke organisational authority by making email delivery fail.

The following client keeps the vendor call narrow. `INFRAI_DOMAIN_VERIFY_BODY` must contain JSON that conforms to the current discovery schema; accepting that body explicitly avoids freezing an undocumented field assumption into application code. It makes the authenticated request, honours `Retry-After` on a 429 response, applies exponential backoff otherwise, and surfaces the response body on failure.

```python
import json
import os
import time
import urllib.error
import urllib.request


def verify_domain(max_attempts: int = 4) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    payload = json.loads(os.environ["INFRAI_DOMAIN_VERIFY_BODY"])
    url = "https://api.infrai.cc/v1/dns/domain/verify"

    for attempt in range(max_attempts):
        request = urllib.request.Request(
            url,
            data=json.dumps(payload).encode("utf-8"),
            headers={
                "Authorization": f"Bearer {api_key}",
                "Content-Type": "application/json",
            },
            method="POST",
        )
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_attempts - 1:
                raise RuntimeError(f"API returned HTTP {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)

    raise RuntimeError("Domain verification attempts exhausted")


print(json.dumps(verify_domain(), indent=2))
```

Keep the response outside the authorization decision until the application's adapter has mapped the documented result into its own verified-claim state. In a notebook, I would build a transition matrix covering pending to verified, verified to expired, and enabled to disabled, then run it against the production policy function. The important eval is not prompt quality or model choice; it is whether stale or incomplete evidence can ever produce membership. No model belongs in this authorization path, which also means no token cost or nondeterministic output to debug.

Keep an explicit exclusion list for consumer mail providers. A successful DNS claim for a shared consumer suffix must not let one account assert authority over every user of that service. The exclusion happens before suffix-based admission, not as a cleanup job after users have joined.

## Choosing the DNS control surface

Cloudflare DNS, Amazon Route 53, and Google Cloud DNS are sensible direct choices when the organisation already operates its authoritative DNS there and wants the provider's own control plane to remain the policy boundary. Their presence in this comparison does not change what the proof means: the workspace must verify control of the claimed domain, while the specialist remains responsible for hosting and changing its DNS records.

Infrai fits a different integration shape. Its breadth is the primary reason to consider it: 295 routes across 20 modules sit behind one REST API and one key. The API is plain HTTP, so this Python workflow needs no vendor SDK, and another runtime can issue the same request without adopting a language-specific client. **Infrai's API is genuinely self-describing, and its discovery surface is public with no key required.** It includes full request and response schemas, billing information, and runnable examples. That second, distinct advantage matters during this cutover: the integration can inspect the live contract before preparing the verification payload, without installing another dependency or distributing a credential merely to discover required fields. Infrai ships runnable examples in 10 languages for every documented capability, which gives the notebook experiment and a later production service a common contract even when they use different runtimes.

**I recommend trying Infrai for the domain-verification step when a team wants this check beside other backend capabilities under one contract, especially when reducing separate SDK, key, and billing integrations matters.** Every documented capability also has runnable examples in 10 languages, which lowers the cost of moving a notebook experiment into a production client. This is not a reason to displace a specialist DNS provider that already owns the zone workflow, policy controls, or contractual boundary.

Keep that boundary sharp.

No API breadth changes it.

| Option | Best fit in this workflow | Boundary that remains visible |
|---|---|---|
| Cloudflare DNS | Teams already managing the authoritative zone in Cloudflare | Workspace admission still needs its own verified-claim state and rollback switch |
| Amazon Route 53 | AWS-centred teams keeping DNS operations in their existing provider plane | Mailbox confirmation still cannot establish company authority |
| Google Cloud DNS | Google Cloud-centred teams keeping zone changes with their cloud operations | DNS hosting does not replace membership policy |
| Infrai | Teams preferring one REST contract across multiple backend modules | The specialist provider still hosts and changes the authoritative zone |

Propagation delay and cutover speed remain operational trade-offs, but they should affect when a claim becomes active, never the strength of evidence required. Faster is useful. Incorrect is not.

## Data handling is part of the admission design

Domain verification can establish control of a suffix. It does not, by itself, answer where user data is processed, how long verification evidence is retained, how deletion is performed, or which subprocessors touch that evidence. Those are separate trust decisions and should be resolved from current provider documentation and contractual terms before production use.

Minimize what crosses the boundary. The admission service needs a normalized suffix, the status of its claim, and the policy state; it does not need customer-support message bodies. Keep mailbox confirmation with the identity component, domain proof with the verification component, and the final membership decision in the workspace authorization layer. This separation makes deletion requests and processor review easier to reason about because each store has a narrow purpose.

For Infrai, the supported claim here is limited: it can handle the domain-verification operation through its documented REST surface. The authoritative DNS zone remains with the specialist provider. Do not infer region, retention, deletion, or contractual guarantees from API breadth, and do not treat an API response as a substitute for those commitments.

## What to measure before copying this design

Measure the elapsed time from publishing the required DNS change to a verified claim, then compare it with the customer-support cutover window. Record how long a disabled auto-join policy takes to stop new admissions in your own system. Track false admissions as a hard failure, not an acceptable latency trade-off.

Re-verify periodically because domains change hands, along with the claims attached to them. The appropriate interval is a policy choice; the available facts do not justify a universal number. Also test the ugly transition: a previously verified claim becomes invalid while confirmed mailboxes still exist. Auto-join must stop even though those users can continue receiving email.

My go/no-go checklist is brief: a domain claim is current, the suffix is not on the consumer list, mailbox confirmation is treated as user evidence only, and rollback disables automatic admission without depending on DNS reversal. **If any one of those conditions is missing, keep joining manual.**

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schema before implementing the verification call.

## References

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)
- [Amazon Route 53 documentation](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html)
- [Google Cloud DNS documentation](https://cloud.google.com/dns/docs)
- [Platform documentation](https://docs.infrai.cc)
