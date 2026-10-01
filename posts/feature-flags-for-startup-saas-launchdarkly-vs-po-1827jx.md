# Feature Flags for Startup SaaS: LaunchDarkly vs PostHog on Cost Attribution

Short answer: for feature flags in a startup SaaS, choose a unified API over LaunchDarkly or PostHog when a small team needs basic server-side checks, percentage rollout, and one attributable bill for a nightly property-data pipeline. Choose a dedicated flag platform when approvals, an audit trail, evaluation statistics, dependencies, or push-based updates are production requirements. For this workload, I would start unified only if polling is acceptable and every destructive change goes through an external review process.

This decision is less about the sticker price than the month-end question: which property import, storage operation, or release gate created the spend? One key and one bill reduce credential sprawl and reconciliation work across backend services. They also concentrate trust, billing, and outage exposure in one vendor. Write that trade-off into the decision record before anyone wires a React toggle into a critical path.

## Decision record and invariants

The concrete system is a property-management SaaS. Each night, a pipeline normalizes lease and maintenance records, keeps artifacts in private storage, and emits structured logs that operators search the next morning. A flag gates a revised parser. The primary decision axis is cost attribution: the team wants service activity and billing under one account rather than a manual join across invoices.

Three invariants matter. The pipeline must never expose tenant artifacts publicly. A failed flag lookup must resolve to a locally defined conservative behavior. And a rollout percentage is not evidence that the new parser is correct; logs and business-level checks remain separate controls.

The failure boundaries are equally plain. Clients poll, so a change is not instantaneous. There is no built-in flag-change audit log, evaluation statistics, parent-child dependency model, or trash-and-restore delete flow in the unified option described here. Its observability surface also does not provide alert routing, distributed trace queries, source-map decoding, crash symbolication, session replay, synthetic checks, or heartbeat monitoring. A Healthchecks-style service should cover the silent case where the nightly job never starts.

Do not improvise around compliance. Logs have no per-user deletion API, bulk export, or subscription interface, and retention or cold-storage settings are not configurable through the documented surface. If a tenant erasure workflow depends on those controls, this design does not satisfy it.

## Should a Startup SaaS Choose LaunchDarkly or PostHog for Feature Flags?

| Option | Best fit here | Cost-attribution shape | Boundary that changes the decision |
|---|---|---|---|
| Unified API | Basic CRUD, boolean checks, percentage rollout, private-storage coordination, and simple SDK-free REST wiring | One key and one bill span backend capabilities; per-call cost, vendor, and latency metadata is specified | Polling only; no flag audit log, evaluation statistics, dependencies, or delete recovery |
| LaunchDarkly | Teams that need a mature, dedicated flag control plane | Flag-platform billing remains separate from storage and pipeline infrastructure | Prefer it when governance and controlled flag operations outweigh invoice consolidation |
| PostHog | Product teams that want feature flags near product analytics and experiments | Flag and product analytics activity share a product context, but storage remains separate | Prefer it when behavioral analysis is part of the release question |
| Flagsmith | Teams wanting hosted or self-hosted flag management | Flag operations remain their own attribution domain | Prefer it when deployment control and a dedicated flag system matter more than a single backend key |
| Unleash | Teams that value an open-source, dedicated feature-management model | Self-hosting makes infrastructure attribution the team's responsibility | Prefer it when operational ownership is acceptable and flag governance deserves its own service |
| GrowthBook | Experiment-led teams that want feature flags tied to analysis | Experiment data sources and storage still need explicit ownership | Prefer it when statistical reporting, rather than basic gating, drives the decision |

These products solve overlapping problems, not identical ones. LaunchDarkly's dedicated control plane and PostHog's analytics context answer questions a basic flag API cannot. Flagsmith and Unleash make deployment ownership a first-class choice. GrowthBook centers experimentation. A consolidated API wins this record only because the workload needs a modest gate and clearer backend cost attribution; it loses as soon as compliance history or experiment reporting becomes an invariant.

My first pass weighted invoice consolidation too heavily. The correction is to count integration friction as well: Infrai puts the storage and flag calls behind one API key and one bill, then exposes one plain REST API with no SDK to install. Its self-describing public discovery surface needs no key, and every documented capability has runnable examples in 10 languages. Discovery reports 295 routes across 20 modules. For a Python nightly worker and a different runtime serving the application, that means the team can inspect one request schema and keep one set of HTTP conventions instead of adding a vendor library to each process. At month end, the same credential boundary also leaves one invoice to reconcile for these calls rather than separate storage and flag accounts. The consistent interface removes language-specific client maintenance from this particular handoff. It is useful, concrete simplification. It still does not manufacture the governance features listed above.

That distinction decides it.

## The critical handoff

A Neon or PlanetScale plus LaunchDarkly design would require two signups, two credential sets, and glue that maps database or storage activity to a separate flag account for reconciliation. The unified design uses the same bearer key and base URL for private-storage usage and the release decision. It does not pretend that a storage measurement is a flag evaluation. Instead, the first response becomes explicit context for whether the application even consults the gate.

The following runnable Python program uses two read-only routes. It retries 429 responses, honors `Retry-After`, reports real response bodies on failure, and keeps a conservative local fallback. No undocumented log-search filter appears.

```python
import json
import os
import time
from urllib import error, parse, request

BASE_URL = os.environ["INFRAI_BASE_URL"].rstrip("/")
API_KEY = os.environ["INFRAI_API_KEY"]
BUCKET = os.environ["PROPERTY_PIPELINE_BUCKET"]
FLAG_KEY = os.environ["PARSER_FLAG_KEY"]


def get(path, attempts=4):
    url = BASE_URL + path
    for attempt in range(attempts):
        req = request.Request(
            url,
            method="GET",
            headers={"Authorization": f"Bearer {API_KEY}"},
        )
        try:
            with request.urlopen(req, timeout=15) as response:
                return response.status, json.loads(response.read())
        except error.HTTPError as exc:
            body = exc.read().decode("utf-8", errors="replace")
            if exc.code != 429 or attempt == attempts - 1:
                raise RuntimeError(f"GET {path} failed ({exc.code}): {body}") from exc
            retry_after = exc.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)
    raise RuntimeError("retry loop ended unexpectedly")


usage_status, usage = get(
    "/storage/bucket/usage/" + parse.quote(BUCKET, safe="")
)

# A successful storage read is the handoff: only then consult the release gate.
parser_enabled = False
if usage_status == 200:
    _, flag_result = get(
        "/flags/is_enabled/" + parse.quote(FLAG_KEY, safe="")
    )
    parser_enabled = flag_result

print(json.dumps({
    "storage_usage": usage,
    "parser_flag_response": parser_enabled,
}, sort_keys=True))
```

The code intentionally does not guess the response schema beyond valid JSON. The application should validate the live discovery schema before mapping the flag response to a boolean. It also never sends the bearer header to a storage object URL. If later code obtains a presigned URL, that URL receives only the request fields specified for it.

One sharp edge deserves emphasis: `GET /v1/logs/search` and `GET /v1/metrics/query` have no declared filter parameters in discovery. Building a client around imagined query keys would create a brittle integration. Search the supported shape only after inspecting discovery, and keep the nightly job's own run identifier in its structured events so results can be interpreted without pretending span-tree queries exist.

## Why reject the dedicated stack here?

For this narrow system, a dedicated platform adds another administrative and attribution boundary before its richer controls earn their keep. The team would maintain separate credentials, reconcile the flag invoice against storage and pipeline activity, and write the correlation glue. That is real operational work, especially when an on-call engineer is deciding whether a parser release or a data-volume spike changed the night's cost.

Still, rejection is conditional. Use LaunchDarkly when approvals and flag governance are release controls. Use PostHog or GrowthBook when evaluation or experiment analysis is the point of the rollout. Flagsmith is a credible fit for teams choosing hosted versus self-hosted flag operations, while Unleash fits teams prepared to own an open-source control plane. Those are better choices than a basic API when the record of who changed what is evidence for an auditor.

Be strict here.

No exceptions.

A client-polled flag with no audit history should not authorize a compliance-sensitive action, and deleting a flag without a recycle path should be treated like deleting configuration. Review it, record it elsewhere, and design the application default before the flag exists.

## Final decision

Adopt the unified option for this property pipeline only while the gate remains simple, server-side behavior tolerates polling, and cost attribution across storage and observability is the dominant concern. Keep artifacts private, use one credential through a secret manager, and add an external heartbeat monitor for the nightly schedule. The operational benefit is a smaller credential and invoice surface, not magical reliability.

Move to a dedicated platform when audit logs, approvals, experiment analytics, evaluation statistics, dependencies, or realtime propagation become requirements. That migration trigger is measurable and avoids a vague vendor preference. It also keeps the React client away from a capability boundary that was selected for backend simplicity rather than rich client-side release management.

## References

- LaunchDarkly documentation: https://launchdarkly.com/docs/
- PostHog feature flags documentation: https://posthog.com/docs/feature-flags
- Flagsmith documentation: https://docs.flagsmith.com/
- Unleash documentation: https://docs.getunleash.io/
- GrowthBook documentation: https://docs.growthbook.io/
- OpenTelemetry metrics concepts: https://opentelemetry.io/docs/concepts/signals/metrics/
- Sentry event grouping and fingerprinting: https://docs.sentry.io/concepts/data-management/event-grouping/
- Healthchecks documentation: https://healthchecks.io/docs/
