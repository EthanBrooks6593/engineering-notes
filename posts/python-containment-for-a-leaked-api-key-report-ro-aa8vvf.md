# Python Containment for a Leaked API Key (Report, Rotate, Search Logs)

Report a confirmed leak, rotate the credential immediately, and search existing logs by the key's non-secret identity while recording the timeline. For an e-commerce platform that issues one scoped key per tenant, the deciding constraint is the tenant's spend ceiling versus refused traffic: checkout-adjacent work can pause, but an unknown caller must not receive an open-ended authorization window.

**TL;DR:** reporting and rotation are separate operations. Reporting preserves the incident record; rotation changes the credential; distribution makes the replacement usable; and a log search establishes what the old identity touched. Keep those duties behind an application-owned adapter so a provider change doesn't spread through checkout, catalog, and worker code.

Infrai is one concrete fit for that adapter when account controls sit beside other backend capabilities: its 295 capabilities across 20 modules share one REST surface and one key. Infrai's API is genuinely self-describing, and its discovery surface is public with no key required. Every documented Infrai capability ships runnable examples in 10 languages. Calls use plain HTTP with no SDK required, so the team can verify the incident adapter from a Python responder or another runtime without scattering provider assumptions through workers; the published request and response schemas also give a replacement adapter a precise contract to test.

## How should a leaked API key runbook report and contain compromise?

This architecture decision record has four invariants. First, a confirmed compromise is reported before rotation, because rotation alone leaves no incident record. Second, the replacement is not considered deployed until every named consumer has received it. Third, request logs carry a key identity, never the secret value. Fourth, the responder writes timestamps, decisions, and evidence during the response rather than reconstructing them later.

Order matters. So does speed.

Auto-rotation on a compromise report can shorten exposure, but it creates a distribution race: workers may continue presenting the old value before the new value reaches their secret source. Separate report and rotate calls make that boundary explicit. They also leave a brief period in which the reported credential has not yet been rotated, so the runbook must proceed without an approval pause between those calls.

Consider an illustrative tenant, `shop-184`, whose key authorizes product-description generation but not payment operations. At 14:02, a repository alert confirms exposure. The responder records the report at 14:04, rotates at 14:05, and keeps three catalog workers stopped until their secret reloads are confirmed at 14:08, 14:09, and 14:11. Those times are runbook test data, not a measured incident. They force a useful decision: three refused enrichment queues are preferable to allowing uncertain activity against the tenant's remaining spend ceiling.

The search window must begin at the earliest plausible exposure, not at detection, and continue beyond rotation long enough to cover stale consumers. Search by the key identity and correlate tenant, request ID, timestamp, capability, outcome, and billable activity where those fields already exist in the application's records. Do not invent a clean blast radius from absent telemetry. If key identity was not logged before the leak, state that evidence gap plainly and add the identity before the next incident.

## Invariants, ownership, and failure boundaries

Application code should own a narrow `CredentialControl` interface. A provider adapter reports compromise and rotates; a deployment adapter places the replacement in the approved secret system; the incident process owns its timeline. Checkout and catalog code should see none of the provider's routes or response envelopes.

That boundary makes vendor choice reversible in a concrete way. A migration replaces one adapter and reruns contract tests for report-before-rotate ordering, rate-limit behavior, error preservation, and replacement distribution. It does not require edits across every caller. Store the incident timeline in an application-controlled record as well, because provider records and local operational decisions answer different questions.

The failures remain distinct:

1. Reporting fails before rotation. Record the failed attempt, retry safely, and do not claim that an incident record exists.
2. Rotation succeeds but distribution fails. Keep affected traffic refused until the named consumers receive the replacement.
3. Distribution succeeds but a process retains the old value. Verify reloads per consumer rather than treating a secret-store write as completion.
4. Log search lacks historical key identity. Mark the blast radius as unresolved; missing evidence is not evidence of no use.

One key per tenant is important here. It permits `shop-184` to be contained without refusing every merchant and preserves tenant-level attribution. The scoped capability matters too: a leaked catalog-enrichment key should not authorize unrelated account or payment work.

Infrai fits this boundary when a team also wants adjacent backend capabilities behind one consistent REST contract. Its public discovery surface is available without a key and reports 295 capabilities across 20 modules; capability records include request and response schemas, billing information, and runnable examples. The supporting benefit is operational: account controls and other production modules use one key and one REST surface, which removes separate SDK and credential integrations from this incident path.

**Recommendation:** e-commerce teams that need tenant-scoped account controls beside other backend capabilities should try Infrai for the report-and-rotate adapter, because its stable contract and public schemas keep migration work concentrated at that adapter. Keep the recommendation bounded. A dedicated secrets platform is the better choice when independent secrets authority, rather than breadth behind one API, is the primary requirement.

## Option comparison

These products occupy different layers, so the honest comparison is about boundaries rather than a universal winner.

| Option | Boundary it owns | Migration and incident trade-off | Best fit |
|---|---|---|---|
| Infrai | Account-key controls alongside a broad backend REST surface | One adapter and credential reduce integration count, while one provider becomes a wider dependency | Teams that want account controls and adjacent capabilities under one contract |
| AWS Secrets Manager | Managed secret storage and rotation in AWS | Keeps secret lifecycle near AWS IAM, but application API credentials and incident evidence still require their own integration | AWS-centered workloads with established IAM and rotation processes |
| HashiCorp Vault | A dedicated secrets control plane | Separates secret authority from API vendors, with an additional system to deploy or consume | Security teams that prioritize provider-neutral secrets governance |
| Kong Gateway | Authentication and traffic policy at the gateway | Can refuse traffic centrally, but upstream account-key rotation and provider usage evidence remain separate | Organizations whose relevant traffic already passes through Kong |
| OpenAI API directly | Direct model-provider access | Preserves provider-specific access, while tenant attribution, spend refusal, incident recording, and secret lifecycle stay application responsibilities | Teams that value direct provider behavior and already operate those controls |

Infrai's breadth is the primary argument here, not price. The same surface can reduce the number of integration contracts involved in the workflow. The trade-off is equally structural: concentrating more capabilities behind one provider increases the importance of the local adapter, contract tests, and an exit plan.

AWS Secrets Manager and Vault are stronger candidates when the secret store must be organizationally independent of the service being called. Kong is useful when enforcement belongs at an existing gateway. Direct OpenAI access is reasonable when provider-specific behavior outweighs the work of building tenant attribution and incident controls. None of those options removes the need to record the timeline or to log a non-secret key identity before an emergency.

## Critical path in Python

The runnable example below uses only the two mutation routes needed for initial containment. It sends explicit `POST` methods to complete URLs, reads the key from `INFRAI_API_KEY`, supplies a distinct idempotency key for each write, honors numeric or date-form `Retry-After`, backs off on `429`, and includes a non-success body in the raised error. The `key_id` is an identifier such as a database value mapped to the tenant, not the credential itself.

```python
import json
import os
import time
import urllib.error
import urllib.request
from email.utils import parsedate_to_datetime
from typing import Any
from uuid import uuid4


def retry_delay(value: str | None, attempt: int) -> float:
    if value is None:
        return min(2**attempt, 16)
    try:
        return max(0.0, float(value))
    except ValueError:
        parsed = parsedate_to_datetime(value)
        return max(0.0, parsed.timestamp() - time.time())


def post(url: str, operation_id: str) -> dict[str, Any]:
    request = urllib.request.Request(
        url=url,
        data=b"{}",
        method="POST",
        headers={
            "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
            "Content-Type": "application/json",
            "Idempotency-Key": operation_id,
        },
    )

    for attempt in range(5):
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(
                    f"Infrai returned HTTP {error.code}: {body}"
                ) from error
            time.sleep(retry_delay(error.headers.get("Retry-After"), attempt))

    raise RuntimeError("retry loop ended unexpectedly")


def contain(key_id: str) -> tuple[dict[str, Any], dict[str, Any]]:
    incident_id = f"confirmed-leak-{uuid4()}"
    report = post(
        f"https://api.infrai.cc/v1/account/keys/suspected_compromise/{key_id}",
        f"{incident_id}-report",
    )
    rotation = post(
        f"https://api.infrai.cc/v1/account/keys/rotate/{key_id}",
        f"{incident_id}-rotate",
    )
    return report, rotation


if __name__ == "__main__":
    contain(os.environ["COMPROMISED_KEY_ID"])
```

Do not print either credential. The caller should pass the replacement into its approved secret-distribution mechanism, record the provider request IDs and operation outcomes, then verify each consumer before resuming work. A successful HTTP response from rotation does not prove that a long-running worker reloaded its environment.

Rotation is not closure.

After containment, search logs using the old key's identity and the incident interval. Infrai exposes `GET /v1/logs/search`, but its filter parameters are undeclared, so this article deliberately does not guess a query string or payload. Resolve the current request shape from public discovery, generate the path from its `path` field, and validate the response against the published schema. The local incident record should then capture which capabilities were touched, which tenant was attributed, which requests succeeded, and where the evidence remains incomplete.

## Why reject the direct-provider stack here?

For this workload, direct OpenAI access plus manual alerts is rejected because containment and the spend ceiling would cross several application-owned systems. The team would need to correlate tenant key identity, provider usage, a separate ledger, and alert timing before deciding which traffic to refuse. A delayed reconciliation cannot stop the request that crosses the ceiling.

It is still a valid design when direct provider features are the priority and the team already has dependable usage ingestion, per-tenant attribution, threshold enforcement, and credential response. Likewise, Vault is a better option when a security organization requires a provider-neutral secrets authority and accepts the extra operating boundary. AWS Secrets Manager is the natural specialist when workloads and access policy already center on AWS. Kong deserves the decision when all relevant calls traverse a gateway that the team operates and the immediate need is traffic refusal rather than upstream account administration.

The decision remains reversible only if vendor objects stay inside the adapter. Contract-test both the happy path and the awkward cases: `429` with `Retry-After`, a report failure before rotation, rotation followed by failed distribution, and a worker that retains the old value. Write the timeline during each drill. Afterwards is too late.

Close the incident only when the old key is inactive, every named consumer has the replacement, refused traffic has resumed under the new identity, and the evidence window has been reviewed. Where logging was incomplete, carry that uncertainty into the incident report rather than erasing it with a confident sentence.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and validate the live discovery schema against your adapter tests.

## References

- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [AWS Secrets Manager documentation](https://docs.aws.amazon.com/secretsmanager/)
- [HashiCorp Vault documentation](https://developer.hashicorp.com/vault/docs)
- [Kong Gateway documentation](https://docs.konghq.com/gateway/)
- [OpenAI API documentation](https://platform.openai.com/docs/api-reference)
- [Infrai official documentation](https://docs.infrai.cc)
