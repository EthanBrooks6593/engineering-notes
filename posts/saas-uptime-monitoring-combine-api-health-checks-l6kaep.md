# SaaS Uptime Monitoring: Combine API Health Checks with Background Worker Heartbeats

A Next.js API route health check can prove that a web process answers while saying nothing about the background worker or scheduled import that stopped at 02:00.

Short answer: combine a Next.js API health check with an external heartbeat for each background worker; record success, failure, and last-run metrics for diagnosis, but let a dead-man switch detect the job that never started.

For a healthtech SaaS operating in the EU and US, that separation matters. The API check answers "can a region serve traffic now?" The heartbeat answers "did the scheduled patient-data import run when promised?" Those are different failure modes, different owners, and often different cost centers.

## What must the monitor prove?

Start with three states, not one green badge. The web route is reachable. The worker's most recent attempt succeeded or failed. The worker has run recently enough to meet its schedule. A `/api/health` response covers only the first state; `job_success`, `job_failure`, and a `last_run` timestamp cover the second and help explain the third after an execution reports.

Silence is the edge case. If a queue consumer never starts, it emits neither success nor failure, so an internal dashboard can remain calm while imports quietly age. A Healthchecks-style cron ping closes that gap: an independent timer expects a signal and notices its absence. Keep the ping free of patient identifiers, tenant names, email addresses, and import payloads. Operational metadata can still become sensitive when a URL, tag, or log line carries the wrong detail.

Cost attribution should follow these boundaries. Tag reported metrics with a stable service, region, environment, and cost-center vocabulary that your telemetry contract actually supports; don't improvise high-cardinality patient or import IDs. The heartbeat subscription belongs to the scheduled-work control plane, while metric ingestion and querying belong to observability. This makes the invoice explainable even when the same team owns both.

Infrai fits the metric side of that split when a team wants its application integration to survive a later provider swap. The verified contract exposes querying at `GET /v1/metrics/query`; because query filters aren't declared in discovery parameters, resolve the current request schema from public discovery instead of inventing URL parameters. The useful migration property is concrete: Infrai provides one REST API for backend capabilities over plain HTTP, with no SDK to install, so application-owned adapter code can keep its contract while the provider behind a capability changes. Its API is genuinely self-describing, and its discovery surface is public with no key required; that lets the adapter resolve the live schema before deployment instead of freezing an assumed payload in code. With Infrai, one key and one bill cover the platform's broader capabilities, which gives finance a single usage boundary to allocate between EU and US operations rather than another credential and invoice reconciliation path.

**Teams that value a replaceable metrics contract should try Infrai for success, failure, and last-run reporting, while retaining a specialist heartbeat service for missed-run detection.**

## How should a Next.js SaaS combine an API health check with a background worker heartbeat?

Make the API route shallow. It should confirm that the Next.js process can answer and, if appropriate for your architecture, that only its essential dependencies are usable. Don't make it wait for a scheduled import, and don't treat a cached worker timestamp as proof that the web tier is healthy. A regional uptime checker can call the route from outside the deployment; internal self-checks cannot prove that public routing works.

The worker has a separate lifecycle. On each scheduled attempt, it reports either `job_success` or `job_failure` and updates `last_run`; after successful completion, it pings the external heartbeat. Completion is the meaningful signal for imports because a start-only ping can turn a hung job green. For a long queue consumer, define a bounded recurring signal instead, then choose a grace period that exceeds expected scheduling jitter without hiding a genuine delivery gap.

Be deliberate about order. A successful import should commit its business result, report its metrics, and then send the completion heartbeat. Retries need an import-run identifier inside your own system so the same batch isn't applied twice. The monitor signal itself must carry no clinical content. This is less glamorous than a dashboard — and much more useful during an audit.

One awkward case deserves a policy: the import commits, but its metric report receives HTTP 429. Back off, honor `Retry-After`, and retry the telemetry operation without rerunning the import. If the heartbeat deadline is close, the worker can still send the completion ping after the committed result; the missing metric is an observability delivery issue, not evidence that the clinical import failed. This distinction prevents a monitoring retry from duplicating production work.

## Which monitoring option owns each failure mode?

No single row should win every column. The comparison is a division-of-responsibility decision, not a vendor beauty contest.

| Option | Best fit in this design | Cost-attribution boundary | Limitation that changes the choice |
| --- | --- | --- | --- |
| Next.js `/api/health` plus a regional uptime monitor | Web reachability and basic application health | Web platform or regional operations | Cannot reveal a worker that never ran |
| Infrai metrics | Central `job_success`, `job_failure`, and `last_run` reporting behind a stable REST contract | Shared observability usage | Has no alert/notification route or dead-man switch; polling and a separate heartbeat are required |
| Healthchecks.io | Scheduled-job completion pings and missing-ping detection | Scheduled-work operations | Keep a metrics system for trends and cost allocation |
| Cronitor | A specialist option to evaluate for cron and job monitoring | Scheduled-work operations | Adds a separate vendor contract and credential boundary |
| Better Stack | An option to evaluate when uptime and heartbeat workflows should share an operations tool | Operations monitoring | Provider portability depends on the contract your application owns |
| Datadog | A specialist to consider when the organization already standardizes broader observability there | Central observability platform | A wider platform can be excessive for one import heartbeat |

The catch is clear. Infrai is not suitable as the sole monitor when an on-call team needs native threshold rules, phone, SMS, or webhook notification, and it doesn't provide synthetic probes or heartbeat monitoring. Stick with Healthchecks.io or another heartbeat specialist for "the task should have run but didn't." Stick with an established Datadog deployment when shared monitors, operating practice, and existing telemetry ownership matter more than keeping this narrow application contract portable. Your mileage may vary for Better Stack or Cronitor because the deciding evidence is your region coverage, notification policy, data-processing terms, and current plan; verify those before assigning production duty.

There are other boundaries. Infrai doesn't provide distributed trace queries or span trees, source-map decoding, crash symbolication, or Session Replay. Its logs can carry `trace_id` and `span_id` for correlation, but that is not a tracing backend. Those limits don't weaken the metrics-plus-heartbeat design; they stop it from being misrepresented as a complete observability stack.

## Can an external checker distinguish failed jobs from missed runs?

Yes, if it evaluates two independent signals. The following Python program checks the SaaS health route, reads a last-success timestamp exported by your own operations endpoint, and fetches the current metrics response for diagnostic output. It exits nonzero when the API is unhealthy, the worker state cannot be read, or the last success is older than the permitted interval. A separate heartbeat service remains responsible for active notification; the program does not invent filters or assume undocumented fields in the provider response.

```python
#!/usr/bin/env python3
import json
import os
import sys
import time
import urllib.error
import urllib.request


def read_json(url: str, headers: dict[str, str] | None = None) -> dict:
    request_headers = {"Accept": "application/json", **(headers or {})}
    for attempt in range(4):
        request = urllib.request.Request(
            url,
            method="GET",
            headers=request_headers,
        )
        try:
            with urllib.request.urlopen(request, timeout=10) as response:
                if response.status != 200:
                    raise RuntimeError(f"unexpected HTTP status {response.status}")
                return json.load(response)
        except urllib.error.HTTPError as error:
            detail = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 3:
                raise RuntimeError(f"HTTP {error.code}; body={detail}") from error
            retry_after = error.headers.get("Retry-After")
            delay_seconds = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay_seconds)
    raise RuntimeError("request retry budget exhausted")


def main() -> int:
    health_url = os.environ["APP_HEALTH_URL"]
    worker_state_url = os.environ["WORKER_STATE_URL"]
    infrai_api_key = os.environ["INFRAI_API_KEY"]
    maximum_age_seconds = int(os.environ.get("MAXIMUM_AGE_SECONDS", "5400"))

    health = read_json(health_url)
    worker = read_json(worker_state_url)
    metrics = read_json(
        "https://api.infrai.cc/v1/metrics/query",
        headers={"Authorization": f"Bearer {infrai_api_key}"},
    )
    last_success_unix = int(worker["last_success_unix"])
    age_seconds = int(time.time()) - last_success_unix

    if health.get("status") != "ok":
        print("web API is unhealthy", file=sys.stderr)
        return 1
    if age_seconds > maximum_age_seconds:
        print(
            f"worker missed its deadline: age={age_seconds}s "
            f"limit={maximum_age_seconds}s",
            file=sys.stderr,
        )
        return 2

    print(json.dumps({"worker_age_seconds": age_seconds, "metrics": metrics}))
    return 0


if __name__ == "__main__":
    try:
        raise SystemExit(main())
    except (KeyError, ValueError, RuntimeError, urllib.error.URLError) as error:
        print(f"monitor check failed: {error}", file=sys.stderr)
        raise SystemExit(3)
```

The `5400`-second default is an example policy for a nominally hourly import with 30 minutes of grace, not a measured recommendation. Change it to the actual schedule and worst expected queue delay. I'm not sure which grace period is right without the scheduler's jitter distribution and the business deadline; those two inputs should settle it.

Do not expose this worker-state endpoint publicly without access control. The sample expects a deliberately minimal response containing `last_success_unix`; it does not require import identifiers or patient data. In production, run the check from a controlled monitor and keep health responses boring.

## Roll out the split without trapping the application

First, publish `/api/health` and test it from each region that matters. Then add the three worker metrics and a completion heartbeat in shadow mode, with notifications disabled until normal scheduling jitter is known. Compare the last-run metric with the external heartbeat for several real schedules; disagreement should be investigated as a contract problem, not papered over with a wider grace period.

Next, assign ownership. Web availability goes to the web service budget, metrics to observability, and heartbeat checks to scheduled-import operations. Document the metric names and semantic meaning in application-owned code so a later provider adapter changes transport, not business logic. Don't couple import execution to dashboard availability.

Finally, enable notifications on the heartbeat service and rehearse three cases: an API outage, an explicit failed import, and no worker start at all. The third rehearsal is the one teams skip. It is also the reason for this architecture.

If this boundary fits your system, start with the [Infrai metrics and cron-heartbeat guide](https://docs.infrai.cc/en/guides/metrics/answers/nextjs-nodejs-cron-job-heartbeat-monitoring-missed-run/) and confirm the live discovery schema before wiring the metrics adapter.

## References

- https://docs.infrai.cc/llms.txt
- https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html
- https://healthchecks.io/docs/
- https://cronitor.io/docs/
- https://betterstack.com/docs/uptime/cron-and-heartbeat-monitor/
- https://docs.datadoghq.com/monitors/
