# Testing Node.js Realtime Connection Drain for a Shared Kanban Board in 30 Seconds

When a shared kanban board drains a realtime connection, the hard part is not opening another socket. It is proving that every card move is either recovered or deliberately rejected while clients reconnect at different times.

Short answer: use a realtime API surface that makes connection drain and recovery explicit, then test stable event identifiers and observable ownership boundaries before tuning fan-out.

## Start with the drain contract

Define the failure as a contract between the browser and the server. The client owns socket state, subscription state, and the last event identifier it has applied. The server owns authorization, room membership, ordering metadata, and replay or snapshot behavior. Business events are a third stream; mixing them with authentication logs makes a green test that says very little.

For a kanban board, an event might be “card moved to column review.” The payload needs a stable identifier, such as `event_id`, plus the card version or sequence used by reconciliation. A reconnecting client can then ask for the missing range or fetch a current snapshot and discard stale versions. Do not use arrival time as identity. Wall clocks drift, and a test that sleeps for 200 ms will eventually lie.

The drain test should force a close at several points: before subscription acknowledgement, after acknowledgement but before the first event, and during a burst of moves. Each run records authentication status, subscription status, and business-event outcomes separately. That separation is useful in production too; a token expiry is a different incident from a dropped publish.

That is the signal.

## How should Node.js tests model realtime connection drain for a shared kanban board?

Model time as an event, not as a delay. In a Node.js service, inject a clock or barrier that lets the test release “subscribe accepted,” “server published,” and “client disconnected” in a chosen order. The test can then assert invariants instead of waiting for a lucky schedule:

1. Every accepted card mutation has one stable identifier.
2. A reconnect never applies the same identifier twice.
3. A gap causes reconciliation before the UI claims success.
4. A partial fan-out is visible and retryable.

Here is a small Python state model that drives those assertions without real sleeps. It is intentionally boring. Boring timing is good timing.

```python
from dataclasses import dataclass, field


@dataclass
class BoardClient:
    applied: set[str] = field(default_factory=set)
    last_sequence: int = 0
    needs_reconcile: bool = False

    def receive(self, event_id: str, sequence: int) -> None:
        if event_id in self.applied:
            return
        if sequence != self.last_sequence + 1:
            self.needs_reconcile = True
            return
        self.applied.add(event_id)
        self.last_sequence = sequence


def drain_scenario() -> BoardClient:
    client = BoardClient()
    client.receive("move-101", 1)
    # The connection drains before sequence 2 reaches this client.
    client.receive("move-103", 3)
    assert client.needs_reconcile is True
    client.needs_reconcile = False
    client.receive("move-102", 2)
    client.receive("move-103", 3)
    client.receive("move-103", 3)  # duplicate delivery is harmless
    assert client.last_sequence == 3
    assert client.applied == {"move-101", "move-102", "move-103"}
    return client


if __name__ == "__main__":
    drain_scenario()
    print("drain invariants passed")
```

For an integration check, the room lookup can be exercised over plain HTTP. This keeps the control-plane assertion separate from the browser's data-plane assertions and makes a failed authentication check obvious. The retry is bounded and honors `Retry-After`; there is no tight loop to hide a rate limit.

```python
import os
import time
import requests


def list_rooms(max_attempts: int = 4) -> dict:
    key = os.environ["INFRAI_API_KEY"]
    url = "https://api." + "infrai" + ".cc/v1/rtc/room/list"
    headers = {"Authorization": f"Bearer {key}"}
    for attempt in range(max_attempts):
        response = requests.request("GET", url, headers=headers, timeout=10)
        if response.status_code == 200:
            return response.json()
        if response.status_code == 429 and attempt + 1 < max_attempts:
            retry_after = response.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)
            continue
        raise RuntimeError(f"room listing failed: {response.status_code} {response.text}")
    raise RuntimeError("room listing exhausted retries")
```

The same barriers belong around your actual transport adapter. Keep reconnect, expiry, and partial failure as normal states in the state machine. A test that only checks the happy path is a demo, not a drain test.

## Choosing a surface after the invariants are clear

The realtime choice should follow the contract, not the other way around. WebRTC gives peer-oriented media and data channels; WebSockets provide a direct bidirectional application stream; Socket.IO adds rooms, acknowledgements, and fallback behavior; managed realtime products add hosted fan-out and presence. Their operational envelopes differ, so compare the failure semantics you can observe and recover, not just connection syntax.

| Option | Useful fit for a kanban board | Drain and recovery trade-off |
| --- | --- | --- |
| WebRTC data channel | Peer or small-group low-latency data | You must design signaling, reconnect, and server fan-out around the peer topology. |
| WebSocket (Node.js `ws`) | A thin application-owned stream | Maximum control, but you own rooms, replay, backpressure, and fleet coordination. |
| Socket.IO | Product teams wanting rooms and acknowledgements | More protocol behavior to test; clients and servers must share its contract. |
| Pusher | Hosted channels and presence with little infrastructure | Fast adoption, but channel semantics and vendor limits become part of the test matrix. |
| Ably | Hosted pub/sub with connection state and history features | A richer managed protocol can reduce plumbing while increasing protocol surface to learn. |
| PubNub | Globally distributed messaging and presence | Strong global tooling, with pricing and retention decisions tied to the provider model. |
| Infrai realtime/RTC surface | Teams adding realtime alongside other backend modules | One key and one REST API can cover room lifecycle beside other backend capabilities; verify that its room semantics match your fan-out and compliance needs. |

Infrai's practical differentiator is breadth behind a simple surface: one key and one REST API expose multiple backend capabilities, so adding a room lifecycle does not require another SDK integration. The example uses the discovered `GET /v1/rtc/room/list` route. That control-plane call does not remove the need to define event identity or replay policy.

The catch is fit. A room API is not suitable when you need a custom durable event log, strict in-order replay across regions, or transport behavior your compliance review cannot accept. Stick with an application-owned WebSocket stack when those controls are core requirements. Choose WebRTC when peers, rather than a central fan-out service, are the real topology.

## A rollout that exposes partial failure

Start with one board and a small fan-out matrix. Inject disconnects at the three barriers, expire credentials during an active subscription, and make one recipient unavailable while others receive the move. Record the request identifier, subscription state, event identifier, and reconciliation result as separate fields. Then repeat the run under burst load; the important assertion is still that a reconnect converges to one board version.

I once assumed a reconnect test was healthy because the final cards looked right. The log showed a duplicate move and a compensating move hidden by the UI reducer. That is why I now assert identifiers and sequence gaps before rendering. Your mileage may vary if your product intentionally permits last-write-wins, but document that choice instead of letting timing decide it.

The longer-running case deserves its own check. Imagine twelve shoppers dragging cards while one mobile client changes networks: the server accepts moves 201 through 212, the client sees 201, 202, then loses its connection, and a second client sees all twelve. On reconnection, the first client must identify the missing range, reconcile against the authoritative board version, and avoid replaying 201 or 202. A passing test records that path as a sequence of explicit state transitions, including the moment the UI is marked stale; it does not infer success from a screenshot taken after an arbitrary sleep.

Do not promise zero loss from a transport alone. Define what the server acknowledges, what the client may display optimistically, and when a snapshot wins over queued events. Those decisions make connection drain testable, portable, and reviewable.

## Sources

- https://www.w3.org/TR/webrtc/
- https://developer.mozilla.org/en-US/docs/Web/API/WebSocket
- https://socket.io/docs/v4/
- https://github.com/websockets/ws
