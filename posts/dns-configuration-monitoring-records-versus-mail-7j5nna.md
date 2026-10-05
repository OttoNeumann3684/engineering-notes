# DNS Configuration Monitoring: Records Versus Mail Acceptance Outcomes During Game Cutovers

A game studio cannot call a mail cutover complete merely because the intended MX record appears in its control plane. The release constraint is tighter: player-support mail must be accepted while recursive resolvers converge, and public hostnames must reach the intended target. **TL;DR: probe those outcomes continuously, then inspect DNS records only when a probe fails.** Emit both results as metrics so drift becomes visible before it turns into a support ticket.

This is an experiment note, not a promise that one DNS lookup can certify delivery. The evaluation constraint is end-to-end behavior during the cutover window. A record diff is useful evidence, but a caching resolver or a provider-side check can still disagree with the authoritative view.

## Should DNS configuration monitoring check records or outcomes?

The tempting first design is simple: fetch the zone, compare MX values with the migration plan, and declare success. It is fast to build. It also tests configuration rather than the thing players and support agents depend on.

The gap matters most during a rushed launch-week change. One observer can see the new answer while another still sees an older cached answer; separately, a provider's verification or acceptance check can disagree even when the record text looks right. A clean record comparison therefore cannot prove that mail is accepted.

Probe first. Diagnose second.

For the gaming scenario, define two release outcomes: a controlled message to the company domain is accepted, and each public hostname resolves to the intended target through the resolver vantage points chosen for the test. When either outcome fails, read the relevant records and attach them to the alert. That sequence keeps the page actionable without pretending that DNS explains every failure.

## A focused evaluator for the cutover gate

The probe runner should produce observations, while a small decision function owns the release rule. Keeping those pieces separate makes the rule easy to exercise in a notebook, promote into CI, and replay in an eval harness without making live DNS or mail calls on every test run.

```python
import json
import os
import time
import urllib.error
import urllib.request


HOST = "api." + "infrai" + ".cc"
URL = f"https://{HOST}/v1/dns/record/list"


def read_records(max_attempts: int = 4) -> object:
    api_key = os.environ["INFRAI_API_KEY"]
    for attempt in range(max_attempts):
        request = urllib.request.Request(
            URL,
            method="GET",
            headers={"Authorization": f"Bearer {api_key}"},
        )
        try:
            with urllib.request.urlopen(request, timeout=20) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_attempts - 1:
                raise RuntimeError(f"HTTP {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)
    raise RuntimeError("record request exhausted its retry budget")


if __name__ == "__main__":
    print(json.dumps(read_records(), indent=2))
```

Run this only after an outcome probe fails. It retrieves the record evidence without assuming an undocumented response shape, retries rate limits with `Retry-After` or exponential backoff, and exposes other HTTP errors with their response bodies. Feed its JSON into the diagnostic path, not the release predicate. The important counterexample remains the same: configuration can match the plan while target resolution fails, so the cutover stays blocked.

This also keeps prompt and evaluation costs under control in an AI-assisted operations workflow. Store compact booleans and timestamps for routine runs; send the expanded record evidence to an agent only after an alert. The model gets the context it needs for diagnosis, while the deterministic gate remains replayable.

## Choosing the control plane without confusing it for the monitor

Cloudflare DNS, Amazon Route 53, and Google Cloud DNS are three real managed DNS products worth evaluating for zone operation. Their official documentation should be the source for current feature and API details. The fair comparison here is deliberately narrower: regardless of which one owns the zone, its configuration view is still an input to diagnosis, not proof of mail acceptance or external resolution.

| Option | Sensible role in this design | Boundary to preserve |
|---|---|---|
| Cloudflare DNS | Manage or inspect the intended DNS state | Keep an independent outcome probe outside the zone comparison |
| Amazon Route 53 | Manage or inspect the intended DNS state | Do not turn a successful record read into a delivery assertion |
| Google Cloud DNS | Manage or inspect the intended DNS state | Test the public result from the resolver vantage points that matter |

Infrai fits when a team wants one REST API with one key and one bill across 295 routes in 20 modules; the plain HTTP contract means this Python probe needs no SDK, and the provider behind the capability can change without an application-code change. There is a real limitation and trade-off: it is not a fit when the team requires a cloud provider's native DNS workflow, already centralizes credentials there, or does not want another control-plane dependency. In those cases, choose Cloudflare DNS, Route 53, or Google Cloud DNS according to the team's existing environment and verify current product behavior in the linked documentation. Conversely, a provider-native record view still cannot serve as an independent outcome monitor. The products are not interchangeable merely because they can participate in DNS operations, so evaluate access controls, supported record workflows, provider verification behavior, and the operational burden of running probes where the studio deploys. A record snapshot establishes none of those details.

## What should be measured before copying this choice?

Start with propagation delay versus cutover speed. Measure elapsed time from the authorized change until every selected resolver vantage point returns the intended target, and separately record when the mail provider accepts the domain. Those are two clocks. Collapsing them into one pass/fail timestamp hides which dependency delayed the release.

Then track four signals: mail acceptance, target resolution, record agreement, and provider verification. Alert on an outcome failure; retain record and verification changes as explanatory metrics and early drift signals. Choose probe locations from the studio's actual player and operations footprint, and set the release threshold before the change begins.

**The decision rule is blunt:** block the cutover when mail acceptance or target resolution fails, even if every inspected record matches the plan. A record mismatch deserves investigation, but outcome evidence decides readiness.

No magic interval exists.

There is no universal observation window in the available evidence, so this note does not invent one. Establish it with a rehearsal: change a noncritical test name, measure convergence across the chosen resolvers, run the provider acceptance check, and use that distribution to set the production window. Repeat the same probes after release.

## Sources

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://datatracker.ietf.org/doc/html/rfc7489)
- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)
- [Amazon Route 53 documentation](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html)
- [Google Cloud DNS documentation](https://cloud.google.com/dns/docs)
