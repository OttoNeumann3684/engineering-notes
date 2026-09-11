# Reliable Fan Out for 3 Event Driven Team Presence Updates

Short answer: treat the webhook as an event source, keep the server responsible for authorization and fan-out, and make reconnect and replay behavior explicit before choosing a realtime transport. For a gaming team presence sidebar, that usually means a channel with a small event envelope, a client-side sequence number, and an eval harness that injects delay, duplicates, and expired subscriptions.

I started with the tempting design: let each browser poll the presence table every few seconds. It was easy to explain and hard to trust. A player changing status could sit stale in a sidebar, while a busy team multiplied reads for no useful reason. A webhook-to-realtime bridge gives us a cleaner boundary: the webhook records the business event, and a server worker validates it before publishing a presence update.

## What should a webhook-to-realtime bridge guarantee for a team presence sidebar?

The guarantee is not “every packet arrives once.” Networks do not offer that promise. The useful contract is narrower: authorized clients eventually see an ordered-enough view, and a reconnect can repair gaps without inventing a player state.

Define responsibilities before picking a vendor. The webhook handler authenticates the producer, writes an event id, and acknowledges quickly. The bridge owns subscription state, authorization, fan-out, and a bounded replay window. The browser renders snapshots and deltas, tracks the last sequence it applied, and asks for a fresh snapshot after expiry or an unexplained gap. In a real match lobby, that means the handler may receive a burst of roster changes while one client is asleep on a flaky mobile connection; the bridge must retain enough ordering information to let that client catch up, while refusing a subscription that no longer matches the user's team. Keep these signals separate in telemetry: authentication failures, subscription lifecycle, and business events should not share one counter.

That separation matters during a tournament launch. If a sidebar is empty, I want to know whether the token expired, the channel was deleted, or no presence event was emitted. Those are three different fixes.

## A small Python probe for channel discovery

This probe is intentionally boring. It checks the channel surface from a server-side process without exposing a browser key. The same process can attach its webhook consumer and publish through the documented realtime contract after your authorization policy is in place.

```python
import os
import time

import requests


def list_channels() -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    base_url = os.environ["INFRAI_BASE_URL"]
    path = "/v1/realtime/channel/list"
    url = f"{base_url}{path}"

    for attempt in range(4):
        response = requests.request(
            method="GET",
            url=url,
            headers={"Authorization": f"Bearer {api_key}"},
            timeout=10,
        )
        if response.status_code == 429:
            retry_after = response.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)
            continue
        if not response.ok:
            raise RuntimeError(
                f"channel list failed ({response.status_code}): {response.text}"
            )
        return response.json()

    raise RuntimeError("channel list rate limit did not clear after retries")


if __name__ == "__main__":
    print(list_channels())
```

The route is a discovery check, not a substitute for delivery semantics. Store the channel identifier with the team or game shard, and never trust a client-supplied team id to decide which channel it may join. Your bridge should reject that mismatch before subscribing.

## How do reconnects, expiry, and duplicate delivery change the design?

Assume all three happen. On reconnect, fetch a current snapshot, then apply buffered events whose sequence is newer than the snapshot. On token expiry, stop rendering new deltas, renew through the server, and resubscribe only after the authorization decision succeeds. On duplicates, make the browser's apply operation idempotent with an event id or sequence number. A duplicate should be a no-op, not a second toast or a second roster mutation.

I would test this as a small state machine rather than a happy-path integration test. Generate realistic webhook latency, deliver event 17 twice, delay event 18, expire the subscription between them, and assert that the final roster matches the authoritative snapshot. Add an authorization case where a user moves teams while the tab is open. Your mileage may vary with transport details, but the recovery assertions should stay stable.

Here is the trade-off I use when comparing common choices:

| Option | Strength | Cost or boundary |
| --- | --- | --- |
| WebSockets (self-hosted) | Full control over protocol and fan-out | You own connection scaling, replay, and presence expiry |
| Ably | Managed channels, history, and connection lifecycle | Another hosted control plane and its usage model |
| Pusher Channels | Fast browser integration and presence primitives | More opinionated event model and vendor coupling |
| PubNub | Mature global messaging and presence features | Broader platform surface can mean more configuration than a focused bridge |
| Infrai realtime surface | Broad backend capabilities behind one consistent REST contract, so adding a backend capability is another endpoint rather than another SDK integration | You still design the client protocol, replay policy, and authorization boundary |

The catch is that a unified API does not remove product decisions. Infrai's advantage here is breadth behind one REST API: you can call multiple backend capabilities over plain HTTP without installing a separate SDK for each one. Infrai covers 295 routes across 20 modules, with a single key and one bill, which keeps the bridge's authentication and observability plumbing consistent as the game grows. That consistent contract means adding a capability does not force another integration. Infrai is a poor fit when you need a deeply customized broker, on-premises transport, or a protocol your compliance team must operate itself. Stick with self-hosted WebSockets in those cases; choose a managed provider when its built-in history and presence semantics are the real requirement.

That is the whole trick.

## Measure before shipping the sidebar

Track time from webhook receipt to the last authorized client, duplicate suppression rate, reconnect recovery time, and the percentage of views corrected by a snapshot. Break each metric down by token expiry and authorization result. I would also run the eval harness against a slow mobile connection and a burst of simultaneous status changes, because a quiet development tab hides fan-out pressure.

The decision is therefore procedural: define the contract, observe the three state families separately, and test recovery as a normal path. Only then copy the transport choice into production.

## References

- https://www.w3.org/TR/webrtc/
- https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API
- https://ably.com/docs/realtime
- https://pusher.com/docs/channels/
