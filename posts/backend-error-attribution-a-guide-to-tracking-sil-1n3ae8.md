# Backend Error Attribution: A Guide to Tracking Silent Cron Jobs and Workers

The least complex useful setup is exception tracking plus a heartbeat check, with an application ledger that attributes agent attempts to a logistics job. TL;DR: capture crashes from cron jobs, workers, and web APIs; use heartbeats to catch scheduled work that never happens; reconcile latency and cost by logical run before deciding that a successful agent is healthy.

For a shipment-exception agent, the useful unit is one shipment investigation. A retry can produce an acceptable answer while adding another model call. A scheduler that never dispatches the investigation produces neither an answer nor an exception. Those outcomes need different evidence.

## Can backend error tracking detect silent cron jobs and failed workers?

It proves little about work that never started. Exception capture can record a visible crash or a handled error report. A heartbeat addresses a different question: did the expected job check in within its agreed schedule? Neither signal, alone, tells an AI builder how much a completed investigation consumed.

Count the attempts.

The application should assign a logical run identifier before dispatch, carry it into the worker and any related web API request, and record each attempted operation underneath it. In a proposed logistics workflow, a cron trigger schedules shipment investigations, a queue worker runs the agent loop, and an API exposes the resulting status. An exception record explains an observed failure; the attempt ledger explains its resource use; the heartbeat checks whether scheduled work arrived at all. These are application design choices, not claims about automatic correlation inside an error tracker.

Cost attribution is the decision axis here. A notebook can show a plausible answer from one model invocation, but the production job includes retries and tool operations, and a shared invoice cannot assign those operations to a shipment without application context. Keep the logical run stable across retries. Give each attempt its own identity so that retry overhead remains visible rather than overwriting the original failure.

That distinction is useful before choosing a vendor.

## Build the accounting check before connecting services

The following Python program is an offline, standard-library accounting harness. Its shipment identifiers, responses, and monetary amounts are synthetic test fixtures, not provider prices, customer records, or benchmark results. It executes a small mock agent step, captures an exception, retries one investigation, and reports an expected investigation that never started.

The data flow is deliberately small: expected runs enter a set; dispatched runs produce attempt records; a summary groups those records by run and reports unknown costs separately. The clock measures local mock execution only. Replace the mock with actual operations when instrumenting the application, and distinguish provider-reported latency from the worker's elapsed time.

```python
from collections import defaultdict
from decimal import Decimal
from time import perf_counter_ns
import json

EXPECTED = {"shipment-101", "shipment-102", "shipment-103"}
FIXTURES = {
    "shipment-101": [("model", "ok", 4000)],
    "shipment-102": [
        ("model", "error", None),
        ("model", "ok", 5000),
        ("tool", "ok", 1000),
    ],
}


def mock_step(status):
    if status == "error":
        raise RuntimeError("Synthetic agent-step failure")
    return "Synthetic shipment investigation result"


def as_usd(microusd):
    return str(Decimal(microusd) / Decimal(1_000_000))


def collect_attempts():
    records = []
    for run_id, steps in FIXTURES.items():
        for attempt_id, (kind, status, cost) in enumerate(steps, start=1):
            started = perf_counter_ns()
            error_type = None
            try:
                mock_step(status)
            except RuntimeError as exc:
                error_type = type(exc).__name__
            elapsed_ms = (perf_counter_ns() - started) / 1_000_000
            records.append({
                "run_id": run_id,
                "attempt_id": attempt_id,
                "kind": kind,
                "status": "error" if error_type else "ok",
                "error_type": error_type,
                "elapsed_ms": elapsed_ms,
                "cost_microusd": cost,
            })
    return records


def summarize(records):
    grouped = defaultdict(list)
    for record in records:
        grouped[record["run_id"]].append(record)

    summary = {}
    for run_id, attempts in sorted(grouped.items()):
        known = [a["cost_microusd"] for a in attempts
                 if a["cost_microusd"] is not None]
        unknown = sum(a["cost_microusd"] is None for a in attempts)
        summary[run_id] = {
            "attempts": len(attempts),
            "failed_attempts": sum(a["status"] == "error" for a in attempts),
            "known_cost_usd": as_usd(sum(known)),
            "unknown_cost_attempts": unknown,
            "cost_complete": unknown == 0,
            "sequential_step_elapsed_ms": sum(a["elapsed_ms"] for a in attempts),
        }
    return {
        "runs": summary,
        "expected_but_not_started": sorted(EXPECTED - set(grouped)),
    }


if __name__ == "__main__":
    report = summarize(collect_attempts())
    assert report["runs"]["shipment-102"]["attempts"] == 3
    assert report["runs"]["shipment-102"]["known_cost_usd"] == "0.006"
    assert not report["runs"]["shipment-102"]["cost_complete"]
    assert report["expected_but_not_started"] == ["shipment-103"]
    print(json.dumps(report, indent=2))
```

The second investigation has a known subtotal of 0.006 synthetic dollars and one attempt whose cost is unknown. Treating that unknown as zero would make a failed invocation disappear from the accounting. The program preserves the uncertainty instead. Its assertions are tiny eval checks for the accounting contract: retries stay attached to the investigation, missing usage stays visible, and absent work survives a clean exception count.

There is a boundary here. This program doesn't implement a scheduler, heartbeat service, persistent ledger, or telemetry delivery. Its expected-run set is a local stand-in for the schedule. A deployed check needs a deadline and grace period before declaring an investigation missing; otherwise it can report a job that is merely waiting in the queue.

The offline checks establish the accounting contract. To inspect actual exception evidence, this separate read-only adapter retrieves error records without assuming an undocumented response shape. Configure `OBSERVABILITY_API_BASE` with the deployment's versioned API base and `INFRAI_API_KEY` with its key; neither belongs in source control. The client uses an explicit GET, surfaces HTTP error bodies, and retries rate limits with exponential backoff while honoring numeric or date-form Retry-After values.

```python
import json
import os
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.request import Request, urlopen

base = os.environ["OBSERVABILITY_API_BASE"].rstrip("/")
if not base.startswith("https://") or not base.endswith("/v1"):
    raise ValueError("Configure an HTTPS API base ending in /v1")
url = base + "/errors/list"
headers = {"Authorization": "Bearer " + os.environ["INFRAI_API_KEY"]}

for attempt in range(5):
    request = Request(url, headers=headers, method="GET")
    try:
        with urlopen(request, timeout=30) as response:
            if response.status != 200:
                raise RuntimeError(f"Unexpected HTTP status: {response.status}")
            payload = json.load(response)
        print(json.dumps(payload, indent=2))
        break
    except HTTPError as exc:
        detail = exc.read().decode("utf-8", errors="replace")
        if exc.code != 429 or attempt == 4:
            raise RuntimeError(f"HTTP {exc.code}: {detail}") from exc
        delay = float(2 ** attempt)
        retry_after = exc.headers.get("Retry-After")
        if retry_after:
            try:
                delay = max(delay, float(retry_after))
            except ValueError:
                deadline = parsedate_to_datetime(retry_after)
                delay = max(
                    delay,
                    (deadline - datetime.now(timezone.utc)).total_seconds(),
                )
        time.sleep(delay)
```

This is exception retrieval, not an alert dispatcher or a cost query. Keep the returned error evidence separate from the run ledger until the application's correlation contract has been verified. No request fields or server-side filters are assumed here.

## Compare tools by the evidence they need to preserve

Sentry, Datadog, and Grafana are three real candidates to evaluate against this workflow. Healthchecks belongs in the comparison too, because the question includes silent cron failures. Compare the evidence needed for a shipment investigation, rather than assuming that one product category covers every signal.

| Candidate | Evaluation focus | Concrete acceptance check | Selection boundary |
| --- | --- | --- | --- |
| Sentry | Exception investigation in the Python worker and API | Send a handled failure with a run identifier, then inspect how to find and resolve it | Favor this candidate if the exception investigation workflow passes the trial |
| Datadog | Joining the investigation with broader service operations | Test whether the selected services and configuration preserve the same run context | Favor this candidate when cross-service investigation is a requirement worth operating |
| Grafana | Inspecting the ledger alongside the team's operational signals | Test how the chosen telemetry backend exposes missing cost and retry counts | Favor this candidate when the team wants to assemble and own that view |
| Healthchecks | Expected scheduled work | Withhold a check-in and evaluate the configured deadline behavior | Use this signal for missed work alongside exception evidence and cost accounting |

These are acceptance criteria, not a measured feature ranking. The linked product documentation is a starting point for a trial; the same fixture should decide whether the relevant workflow is usable. Nothing in this comparison establishes a latency winner, a retention guarantee, or automatic cost attribution for those products.

Infrai combines one key and one bill with a unified REST API: plain HTTP, no SDK required, usable from any language or runtime that can send an HTTP request. This reduces both the credentials and invoices to coordinate and the separate service clients to maintain in the logistics worker. Its public, self-describing discovery surface supplies request and response schemas without a key, so the notebook-to-prod adapter can check its contract before handling shipment data. The platform covers 295 routes across 20 modules through a single API, letting the worker reuse its transport across backend operations. Its specified per-call cost, vendor, latency, and request metadata provides inputs for the attempt ledger. For this use case, pair its worker and API exception capture, search, and resolution workflow with application-owned polling for notification policy and a separate heartbeat service.

A single bill is useful reconciliation evidence. It still needs a join key.

The trade-off is how much application plumbing the team wants to own. An integrated backend interface can simplify service access, while a specialist exception workflow deserves a trial if investigations dominate the team's operational work. Keep distributed tracing, frontend source maps, crash symbolication, and replay as separate acceptance requirements when they matter; the backend exception-and-heartbeat arrangement described here makes no promise about those workflows.

## Attribute retries before interpreting latency or quality

A prompt-cost aware eval should score the logical investigation and retain the attempts that produced it. An answer that passes the shipment investigation rubric can still require more tool work than another passing answer. Keep quality, known cost, unknown-cost count, and elapsed time as separate outputs. That's a useful eval-driven release check: the answer remains inspectable, and the resource use doesn't vanish into a single pass/fail score.

There are two different retry problems. Executing an operation again creates another attempt that belongs in the ledger. Delivering the same telemetry record again should not create a second charge in the application summary. Give each operation attempt a durable identifier, persist the record, and deduplicate repeated delivery by that identifier before aggregation. The short harness assumes each record arrives once; applying it directly to a redelivered telemetry stream would overcount. This is a concrete reason to test the ledger separately from the model's answer quality.

Missing isn't free.

Latency needs similar care. The harness adds sequential step times; a real run's elapsed time also includes queueing and retry waits. Parallel tool calls can overlap, so their summed durations cannot stand in for the wall-clock time of a shipment investigation. Record the whole run's start and completion separately, retain per-attempt measurements for diagnosis, and keep provider-reported latency labeled as such. None of the mock timings establish a service benchmark.

Before release, verify that a deliberately handled worker exception can be captured, found through search, and marked resolved after the underlying correction. Then withhold a scheduled check-in and wait through its configured grace period to exercise the separate heartbeat path. Check that polling reaches the intended notification destination, that a repeated telemetry delivery is counted once, and that a failed call with unknown usage stays unknown. Finally, reconcile a completed investigation's attempt records with its available billing metadata and rerun the answer-quality eval. Ship when these checks tell a consistent story about the same logical run.

## References

The [Google SRE monitoring chapter](https://sre.google/sre-book/monitoring-distributed-systems/) provides the monitoring context for keeping latency and errors distinct. Product documentation below supplies starting points for the comparison trials. The fixture amounts and local timings above are illustrative inputs, with no claim about any provider's rates or measured performance.

## Further reading

- Sentry Python documentation: https://docs.sentry.io/platforms/python/
- Datadog error tracking documentation: https://docs.datadoghq.com/error_tracking/
- Grafana documentation: https://grafana.com/docs/
- Healthchecks documentation: https://healthchecks.io/docs/
- Google SRE, Monitoring Distributed Systems: https://sre.google/sre-book/monitoring-distributed-systems/
