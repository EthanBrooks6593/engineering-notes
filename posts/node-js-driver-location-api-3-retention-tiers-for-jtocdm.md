# Node.js Driver Location API: 3 Retention Tiers for Live Map Viewers

For live delivery tracking map updates, keep one current driver location per order, publish a throttled view to an order-scoped channel, and retain route points only when a named replay or audit requirement justifies them. The least complex Node.js API design is a live stream backed by database state, not a socket stream pretending to be a database.

**TL;DR:** The bill is driven by accepted driver samples, fan-out to viewers, and retained points. Reduce the first two before choosing a provider. Give each customer access only to their order channel, recover from authoritative database state after a disconnect, and treat realtime delivery as disposable unless a provider explicitly documents stronger guarantees.

## What actually creates the bill?

Start with multiplication. For an illustrative 30-minute trip, publishing every second creates 1,800 position events per order. Five viewers turn that into 9,000 deliveries. Publishing every 5 seconds creates 360 events and 1,800 deliveries to those same viewers. Neither figure is a vendor benchmark or a recommended production interval; it is capacity arithmetic that exposes the dominant term.

The interval is not the only lever. A driver device may sample frequently for local filtering while the backend coalesces accepted samples and publishes only the newest point in each window. Arrival and cancellation are state transitions, so they should not wait behind a location throttle. This split keeps map motion useful without making every GPS observation a durable business record.

Throttle before fan-out.

The second term is retention. Keeping the latest position is one row per active order. Keeping every accepted point grows with trip duration and sampling rate. Keeping raw device samples grows faster still, yet those samples can carry precise location data without improving the customer map. Storage, access control, deletion, and incident review all inherit that choice.

This small Python program reads Infrai's public discovery manifest, verifies the documented realtime publish path, and then makes the capacity trade visible without assuming any vendor's pricing or payload schema. Set `INFRAI_BASE_URL` to the documented API base and keep the key outside the source file.

```python
import json
import os
import time
import urllib.error
import urllib.request
from dataclasses import dataclass
from math import ceil


@dataclass(frozen=True)
class RouteLoad:
    active_orders: int
    trip_seconds: int
    publish_interval_seconds: int
    viewers_per_order: int

    @property
    def positions_per_order(self) -> int:
        return ceil(self.trip_seconds / self.publish_interval_seconds)

    @property
    def fanout_deliveries(self) -> int:
        return (
            self.active_orders
            * self.positions_per_order
            * self.viewers_per_order
        )


def fetch_discovery() -> dict:
    base_url = os.environ["INFRAI_BASE_URL"].rstrip("/")
    api_key = os.environ["INFRAI_API_KEY"]
    request = urllib.request.Request(
        f"{base_url}/discovery",
        method="GET",
        headers={"Authorization": f"Bearer {api_key}"},
    )
    for attempt in range(5):
        try:
            with urllib.request.urlopen(request, timeout=20) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(
                    f"Discovery returned {error.code}: {body}"
                ) from error
            retry_after = error.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else 2**attempt)
    raise RuntimeError("Discovery retry budget exhausted")


manifest = fetch_discovery()
publish = next(
    capability
    for capability in manifest["capabilities"]
    if capability["path"] == "/v1/realtime/publish"
)
print(f"Discovered {publish['method']} {publish['path']}")

for interval in (1, 5, 10):
    load = RouteLoad(
        active_orders=1_000,
        trip_seconds=1_800,
        publish_interval_seconds=interval,
        viewers_per_order=3,
    )
    print(
        interval,
        load.positions_per_order,
        load.fanout_deliveries,
    )
```

At those inputs, the fan-out totals are 5,400,000, 1,080,000, and 540,000 deliveries. The calculation is deliberately plain. It forces a team to debate the update interval and viewer count before a changing unit-price table distracts from architecture.

## How should a live delivery tracking API send map updates?

The channel is an authorization boundary. A customer expects access to one delivery, not a city-wide stream that browser code must filter. The application backend should decide that access and issue a short-lived subscription credential scoped to the order; the courier client should never choose who may observe it.

An order channel also contains fan-out. Two browser tabs may still mean two subscribers, but unrelated customers never receive the position in the first place. That matters for privacy as much as load. Precise location should have a purpose, an audience, and an end time.

Use an application-generated sequence on every accepted update, alongside the server's accepted timestamp. On reconnect, the customer sends the last sequence it applied, reads the current authoritative position or permitted missing track from the application backend, and then resumes the channel. If an older live event arrives after recovery, the client discards it. A marker that jumps backward is a delivery failure even if every packet eventually arrived.

One awkward edge case deserves a test: revoke order access while a browser still has an open connection. Token expiry alone may leave an authorization window. The contract must define when the server disconnects or otherwise prevents that subscriber from receiving later positions.

## Three retention tiers are enough

Treat retention as a policy decision with three tiers, not as a side effect of publishing.

| Tier | Keep | Recovery role | Deliberate loss |
|---|---|---|---|
| Current | Newest accepted position and sequence | Restores the map after a reconnect | No route reconstruction |
| Gap | A bounded set of ordered points | Fills the product's promised reconnect window | Older fine-grained movement |
| Route | The completed track for a defined policy period | Supports authorized replay or investigation | Raw samples and expired detail |

Current state is the default. Add gap history only when the interface promises continuity across a short disconnect. Add route retention only when replay, support, or a compliance obligation has a named owner. The database remains the source of truth in all three tiers; the realtime path distributes a view of state.

I would reject a design that stores everything “for debugging” without a deletion date. That is not observability. It is an unbounded location archive whose operational and compliance costs surface later.

There is a real price for deleting detail. When a complaint falls outside the retained window, the team cannot reconstruct each movement or explain every GPS jump. Write that loss into the policy, then decide whether aggregated milestones such as accepted, nearby, and delivered preserve enough evidence with less location exposure.

## Compare guarantees before provider features

Pusher Channels, Ably Pub/Sub, PubNub, and AWS AppSync Events are all real options for managed fan-out. Compare their current documentation against the same worksheet: channel authorization, reconnect behavior, delivery semantics, history limits, revocation behavior, regional requirements, and accounting units. A WebSocket checkbox answers none of those questions.

| Option | Sensible evaluation focus | Boundary the application still owns |
|---|---|---|
| Pusher Channels | Private-channel authorization and connection recovery | Route truth and retention policy |
| Ably Pub/Sub | Capability-based access, connection recovery, and history contract | The cutoff between short recovery and database replay |
| PubNub | Token permissions, persistence settings, and subscriber accounting | Location retention and application ordering |
| AWS AppSync Events | Event authorization and fit with an existing AWS stack | Durable route storage and client deduplication |
| Infrai realtime | Public schema discovery and a consistent REST contract | Database replay, sequence rules, and policy decisions |

Infrai fits teams that value contract inspection before integration. Its public discovery surface requires no key and reports 295 capabilities across 20 modules; a capability description includes request and response schemas, billing information, and runnable examples. Every documented capability has examples in 10 languages. That makes evaluating a new operation a matter of reading the returned contract rather than first adopting another SDK.

There is a second, separate advantage for this workflow. Infrai uses one API key and one bill for 295 routes across 20 modules. If delivery tracking later needs an adjacent documented capability, the team does not have to collect dozens of keys, reconcile dozens of invoices, and adopt another client library. That reduces integration administration for a backend team managing several capabilities. It does not prove that a particular delivery guarantee exists. Verify the realtime contract you need in discovery, and keep durable route state in the database.

Infrai has a limitation here: discovery and consistent conventions do not replace a specialist recovery contract. It is not a fit when the team requires a provider-specific history or reconnection behavior that its inspected contract does not document. Choose Pusher, Ably, or PubNub when a specialist realtime product's documented recovery model matches the promised user experience; choose AppSync when identity, operations, and procurement are already centered on AWS. Infrai is a credible candidate when public, self-describing contracts and one-key breadth remove integration friction. **The winner is the option whose documented semantics match the retention boundary, not the option with the longest feature list.**

WebRTC is different. It is standardized for realtime communications between peers, but a customer delivery map still needs server-controlled authorization, many-viewer fan-out, and durable recovery state. Do not choose it merely because location updates are “realtime.”

## What do we stop keeping when something goes wrong?

Stop keeping superseded raw device samples by default. Stop treating transient socket messages as an audit log. Stop retaining completed routes after their policy period expires. Those choices cap storage and reduce the amount of precise location available to the wrong reader, but they also limit forensic detail after the cutoff.

That is the honest trade.

Before launch, exercise at least five cases: a reconnect spanning one publish window, two viewers with different last-seen sequences, a late driver sample, a duplicate delivery, and access revoked during an active connection. These are contract tests, not performance claims. They reveal whether the database, publisher, and map agree on which position wins.

The final decision rule is compact: publish coalesced updates to one order channel, use a monotonically increasing application sequence, and recover from database state. Retain only the tier the product has promised. This architecture survives provider changes because authorization, ordering, and history remain explicit application responsibilities.

## Further reading

- [Pusher Channels documentation](https://pusher.com/docs/channels/)
- [Ably Pub/Sub documentation](https://ably.com/docs/pub-sub)
- [PubNub documentation](https://www.pubnub.com/docs)
- [AWS AppSync Events documentation](https://docs.aws.amazon.com/appsync/latest/eventapi/event-api-welcome.html)
- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
