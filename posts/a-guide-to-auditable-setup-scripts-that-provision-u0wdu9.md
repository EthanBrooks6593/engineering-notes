# A Guide to Auditable Setup Scripts That Provision Scoped API Keys

Provision a CI credential only when the setup job can bind one narrowly defined principal, one permission set, one repository, and one expiration policy into an immutable audit record. Then move the secret directly into the pipeline's secret store and verify identity by reading it back through the consumer path, without printing the value.

Short answer: creation is not proof. A useful B2B SaaS access review needs evidence that the credential the pipeline can retrieve belongs to the expected machine principal and has the approved scope. If the setup script cannot produce that evidence without exposing the secret, stop the rollout.

This is the architecture decision I would record for an account platform whose reviewers need to sign off on machine access. The hard part is not generating a random string. It is preserving identity and custody across two systems that may succeed, fail, or retry independently.

## How should a setup script provision and verify a scoped API key?

Four invariants define the design.

1. The bootstrap identity may create a credential for the CI principal, but it must not become the long-lived runtime identity.
2. The new credential must carry only the permissions required by the pipeline and a finite expiration accepted by policy.
3. Plaintext may exist in process memory during the handoff, but it must not enter arguments, logs, artifacts, exception text, or the audit record.
4. Verification must use the value retrieved from the secret store, not the convenient in-memory copy returned by creation.

The fourth invariant catches a surprisingly easy mistake. A script can create key A, fail while storing it, and still call an identity endpoint with key A. That proves the issuing service works. It says nothing about what tomorrow's build will retrieve. Read-after-write verification crosses the same custody boundary as the consumer, so it detects stale secret versions, wrong names, wrong environments, and permission mistakes. Consider a setup job targeting `billing-prod/send-events`: the issuer returns the intended principal, but a typo writes the key under the staging name while the production name still points to an older credential. Verifying the returned key passes. Fetching the named production version and checking its stable principal ID exposes the mismatch before a build can send anything or create misleading review evidence.

Creation is the easy half.

Keep the evidence separate from the secret. A review record can safely contain the principal identifier, repository identifier, normalized scopes, expiration timestamp, issuer request identifier, secret locator, stored version identifier, verification timestamp, and verifier result. It must not contain the credential, a reversible encoding of it, or a broadly reusable hash presented as authentication material. OWASP's secrets guidance also treats auditing, rotation, expiration, and least privilege as lifecycle concerns rather than one-time setup details.

There are three failure boundaries: issue, store, and verify. If issue fails, there is nothing to clean up. If storage fails after issue succeeds, revoke the newly issued credential. If verification fails after storage succeeds, disable or revoke the credential and mark the stored version unusable according to the store's supported lifecycle. **Never report success from a partially completed handoff.**

Retries need an operation identifier generated before issuance. Pass it as an idempotency token if the issuer supports one; otherwise query the issuer's authoritative metadata before creating another credential. Blind retries can leave several valid keys behind, and an access reviewer cannot infer which one the runner possesses. This is a real design trade-off: automatic retry improves recovery from transient transport failures, while retrying a non-idempotent issuance call can expand access without producing a reliable custody trail.

## Compare the handoff choices

The primary decision axis is auditability, not setup convenience.

| Handoff | Secret exposure surface | Review evidence | Failure behavior | Appropriate use |
| --- | --- | --- | --- | --- |
| Direct API write from setup job | Process memory and encrypted request | Issuer request ID plus store version ID | Script can compensate after a failed write or verification | Default when both systems expose auditable APIs |
| Human copy and paste | Terminal, clipboard, browser, and operator session | Often split across human and system logs | Ambiguous; retries and transcription can create drift | Emergency procedure with explicit approval and immediate rotation |
| File artifact passed to another job | Workspace, artifact service, and downstream job | Artifact metadata adds events but also custody | Cleanup may race replication or retention | Isolated build systems where no direct secret-store API exists |
| Dynamic short-lived credential | No stored long-lived key when federation is available | Subject claims and issuance events | Re-authentication replaces rotation | Workloads with an issuer and runner that share a trust protocol |

Dynamic credentials reduce secret custody, but they do not remove the access review. The reviewer still needs to see which workload subject can assume which role, under which conditions, and for how long. Static API keys remain reasonable where federation is unavailable, provided the compensating controls are explicit. The direct-write approach has a clear limitation: it is a poor fit when the CI runner and secret store cannot establish an authenticated network path, or when the issuer cannot expose stable identity metadata. In the first case, use a tightly controlled encrypted transfer. In the second, fix the identity contract or require manual review rather than claiming that a successful authentication proves ownership.

One distinction matters: a secret-store write receipt proves persistence, while an identity response proves usability and ownership. Keep both. Neither substitutes for the other.

Two receipts. Two claims.

## Implement the critical path

The following Python keeps provider-specific details behind narrow interfaces. `Issuer.create_key` is assumed to return the secret once plus non-secret metadata. `SecretStore.put` returns an opaque version identifier. `IdentityClient.who_am_i` authenticates with the retrieved value and returns a stable machine-principal identifier.

```python
from __future__ import annotations

from dataclasses import dataclass
from datetime import datetime, timezone
from typing import Protocol, Sequence
from uuid import uuid4


@dataclass(frozen=True)
class IssuedKey:
    secret: str
    key_id: str
    principal_id: str
    scopes: tuple[str, ...]
    expires_at: datetime
    request_id: str


@dataclass(frozen=True)
class Identity:
    principal_id: str
    scopes: tuple[str, ...]


class Issuer(Protocol):
    def create_key(
        self,
        *,
        principal_id: str,
        scopes: Sequence[str],
        expires_at: datetime,
        operation_id: str,
    ) -> IssuedKey: ...

    def revoke_key(self, *, key_id: str) -> None: ...


class SecretStore(Protocol):
    def put(self, *, name: str, value: str) -> str: ...
    def get(self, *, name: str, version_id: str) -> str: ...
    def disable(self, *, name: str, version_id: str) -> None: ...


class IdentityClient(Protocol):
    def who_am_i(self, *, credential: str) -> Identity: ...


def provision_ci_key(
    issuer: Issuer,
    store: SecretStore,
    identity_client: IdentityClient,
    *,
    principal_id: str,
    repository_id: str,
    secret_name: str,
    scopes: Sequence[str],
    expires_at: datetime,
) -> dict[str, object]:
    if expires_at <= datetime.now(timezone.utc):
        raise ValueError("expiration must be in the future")

    approved_scopes = tuple(sorted(set(scopes)))
    if not approved_scopes:
        raise ValueError("at least one approved scope is required")

    operation_id = str(uuid4())
    issued = issuer.create_key(
        principal_id=principal_id,
        scopes=approved_scopes,
        expires_at=expires_at,
        operation_id=operation_id,
    )

    version_id: str | None = None
    try:
        version_id = store.put(name=secret_name, value=issued.secret)
        stored_secret = store.get(name=secret_name, version_id=version_id)
        observed = identity_client.who_am_i(credential=stored_secret)

        expected_scopes = set(approved_scopes)
        if observed.principal_id != principal_id:
            raise RuntimeError("stored credential resolved to the wrong principal")
        if set(observed.scopes) != expected_scopes:
            raise RuntimeError("stored credential has unexpected scopes")

        return {
            "operation_id": operation_id,
            "repository_id": repository_id,
            "principal_id": principal_id,
            "key_id": issued.key_id,
            "scopes": list(approved_scopes),
            "expires_at": issued.expires_at.isoformat(),
            "issuer_request_id": issued.request_id,
            "secret_name": secret_name,
            "secret_version_id": version_id,
            "verified_at": datetime.now(timezone.utc).isoformat(),
            "verification": "passed",
        }
    except Exception:
        if version_id is not None:
            store.disable(name=secret_name, version_id=version_id)
        issuer.revoke_key(key_id=issued.key_id)
        raise
```

The returned dictionary is audit evidence, so it intentionally excludes `issued.secret` and `stored_secret`. Those local references should also be short-lived. Do not dump local variables in an exception handler, and configure HTTP clients to redact authorization headers before enabling debug logging.

Exact scope comparison in the example is deliberate. A subset check would accept an unexpected extra permission; a superset check could accept a missing one. If an identity service returns effective permissions rather than assigned scopes, model that distinction explicitly and compare against the approved effective set. Names are weak evidence. Stable identifiers are better.

The script also needs bounded timeouts and classified errors in its concrete adapters. Retry transport failures only where the operation is idempotent, treat authentication and authorization failures as terminal, and preserve non-secret request IDs for investigation. A pipeline should emit one final status tied to `operation_id`, not a transcript containing response bodies.

## Make the review signable

An access review becomes practical when evidence answers a small set of concrete questions: What workload owns this credential? What can it do? Where can it be retrieved? When does it expire? Did the stored value authenticate as the intended principal? Who approved the mapping?

Produce a compact record for each active machine credential and join it with repository ownership data. Reviewers should see stable IDs alongside friendly labels because labels can be renamed. They should also see the last successful verification event, but a recent verification is not proof of recent use. Usage belongs in a separate, clearly labeled signal.

**Fail closed on mismatched identity or scope.** Do not quietly rewrite the evidence to match what the service returned. The approved intent and observed state must remain separate fields so drift is visible.

For an email, SMS, or OTP pipeline, this separation is especially useful. A credential allowed to send transactional messages should not silently inherit template administration, contact export, or account-management permissions. Delivery pressure does not justify a broader key. Rate limits and delivery gaps are operational concerns; permission expansion is an access decision.

Test the orchestration with fakes that force failure at each boundary: issuance rejection, store timeout before acknowledgement, store timeout after persistence, stale read, wrong principal, extra scope, and revocation failure. The awkward case is an indeterminate write. Record it as unresolved, quarantine the target secret name from further automated changes, and reconcile against authoritative store metadata before retrying. Guessing creates duplicate live credentials.

Deployment should use a separately controlled bootstrap identity, protected branch rules for changes to approved scopes, and a scheduled reconciliation that compares issuer inventory, secret-store versions, and review records. Alert on orphaned issuer keys, active secret versions with no approved record, expired credentials that still authenticate, and review records whose principals no longer map to an owned repository.

## Why reject a generated environment file?

Writing the key to an environment file and uploading it as a CI artifact looks portable. It is also a poor default: the artifact becomes another secret store, with its own readers, retention rules, replicas, download logs, and cleanup behavior. The setup job has expanded the custody chain before the first build runs.

The option has a valid use case in an isolated or air-gapped build environment where the target runner cannot call the designated secret store. In that case, encrypt the payload for a runner-held recipient key, set a short retention period, restrict artifact readers, record the transfer identifiers, and rotate the credential after import. Treat this as an explicit exception in the architecture record.

For the normal connected pipeline, direct write plus read-back identity verification is easier to explain and audit. Its cost is adapter complexity and a bootstrap identity trusted by both control planes. The acceptance rule is crisp: the setup succeeds only when stored-version evidence and observed identity match the approved principal, scopes, repository, and expiration policy. That is a record a reviewer can sign.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- https://csrc.nist.gov/pubs/sp/800/53/r5/upd1/final
- https://datatracker.ietf.org/doc/html/rfc6749#section-4.4
- https://slsa.dev/spec/v1.2/provenance
