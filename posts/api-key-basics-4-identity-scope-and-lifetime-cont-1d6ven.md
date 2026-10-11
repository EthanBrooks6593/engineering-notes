# API Key Basics: 4 Identity, Scope, and Lifetime Controls for Prepaid Automation

A balance watcher has an awkward constraint: it must be able to spend when nobody is present, yet its failure must not turn a low-balance alert into unlimited purchasing. The right credential is therefore a dedicated machine identity with the narrowest useful reach, a bounded lifetime, and an independent spend ceiling. Those are separate controls. Treating an API key as a single magic string hides the decision that matters.

**TL;DR:** Bind each key to one workload, authorize only the read and funding actions that workload needs, expire and rotate it on a rehearsed schedule, and enforce a monetary cap outside the key itself. For a prepaid developer-tools account, favor refused traffic once the approved ceiling is reached. Availability matters, but silent overspend is a worse failure mode.

## How should API key identity, scope, and lifetime work together?

An API key is both a secret presented during authentication and a handle that a service can associate with a caller. The useful identity is not "the backend." It is the specific workload and environment: for example, `balance-watcher` in production, distinct from its staging instance and from a human operator. That granularity gives audit records a subject and lets responders revoke one automation path without disabling unrelated jobs.

Do not embed the raw key in source code, a container image, or a workflow file. OWASP recommends centralized secrets management, least-privilege access control, auditing, rotation, and avoiding secrets in logs. Inject the secret at runtime through the deployment environment's secret facility, and ensure exception paths redact it. A debug log that prints request headers can defeat every careful policy around the credential.

Identity also answers ownership questions before an incident. Which team operates this watcher? Which repository deploys it? Which environment may retrieve its secret? Which runbook defines revocation? Store that metadata beside the secret record, while keeping the actual secret out of tickets and chat.

One identity per workload may feel fussy when there are only two cron jobs. It pays for itself when one job behaves badly. Shared credentials collapse attribution and force an all-or-nothing response.

Keep the boundary visible.

## Why can't scope solve the spending problem by itself?

Scope is the set of operations and resources the service will accept for the authenticated identity. A useful balance watcher might need to read one account's balance and request a constrained top-up. It does not need user administration, key creation, billing-profile edits, or access to every account in the organization. Resource boundaries matter as much as action names.

Yet an authorization scope such as `fund:write` usually says what may happen, not how much money may be committed across retries. **Scope limits capability; a spend ceiling limits consequence.** Enforce both. Put the approved amount per funding action, the cumulative window limit, and any account-level maximum in the transaction path rather than trusting the scheduler to behave.

Consider a policy example, not a universal recommendation: the watcher reads the balance every 5 minutes, asks for a $100 refill below a $200 threshold, and may commit no more than $300 in 24 hours. Those values belong in reviewed configuration. The funding service should also accept an idempotency key tied to the intended refill, so a timeout and retry do not represent two independent business intentions. The API key authenticates the caller; it does not make repeated calls equivalent.

The failure chain is worth spelling out because each control catches a different mistake. The watcher reads a low balance, submits a refill, and loses the response. Authentication has succeeded, and the requested operation is inside its scope. If it retries with a fresh business identifier, the server may see a second valid funding intention; a narrow key has not helped. Reusing the identifier lets the transaction layer recognize the retry, while the rolling ceiling limits damage if the caller still generates several distinct intentions. Now suppose the key leaks from an unredacted exception. Workload-specific identity makes the affected caller attributable, restricted scope keeps the secret away from user and credential administration, expiry bounds its normal acceptance period, and revocation provides the emergency stop. None of those controls proves that a refill is affordable. The spend policy does that job. This layered reading is less tidy than assigning one security label to a key, but it gives operators a precise answer when traffic is refused: the identity was wrong, an operation was outside scope, the credential was stale, a duplicate was suppressed, or the approved monetary boundary was reached.

| Identity | Allowed reach | Explicitly refused | Independent limit |
|---|---|---|---|
| Production balance watcher | Read its account balance; request a top-up | Manage users, credentials, or billing profile | Per-action and rolling spend ceiling |
| Staging balance watcher | Read and modify only a test account | Any production account | Non-production funding cap |
| Human responder | Inspect audit events; suspend automation | Routine unattended funding | Approval policy for exceptional action |

The denial path deserves design attention. When the ceiling is exhausted, return a stable refusal, record the reason without the secret, alert the owning team, and leave the account unchanged. Do not keep retrying a policy denial as if it were a transient network error. For this system, refused traffic is the deliberate side of the trade-off.

## Lifetime is a recovery budget

A credential lifetime is the interval during which the credential may be accepted, subject to earlier revocation. Shorter lifetimes reduce how long a copied secret can remain useful, but they also increase rotation frequency and the chance that poor deployment coordination causes an outage. There is no responsible universal duration. Choose it from the exposure of the scope, the reliability of rotation, and how quickly revocation can propagate.

Rotation is not "generate a new string and hope." A workable sequence creates a replacement, deploys it, verifies that the workload is using it, revokes the old credential, and confirms that old-key use now fails. During a controlled overlap, both credentials may need to work. Keep that overlap short and observable; otherwise it quietly becomes permanent.

Here is a generic validator for the controls an internal credential broker could apply before releasing a secret reference. It intentionally does not fetch or print the secret:

```python
from dataclasses import dataclass
from datetime import datetime, timezone


@dataclass(frozen=True)
class CredentialGrant:
    workload: str
    environment: str
    scopes: frozenset[str]
    expires_at: datetime


REQUIRED = frozenset({"balance:read", "funding:request"})
FORBIDDEN = frozenset({"users:write", "credentials:write"})


def validate_grant(grant: CredentialGrant, expected_workload: str) -> None:
    now = datetime.now(timezone.utc)
    if grant.workload != expected_workload:
        raise ValueError("credential belongs to a different workload")
    if grant.environment != "production":
        raise ValueError("production job received a non-production grant")
    if grant.expires_at <= now:
        raise ValueError("credential has expired")
    if not REQUIRED.issubset(grant.scopes):
        raise ValueError("required scope is missing")
    if grant.scopes & FORBIDDEN:
        raise ValueError("grant includes a prohibited scope")
```

This check is intentionally incomplete. The resource identifier, issuer, and permitted audience also need validation when the credential format and authorization system expose them. The monetary controls still belong in the funding transaction. One layer cannot substitute for the others.

Revocation is the emergency lifetime. Test it. If an operator can mark a key revoked but accepting systems continue honoring it for an unknown period, the stated response time is fiction. The same test should verify that audit records show the credential identifier, workload, requested operation, decision, and time without recording the secret value.

## Roll out the boundary before tightening it

Start by inventorying which workload holds each key, where the key can be retrieved, what it actually calls, and who can revoke it. Missing ownership is a deployment blocker. Then issue the dedicated production identity and observe its authorization decisions while the old path remains available. This observation period should capture denied operations and retry behavior, not secret material.

Next, enforce resource and action scope. Add the transaction-level spend ceiling before enabling unattended funding, because a perfectly scoped funding key can still repeat an expensive allowed action. Exercise four cases in a non-production account: normal refill, duplicate request, expired credential, and exhausted ceiling. The expected last result is refusal plus an alert.

Finally, rehearse replacement and revocation, then remove the shared or long-lived predecessor. Track failed authentication, denied authorization, ceiling refusals, duplicate suppression, credential age, and use of a retired key. These signals separate delivery trouble from policy doing its job.

**The decision rule is strict:** automation may preserve service only inside the amount and time window the organization approved in advance. Past that boundary, fail closed, page a human, and preserve the audit trail.

## Sources

- https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
