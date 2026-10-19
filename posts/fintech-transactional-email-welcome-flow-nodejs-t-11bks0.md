# Fintech Transactional Email Welcome Flow — Nodejs Template Preview and Delivery Evidence

Send each generated report through an asynchronous, idempotent email pipeline, and treat preview, provider acceptance, and recipient delivery as separate states with separate evidence. Delivery reliability is the deciding constraint: a successful API response proves submission, not inbox arrival, while a financial report creates stricter requirements around recipient correctness, attachment integrity, and auditability.

**Short answer:** freeze the rendered message before enqueueing it, assign one internal message ID, submit from a worker, ingest authenticated delivery events, and reconcile unsettled messages by polling only when events are late or unavailable. Never attach the report directly inside the signup request path.

This architecture decision record covers a welcome flow in which a newly registered fintech user receives a generated account report as an email attachment. The same record drives the human preview, the send attempt, and every later status change. That makes retries boring, investigations possible, and accidental duplicate reports much less likely.

## How should a Nodejs transactional email welcome flow freeze its template?

The first invariant is recipient binding. The account ID, normalized destination address, report ID, and authorization decision belong in one immutable send intent. If a user changes an address after signup, that change must not silently redirect a report that was already approved for another destination. Create a new intent after repeating the authorization check.

The second invariant is content identity. Store a cryptographic digest for the exact attachment bytes and a version for the subject, text body, HTML body, and template inputs. The preview must render that frozen version. Re-rendering a mutable template in the worker can produce a message different from the one an operator reviewed, especially during a deployment.

The third invariant is idempotency under retries. A queue may redeliver work, a worker may lose its response after the remote system accepted a message, and an event may arrive more than once. One internal message ID therefore needs one durable submission record and monotonic state transitions.

Duplicate input is normal.

Design for it.

There is also a security boundary. Email is a transport channel, not proof that the intended person opened the attachment. NIST SP 800-63B discusses protected channels and authenticator handling; a generated financial report should follow the application's data-classification policy rather than being treated like harmless welcome copy. If policy prohibits sensitive attachments, email a short-lived authenticated retrieval link instead. Do not put an OTP, password, or account secret into the report or message log.

SPF helps receiving systems evaluate whether a host is authorized to use a domain in the SMTP envelope, but RFC 7208 explicitly does not authenticate message content. Domain authorization is useful evidence, not a guarantee of delivery or message integrity. DKIM signs selected message content, and DMARC adds domain-alignment policy and reporting; deploy these controls together according to the sending domain's policy.

Acceptance is not arrival.

## Build the evidence ledger before choosing the transport

Use a transactional outbox beside the signup transaction. The signup operation commits the account and a `report_email_requested` record atomically. A separate report job produces immutable bytes, then an email worker performs the network call. This prevents a slow report renderer or mail endpoint from holding open the user-facing request, and it closes the gap where signup commits but the request process dies before enqueueing the welcome message.

The state machine should be small: `requested`, `rendered`, `submitted`, `delivered`, `failed`, and `unknown`. Keep transport-specific labels in event metadata rather than leaking them into business logic. `Submitted` means that the transport accepted responsibility. `Delivered` means the receiving mail system reported acceptance. Neither means the user saw the message. A bounce after submission moves the record to `failed`; silence after the reconciliation deadline moves it to `unknown`, not `delivered`.

## Where should each failure be owned?

| Option | Signup latency | Duplicate control | Delivery evidence | Appropriate use |
|---|---:|---|---|---|
| Send inside signup | Network-bound | Fragile after timeouts | Usually one response | Low-impact messages where loss is acceptable |
| Queue after commit | Low | Depends on enqueue reliability | Good if events are retained | Systems tolerating a commit-to-enqueue gap |
| Transactional outbox | Low | Strong with a unique intent key | Durable with reconciliation | Financial reports and other auditable workflows |

The attachment boundary deserves its own limits. Enforce an application maximum before MIME encoding, verify the generated media type, use a safe filename, and record the byte count and SHA-256 digest. RFC 2045 defines Base64 output in four encoded characters for each group of three input bytes, with padding for the final partial group, so a report that fits the application's raw-byte limit may still exceed a downstream message-size limit once headers and MIME boundaries are included. Check the fully assembled message before submission. Keep the limit configurable rather than baking a guessed transport ceiling into domain logic, and return a permanent generation failure when the report is too large; retrying identical bytes cannot change their size. The explicit trade-off is a little more storage and validation work in exchange for knowing exactly which artifact was previewed and submitted.

Bytes settle the argument.

Failures split into three zones. Before `rendered`, report generation can retry without contacting email transport. Between `rendered` and `submitted`, submission can retry only through the idempotency guard. After `submitted`, transport events or reconciliation decide the outcome; regenerating and resending is a separate, authorized operation.

Those boundaries matter.

## Encode create, preview, send, and polling as state transitions

The domain interface below is transport-neutral. Its adapter may use SMTP or an HTTP email service, but the workflow does not depend on one event vocabulary. Repository methods need transactions and unique constraints in the real implementation; the names make those boundaries explicit.

```python
from dataclasses import dataclass
from hashlib import sha256
from typing import Protocol


@dataclass(frozen=True)
class FrozenMessage:
    message_id: str
    recipient: str
    subject: str
    text_body: str
    html_body: str
    attachment_name: str
    attachment_bytes: bytes
    attachment_sha256: str


class MailTransport(Protocol):
    def submit(self, message: FrozenMessage, idempotency_key: str) -> str:
        """Return the transport receipt ID after acceptance."""

    def status(self, receipt_id: str) -> str:
        """Return a transport status for reconciliation."""


def freeze_report_email(intent, report_bytes: bytes, renderer) -> FrozenMessage:
    rendered = renderer.render(
        template_version=intent.template_version,
        data={"display_name": intent.display_name},
    )
    return FrozenMessage(
        message_id=intent.message_id,
        recipient=intent.approved_recipient,
        subject=rendered.subject,
        text_body=rendered.text_body,
        html_body=rendered.html_body,
        attachment_name=f"account-report-{intent.report_id}.pdf",
        attachment_bytes=report_bytes,
        attachment_sha256=sha256(report_bytes).hexdigest(),
    )


def process_message(message_id, repository, transport, renderer):
    intent = repository.lock_intent(message_id)
    if intent.state in {"submitted", "delivered"}:
        return intent.state

    report_bytes = repository.load_authorized_report(intent.report_id)
    message = freeze_report_email(intent, report_bytes, renderer)
    repository.save_rendered(message)
    receipt_id = transport.submit(message, idempotency_key=message.message_id)
    repository.mark_submitted(message.message_id, receipt_id)
    return "submitted"


def reconcile(message_id, repository, transport):
    record = repository.get(message_id)
    if record.state != "submitted":
        return record.state
    observed = transport.status(record.receipt_id)
    return repository.apply_observation(
        message_id=message_id, observed=observed, source="poll"
    )
```

The preview endpoint should read `save_rendered` output, escape any surrounding diagnostic UI, and require the same staff or account authorization as the report itself. It must not invoke `submit`. A preview token should be scoped to one message and expire; logging the full preview body would create another copy of financial data.

Event ingestion and polling both call the same `apply_observation` function. Validate event authenticity before accepting a transition, retain the raw event identifier for deduplication, and reject backward transitions such as `delivered` to `submitted`. Poll with bounded exponential backoff and jitter. Stop after a defined reconciliation window, because indefinite polling hides uncertainty and consumes capacity without producing stronger evidence.

One sharp edge is the crash after remote acceptance but before `mark_submitted`. An idempotency key can let a capable transport return the earlier result. If the chosen transport cannot provide idempotent submission or lookup by that key, the system cannot prove whether retrying will duplicate the email. Record that limitation and route ambiguous cases to reconciliation rather than blindly resending.

Do not guess.

## Retention, access, and deletion finish the audit trail

Track each transition with the internal message ID, report digest, attempt number, timestamp, transport receipt ID, and evidence source. Exclude attachment bytes, rendered bodies, secrets, and unnecessary personal data from logs. Retention periods should follow the organization's legal and security policy, including deletion of preview artifacts and attachments after their operational purpose ends.

Useful service-level indicators separate stages: time from request to render, render to submission, submission to terminal observation, and the share left `unknown`. Aggregate bounce categories and complaints without turning a person's mailbox history into a broad analytics dataset. Alert on changes in unknown outcomes, rejected submissions, event-ingestion age, and outbox backlog. A single end-to-end success percentage conceals where the pipeline is failing.

Stage timing is actionable.

Test the awkward paths. Kill a worker immediately after submission, deliver the same event twice, deliver events out of order, change a template between preview and processing, generate a corrupt PDF, and change the user's address while work is queued. Also test Unicode display names and filenames, plain-text rendering, HTML escaping, and a report just above the configured size boundary.

For deployment, expand before contracting: add new states and fields first, deploy code that can read both record shapes, migrate outstanding work, then remove obsolete fields. Workers from two releases may overlap. State transitions must remain compatible during that interval.

Mixed versions happen.

A staging inbox can verify MIME structure and links, but it does not establish production deliverability. Production checks should include DNS authentication, bounce processing, complaint handling where applicable, suppression behavior, and monitoring across representative recipient domains. Keep test recipients controlled; sending synthetic financial-looking attachments to arbitrary addresses is a compliance problem, not a load test.

## Browser polling stays outside the audit boundary

I would reject synchronous generation and sending during signup, followed by repeated status polling from the browser. It couples account creation to two external workloads, encourages the client to treat a transient timeout as a failed signup, and exposes transport timing to a user flow that cannot resolve delivery ambiguity. Browser polling also disappears when the tab closes, so it can't serve as durable operational evidence.

It still has a valid use case: an internal prototype with disposable messages, no sensitive attachment, no delivery audit requirement, and explicit tolerance for loss. Even there, return a message ID and show a noncommittal state such as `processing`; do not claim delivery from an accepted submission.

For the fintech report flow, the durable outbox is the better boundary because it preserves intent across process failure and keeps user latency independent of report generation. **The decision rule is concise:** if losing, duplicating, or misdirecting the attachment would require an incident review, freeze the artifact, persist the intent, and reconcile transport evidence asynchronously.

## References

- RFC 7208, Sender Policy Framework (SPF): https://datatracker.ietf.org/doc/html/rfc7208
- RFC 6376, DomainKeys Identified Mail (DKIM) Signatures: https://datatracker.ietf.org/doc/html/rfc6376
- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance (DMARC): https://datatracker.ietf.org/doc/html/rfc7489
- RFC 2045, Multipurpose Internet Mail Extensions (MIME) Part One: https://datatracker.ietf.org/doc/html/rfc2045
- NIST SP 800-63B, Digital Identity Guidelines: https://pages.nist.gov/800-63-3/sp800-63b.html
- OWASP Logging Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html
