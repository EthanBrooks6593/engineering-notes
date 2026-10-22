# Postgres Transactional Email Setup — Custom Domain DKIM, SPF, and DMARC Evidence

TL;DR: A beginner SaaS transactional email setup, including welcome email and marketplace receipt flows, should use a custom domain with SPF, DKIM, and DMARC. For a marketplace sending an order receipt after payment settles, keep the durable payment-to-message decision and a small set of delivery outcomes in Postgres; keep neither rendered message bodies nor an endless firehose of provider events. The least complex defensible design is an outbox row plus an append-only evidence record with explicit retention dates.

The storage bill is driven mainly by event volume and retention, not DNS authentication. If 200,000 receipts a month produce four stored delivery events apiece, the system creates 800,000 event rows monthly and 9.6 million yearly. At an illustrative 1 KB per row before indexes, that is 9.6 GB of raw event data, while the 2.4 million annual receipt decisions need far less space. Indexes, replicas, backups, and JSON overhead multiply the first number. This is the term worth changing.

My default is narrow: retain the business decision, authentication configuration version, recipient-domain hash, message identifier, timestamps, and terminal disposition. Collapse noisy intermediate events after the troubleshooting window. This gives compliance and support teams evidence that a receipt was commissioned and processed without turning an operational log into a shadow mailbox.

Keep less.

## What are you actually proving?

A receipt pipeline has three separate questions. Did the marketplace decide to send after payment reached the required state? Did the message leave through the authenticated domain configuration in force at that time? What did the receiving system report afterward? A single `sent=true` flag cannot answer all three.

The payment record remains the authority for the order. The outbox connects that state change to an idempotent send request. Evidence then records the attempt independently of mutable order fields. This boundary matters when a buyer changes an address later or a worker retries after a timeout: historical proof should not silently change with the customer profile. Authentication supplies domain-level evidence, not proof that a human read a message. SPF authorizes hosts to use a domain in the SMTP envelope identity. DKIM adds a cryptographic signature whose domain can align with the visible From domain. DMARC evaluates aligned SPF or DKIM and publishes a policy plus aggregate-reporting instructions. Passing DMARC still does not establish inbox placement, delivery to a particular person, or a read. In other words, the database can preserve an accountable chain of actions, but it cannot upgrade a mailbox provider's report into evidence of human attention.

Open tracking is especially weak evidence. Apple Mail Privacy Protection downloads remote content in the background, so a pixel load cannot safely carry the meaning "the buyer opened this receipt." Treat opens as a coarse product signal, if they are collected at all, never as the compliance record.

## The dominant retention cost is duplicate delivery telemetry

Suppose one logical receipt generates `queued`, `accepted`, `delivered`, and a later duplicate `delivered` callback. Retaining every callback body forever preserves transport noise, often including headers or addresses that have no lasting evidentiary value. The useful long-lived state is smaller: which event won, when it occurred, and enough identifiers to reconcile it with the original request.

I use two windows because the data has two jobs. A short operational window keeps normalized event detail for debugging retries, bounces, and complaints. A longer evidence window keeps only the receipt decision and terminal outcome. The exact durations are policy decisions: legal, security, privacy, finance, and support owners should approve them for the marketplace's jurisdictions and dispute cycle. Engineering should encode those decisions as data, not bury them in a cleanup job.

| Record | Short operational copy | Longer-lived evidence | Reason |
| --- | --- | --- | --- |
| Rendered HTML and text | Only while needed to send or diagnose | No | It duplicates order and customer data |
| Raw webhook payload | Time-boxed, access-controlled | No | It is verbose and may contain personal data |
| Send decision | Yes | Yes | It links payment state to the communication |
| Authentication revision | Yes | Yes | It identifies the domain policy used |
| Terminal outcome | Yes | Yes | It supports reconciliation without every transition |
| Open event | Optional | No | It is not reliable proof of reading |

This design changes the dominant term from `messages × callbacks × long retention` to `messages × compact terminal records`, while detailed callbacks age out on the shorter schedule. Keep the arithmetic visible in capacity reviews. A retention label with no row-count estimate is wishful thinking.

## Model the evidence boundary in Postgres

The application should insert the receipt outbox record in the same database transaction that records the qualifying payment transition. A worker can claim it later and send idempotently. Do not hold a payment transaction open across a network call.

The focused Python example below builds a compact evidence object after a terminal event. It deliberately stores a keyed recipient digest rather than an address, rejects unknown outcomes, and calculates deletion from the policy selected for that record. The secret should come from a managed secret store and be rotated under a documented process; the example reads it from the environment only to keep the boundary obvious.

```python
from dataclasses import dataclass
from datetime import UTC, datetime, timedelta
import hashlib
import hmac
import os
from uuid import UUID


TERMINAL_OUTCOMES = {"delivered", "bounced", "complained"}


@dataclass(frozen=True)
class ReceiptEvidence:
    order_id: UUID
    message_id: str
    recipient_digest: str
    auth_revision: str
    payment_settled_at: datetime
    terminal_outcome: str
    outcome_at: datetime
    delete_after: datetime


def digest_recipient(address: str, key: bytes) -> str:
    normalized = address.strip().lower().encode("utf-8")
    return hmac.new(key, normalized, hashlib.sha256).hexdigest()


def build_evidence(
    *,
    order_id: UUID,
    message_id: str,
    recipient: str,
    auth_revision: str,
    payment_settled_at: datetime,
    terminal_outcome: str,
    outcome_at: datetime,
    retention_days: int,
) -> ReceiptEvidence:
    if terminal_outcome not in TERMINAL_OUTCOMES:
        raise ValueError("A terminal outcome is required")
    if payment_settled_at.tzinfo is None or outcome_at.tzinfo is None:
        raise ValueError("Timestamps must be timezone-aware")

    digest_key = os.environ["RECEIPT_DIGEST_KEY"].encode("utf-8")
    return ReceiptEvidence(
        order_id=order_id,
        message_id=message_id,
        recipient_digest=digest_recipient(recipient, digest_key),
        auth_revision=auth_revision,
        payment_settled_at=payment_settled_at.astimezone(UTC),
        terminal_outcome=terminal_outcome,
        outcome_at=outcome_at.astimezone(UTC),
        delete_after=outcome_at.astimezone(UTC) + timedelta(days=retention_days),
    )
```

The digest is for correlation, not anonymity. If the input space is guessable or the key is exposed, an address can be tested against it. Restrict access, rotate keys deliberately, and delete the digest when its purpose expires. Also make `message_id` unique at the database boundary so retries converge on one logical receipt rather than creating several plausible histories.

Store the authentication revision as an internal immutable identifier that resolves to the approved configuration: From domain, envelope domain, DKIM selector and public-key fingerprint, and DMARC policy revision. Do not copy private signing keys into evidence. DNS changes over time; recording only "DKIM passed" loses which configuration was expected when the worker sent the message.

## How should a beginner SaaS set up a custom domain for transactional email?

The same authentication foundation serves a welcome email and a post-settlement receipt, even though their triggers and evidence differ. Use a dedicated subdomain for transactional mail so its operational ownership and reputation boundary are clear. Publish a single SPF record for each SPF identity; RFC 7208 specifies that multiple SPF records produce a permanent error. Keep the authorized sender set small, because SPF evaluation has a limit of 10 terms that cause DNS queries. A sprawling include chain can fail even though every individual entry looked reasonable during review.

DKIM needs an operational rotation plan before the first receipt. Generate the new selector, publish and verify its public key, begin signing, then retain the old public key long enough for messages already in transit to validate. Removing the old selector at the moment of deployment creates an avoidable verification gap.

Begin DMARC with aggregate reporting and a monitoring policy. Inspect alignment for legitimate streams, including support tools and any separate return-path domain, before moving toward enforcement. RFC 7489 defines `p=none`, `quarantine`, and `reject`; it also makes clear that receivers apply their own local policy. Enforcement is not a magic deliverability switch.

For US and EU recipients, resist the temptation to make geography choose the authentication path. SPF, DKIM, and DMARC should be consistent. Geography belongs in the data-purpose, access, and retention decisions, reviewed by counsel against the actual service and role of each party. A transactional receipt should also stay transactional in content; mixing promotional copy into it complicates both user expectations and compliance classification. These practices support deliverability, but no DNS setup guarantees inbox placement.

Deploy in stages: validate DNS from outside the authoritative system, send to controlled accounts at several mailbox domains, inspect complete headers, then increase traffic while watching bounces and complaints. Alarm on authentication failure rates and queue age, but use absolute counts beside percentages at low volume. One failure out of two messages is a frightening 50% and a weak trend.

Small samples lie.

## Test deletion as seriously as delivery

The unhappy paths define whether the evidence is credible. Test duplicate callbacks, callbacks received before the worker commits its send result, a timeout after remote acceptance, a payment reversal racing the outbox worker, an expired DKIM selector, and a receipt address corrected after settlement. Replay the same event twice and verify that the evidence row does not fork.

Deletion needs its own acceptance test. Select expired rows in bounded batches, record the policy and aggregate count in a deletion-run audit, delete the detailed records, and verify that backups follow the approved expiration process. Avoid putting recipient addresses or raw payloads in deletion logs; otherwise cleanup merely moves the data.

There is a real trade-off. Once raw callbacks and rendered bodies expire, an old dispute may be harder to reconstruct, and a subtle template defect may no longer be reproducible byte for byte. I would accept that loss only after the owners of dispute handling define the minimum evidence they need. Keeping everything feels safer until a privacy request, access review, or breach turns every forgotten payload into scope.

The final rule is plain: retain the compact proof for an approved purpose, retain detailed telemetry briefly for operations, and stop keeping message bodies and intermediate events when those purposes end. Authentication improves the trustworthiness of the sending identity. Disciplined retention keeps that trust from becoming an unlimited data liability.

## Further reading

- RFC 7208, Sender Policy Framework (SPF): https://datatracker.ietf.org/doc/html/rfc7208
- RFC 6376, DomainKeys Identified Mail (DKIM): https://datatracker.ietf.org/doc/html/rfc6376
- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance (DMARC): https://datatracker.ietf.org/doc/html/rfc7489
- Apple, Mail Privacy Protection guide: https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios
- European Commission, data protection principles: https://commission.europa.eu/law/law-topic/data-protection/rules-business-and-organisations/principles-gdpr_en
- US Federal Trade Commission, CAN-SPAM Act compliance guide: https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business
