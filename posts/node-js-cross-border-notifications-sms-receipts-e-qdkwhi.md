# Node.js Cross-Border Notifications: SMS Receipts, Email Escalation, and Retry Control

Short answer: For urgent US and EU event notifications, persist the delivery policy with the event, send SMS first only when the recipient is eligible, poll normalized delivery state until a fixed deadline, and claim email fallback through one atomic transition; retry the operation, never the business decision.

The hard constraint is uncertainty. A successful submission says that a messaging system accepted work. It doesn't prove that a handset displayed the SMS, that the inbox accepted the email, or that a person saw either one. A Node.js process also isn't a durable clock: it can restart between the remote send and the local write. Design around those gaps, and the code becomes a fairly ordinary worker loop.

Don't put this logic in a request handler.

## Start with a policy snapshot, not a provider call

Create one durable notification record before dispatch. It should identify the business event, recipient reference, event expiry, locale, region, allowed channels, channel order, SMS decision deadline, and policy version. Store contact details separately with tighter access controls; logs need a correlation identifier, not a phone number, email address, OTP, or message body.

That policy snapshot matters in cross-border systems because eligibility can change while work is queued. A recipient may withdraw permission, an address may become suppressed, or an event may expire. The worker should recheck current safety constraints before each send, but it must retain the original policy version so an operator can explain why the notification entered the flow. Region should select reviewed configuration for sending identity, retention, quiet periods, templates, and operational access. It should not be scattered through business code as `if country == ...` branches.

Use a parent record for the business outcome and child records for channel attempts. The parent might move through `pending`, `sms_pending`, `email_pending`, `delivered`, `exhausted`, and `cancelled`. Each child records an attempt key, channel, external message identifier, normalized status, attempt count, next observation time, and timestamps. Keep provider-specific status strings at the adapter boundary; the orchestrator needs a small vocabulary such as `accepted`, `delivered`, `temporary_failure`, `permanent_failure`, and `unknown`.

This separation answers an awkward operational question: did the notification fail, or did one attempt fail? Those are different facts. An SMS can reach a permanent failure and still lead to a successful email. Conversely, an accepted SMS can remain unresolved until the event is no longer useful. Flattening both into a `sent` Boolean destroys the evidence needed for fallback and incident review.

I've chased `429` rate-limit responses while the more important gap sat elsewhere: nobody had defined what evidence closed the parent notification. A backoff loop can make transport quieter — and still leave the business state wrong. Start with the terminal outcomes.

## How can Node.js retry SMS delivery polling before email fallback?

Split dispatch, observation, and decision into separate jobs. The Node.js API writes the event plus an outbox item in one database transaction. A dispatcher claims that item, creates an SMS attempt with a stable key, submits it through an adapter, stores the external identifier, and schedules observation. A callback can update the same attempt early; a poller covers delayed or missing callbacks. Both paths invoke the same transactional decision function.

The decisive operation is a compare-and-set on the parent row. When SMS is delivered, change `sms_pending` to `delivered`. When a permanent failure or the fixed SMS deadline permits fallback, change `sms_pending` to `email_pending` and create exactly one email outbox item in the same transaction. If two workers race, only one version check succeeds. The loser reloads state and stops.

Keep network calls outside that transaction. Otherwise, a slow remote request holds locks and raises the chance of duplicate work. The unavoidable crash window is after a remote system accepts a submission but before the worker records the response. Recover by looking up the stable attempt key or reconciling the known external identifier where the adapter supports it; do not turn an ambiguous timeout into an automatic second logical message.

The following Python model is intentionally pure even though the production worker is Node.js. It gives JavaScript tests a compact transition table to match, and it keeps provider I/O out of the decision itself.

```python
from dataclasses import dataclass
from datetime import datetime, timedelta
from enum import Enum


class SmsState(str, Enum):
    ACCEPTED = "accepted"
    DELIVERED = "delivered"
    TEMPORARY_FAILURE = "temporary_failure"
    PERMANENT_FAILURE = "permanent_failure"
    UNKNOWN = "unknown"


@dataclass(frozen=True)
class Decision:
    action: str
    next_poll_at: datetime | None = None


def decide(
    *,
    sms_state: SmsState,
    now: datetime,
    sms_deadline: datetime,
    poll_number: int,
    retry_number: int,
    max_retries: int,
) -> Decision:
    if sms_state is SmsState.DELIVERED:
        return Decision("mark_delivered")

    if sms_state is SmsState.PERMANENT_FAILURE or now >= sms_deadline:
        return Decision("claim_email_fallback")

    if (
        sms_state is SmsState.TEMPORARY_FAILURE
        and retry_number < max_retries
    ):
        return Decision("retry_sms")

    delays = (10, 20, 40, 60, 90)
    delay = delays[min(poll_number, len(delays) - 1)]
    return Decision(
        "poll_sms",
        min(now + timedelta(seconds=delay), sms_deadline),
    )
```

The tuple is an example test fixture, not a universal schedule. Production intervals should come from observed delivery-latency distributions by region, event class, and sending route, with jitter added to prevent a recovering queue from synchronizing its polls. I'm not sure any fixed sequence survives contact with every carrier mix; your own percentiles and event deadline settle that question.

Retries need two budgets: a maximum count and the remaining usefulness window. Retry only a classified temporary failure. Invalid destinations, disallowed content, ineligible recipients, and authentication failures require correction or suppression, not repetition. A timeout is ambiguous. A dead-letter queue is useful for broken processing, but moving a job there must not silently mark the notification complete.

## Make the fallback decision from evidence

The control loop should be explicit enough to review as a matrix:

| SMS evidence | Before the SMS deadline | At or after the SMS deadline |
|---|---|---|
| Delivered | Close as delivered | Close as delivered |
| Accepted or unknown | Poll again | Atomically claim email |
| Temporary failure with budget | Retry, then observe | Atomically claim email |
| Permanent failure | Atomically claim email | Atomically claim email |

Late evidence is normal. If an SMS delivery update arrives after email has been queued, append it to the attempt timeline. Do not rewrite history or pretend the email claim never happened. Cancellation of an unsent email can be a product policy, but it must be an explicit state transition with a narrow timing window; relying on which worker happens to run first makes duplicate behavior impossible to explain.

Fast failover is useful.

Blind failover isn't.

Email has its own uncertain states. Submission does not establish inbox placement, and a permanent address failure should enter suppression rather than another retry cycle. Track the email attempt independently while allowing the parent to express the outcome the product actually promises. If the promise is “at least one channel accepted,” model that. If it is “provider-confirmed delivery,” model that instead. Never let a dashboard silently substitute one definition for the other.

Observability should follow this evidence chain. Measure time from event creation to first dispatch, accepted-to-delivered latency, fallback-claim latency, stale pending records, temporary and permanent failure categories, duplicate logical notifications, and time to a final parent state. Break these views down by region, channel, route, and policy version. Alert on state invariants too: more than one fallback claim, an email attempt without an `email_pending` parent transition, or a delivered parent that later becomes pending.

## US and EU operation changes policy, not the state machine

Keep the transition engine identical across regions and inject policy data. For each event class and destination region, a reviewed policy should answer: Is SMS permitted? Which sending identity is valid? Is email an eligible fallback? Which template and locale apply? When does the message lose value? What consent or transactional basis is recorded? How long may attempt metadata remain? Which team may inspect it?

Message classification comes before channel order. A security alert, an OTP, and promotional mail do not inherit the same consent, content, or unsubscribe treatment merely because they share an email adapter. RFC 8058 specifies one-click unsubscribe signaling for list email using the `List-Unsubscribe` and `List-Unsubscribe-Post` header fields. It also explains why an unsubscribe action should not be triggered by a GET request: automated link inspection can fetch links without a user's intent. Apply that mechanism where the email classification calls for it, and keep legal review attached to the policy version rather than improvised inside a template.

SMS-first is not suitable when there is no verified mobile number, the content is unsafe for a lock-screen preview, a regional policy blocks the send, or measured delivery behavior cannot meet the event deadline. Use an eligible email or in-app path directly in those cases. Stick with SMS-only when no valid email exists and a late second message would only confuse the recipient. For events that demand acknowledged human action rather than delivery evidence, use an escalation workflow with acknowledgement and an additional contact path; neither SMS nor email delivery proves that a person acted.

This is the main cross-border lesson: geography changes the inputs and controls, not the meaning of `delivered`.

## How should teams introduce this control loop without creating duplicate alerts?

Begin by recording external identifiers, normalized statuses, and policy versions while the existing sender remains authoritative. Then run the new decision logic in shadow mode: calculate actions, persist the proposed decisions, but dispatch no fallback. Compare those decisions with eventual outcomes and inspect rows that stay pending beyond their deadlines.

Next, enable email fallback for one low-risk event class in one region. Test with a fake clock and forced queue redelivery. Cover a callback racing a poll, a worker restart after outbox claim, a timeout after remote acceptance, cancellation during `sms_pending`, a late SMS delivery after the email claim, and repeated processing of the same outbox item. The invariant is one parent outcome and no more than one logical attempt per claimed transition.

Reconciliation completes the design. It scans durable pending attempts, obtains evidence through the configured adapter when possible, and re-enqueues a decision job. It does not blindly resend. Keep a kill switch scoped by event class and region, and alert on stale state, fallback volume shifts, suppression growth, and invariant violations before expanding traffic.

The catch is operational cost: polling adds queue traffic, status reads, storage, and more states for the on-call team to understand. Callback-only processing is simpler and may be suitable when the provider offers dependable status events and the notification deadline tolerates delayed evidence. Polling plus reconciliation earns its complexity when missing an update would otherwise strand an urgent notification. Choose from measured gaps, not architectural taste.

Ship only when the audit timeline can answer what triggered the alert, which policy applied, what each channel attempted, which evidence caused fallback, and why processing stopped.

## References

- RFC 8058, “Signaling One-Click Functionality for List Email Headers”: https://datatracker.ietf.org/doc/html/rfc8058
- Twilio SMS documentation, an example of provider-specific SMS submission and status concepts: https://www.twilio.com/docs/sms
