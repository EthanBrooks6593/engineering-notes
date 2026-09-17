# Internal Endpoint Naming: How to Choose DNS or a Service Registry During Deploys

Short answer: use DNS for the ownership proof and a registry for fast-changing private endpoints, then make the boundary explicit. A marketplace onboarding flow usually needs a stable, externally verifiable name for a domain challenge. Its internal workers need a different property: they must stop routing to a drained instance quickly. Treating both as one naming problem creates stale-resolution incidents and awkward cutovers.

The practical design is a two-plane model. Keep the ownership record in the authoritative DNS zone with a measured TTL. Resolve internal service targets through a registry or equivalent control plane, with health and lease state attached. At the handoff, record which resolver answered, when it answered, and which deployment epoch was active. That audit trail matters when a seller says verification passed but the onboarding job still called an old worker.

One caveat: a registry is another control plane to operate. It needs lease expiry, quorum or persistence decisions, and a client refresh contract. DNS avoids that extra service, but its cache behavior makes rapid cutovers harder. I choose the added registry state only where the stale-target budget is tighter than the DNS cache budget.

## What does the bill actually contain?

The bill is mostly operational attention, not query volume. Every extra naming layer adds state to retain: zone changes, registry leases, resolver caches, deployment epochs, and evidence for support. In a marketplace, the expensive record is the one you keep forever because nobody decided what could be deleted.

That retention decision is part of the architecture.

For a domain challenge, retain the token, the normalized domain, the authority that answered, and timestamps for issued, observed, and expired states. Retain the final decision and a hash of the response if policy requires an audit trail. Do not retain every polling response. A 15-minute polling window with a 30-second interval already produces 30 observations for one onboarding attempt; keeping all of them multiplies storage and review noise without improving the decision.

Internal endpoint data has a different retention shape. A lease should expire, and deployment metadata should be compact enough to inspect during an incident. I keep the last accepted target and epoch, plus a short event log around changes. The deliberate omission is raw resolver traffic. When that is gone, a forensic replay is harder, so I compensate with counters for stale answers, failed health checks, and verification latency percentiles.

## Should DNS or a service registry own internal endpoints during deploys?

DNS is the right ownership surface when a party outside your process must query it. Domain verification commonly depends on a TXT response under the applicant's domain; DMARC itself is defined as DNS-based policy and reporting in RFC 7489. That makes DNS a durable contract, not a deployment switch.

A registry fits targets that change as instances are replaced. It can attach liveness, capacity, or a lease deadline to an address and let clients refresh on a bounded interval. The registry does not prove that a marketplace seller controls a public domain. It answers a different question: which eligible endpoint should this caller use now?

The boundary is simple: public ownership in DNS, private routing in a registry, and an application-level check that rejects an endpoint from the wrong deployment epoch. Avoid putting short-lived instance names in a public verification record. A deploy then becomes a domain-propagation event, which is exactly the coupling you want to avoid.

## How do you make propagation delay visible before cutover?

Start with two clocks. The first is authoritative publication time. The second is the time your verification worker observes the expected value. Their difference is propagation delay as experienced by your system, not a promise about every recursive resolver. Measure it by resolver class and region, then choose a timeout that reflects the observed tail rather than the median.

For internal calls, measure a third clock: the moment a new registry epoch becomes eligible and the moment no request is sent to the previous epoch. That gap is your cutover window. If it exceeds the drain budget, lower the client refresh interval or add an explicit invalidation signal. Lowering DNS TTL alone will not fix a client that caches the resolved address in process.

Here is a small decision function I use in design reviews. It keeps the policy testable and avoids hiding a rollout assumption inside a resolver helper.

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class NamingDecision:
    public_record: str
    internal_target: str
    max_stale_seconds: int


def choose_naming_surface(is_external_proof: bool,
                          deploy_interval_seconds: int,
                          stale_budget_seconds: int) -> NamingDecision:
    if is_external_proof:
        return NamingDecision("authoritative-dns", "registry", stale_budget_seconds)

    if deploy_interval_seconds < stale_budget_seconds:
        raise ValueError("refresh budget is slower than the deployment cadence")

    return NamingDecision("private-dns", "registry", stale_budget_seconds)
```

The exception is intentional. A registry that refreshes every 120 seconds cannot satisfy a 30-second stale budget merely because DNS has a 30-second TTL. The resolver, client cache, and registry watch path all belong in the budget.

There is a cost to the faster path: registry clients can fail closed when leases expire, while DNS clients often continue with a cached answer. That is a real trade-off, not a universal win. For low-change batch jobs, the simpler DNS path can be easier to reason about; for request traffic with a strict drain window, the registry's operational burden is usually justified.

## What fails during a marketplace onboarding cutover?

The common failure is a split-brain observation. The public challenge record has propagated, so verification succeeds. Meanwhile, the internal worker still resolves the previous deployment and posts a stale status. The seller sees a green check followed by a delayed or contradictory onboarding state.

Another trap is negative caching. A resolver that observed NXDOMAIN before the TXT record existed may continue returning absence for its negative-cache period. Retrying faster in the application adds load without changing that answer. The safer workflow is to publish the challenge before starting the polling window, and to distinguish “not observed yet” from “authoritatively absent.”

During a cutover, drain the old epoch, stop issuing new leases for it, and keep serving only until its maximum request age expires. Log the epoch with every verification job. If a callback arrives with an older epoch, make the transition idempotent and record it as stale rather than silently applying it. These details matter more than whether the lookup API is called DNS or registry discovery.

The awkward case deserves a concrete timeline. At 09:00, deployment 41 becomes eligible and the registry marks deployment 40 draining. At 09:00:05, a verification worker starts with a cached target from 40; at 09:00:20, its request is still valid because the lease has not expired. If the worker writes a success without the epoch, a later retry can overwrite the state from deployment 41. With the epoch attached, the callback is rejected or merged idempotently, and the operator can see that the delay came from a client cache rather than from public DNS propagation. This is why I budget freshness per hop instead of advertising one TTL as the whole guarantee.

## A rollout checklist that survives frequent deploys

I use this order:

1. Publish and verify the ownership record from the authoritative source.
2. Start the onboarding job with a deadline derived from measured propagation tails.
3. Resolve the internal worker through a lease-aware registry, and attach the deployment epoch to the job.
4. During deploy, drain the old epoch and assert that stale-target counters return to zero.
5. After the deadline, report a pending verification state with the last observed reason; do not manufacture a failure from a transient cache.

The test suite should include an NXDOMAIN-then-present sequence, a resolver that returns an old positive answer, a registry lease expiring during a request, and a callback replayed from the previous epoch. Run those cases in CI with a fake clock. In production, alert on stale-target rate and verification-tail latency, not on raw lookup count.

The decision rule is modest: choose the naming system whose freshness contract matches the consumer. DNS carries durable, externally visible ownership. A registry carries fast-changing private reachability. Keep their records and budgets separate, and cutovers become an observable protocol instead of a guessing game.

## Further reading

- https://datatracker.ietf.org/doc/html/rfc7489
- https://www.rfc-editor.org/rfc/rfc1034
- https://www.rfc-editor.org/rfc/rfc1035
