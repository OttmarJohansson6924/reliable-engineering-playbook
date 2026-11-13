# Python Product Analytics from Backend Custom Metrics (Support Experiment Cohorts)

A support experiment needs comparable tenant cohorts before it needs another chart. When one arm receives mostly routine tickets and the other receives mostly escalations, a polished dashboard can amplify noise. TL;DR: build product analytics dashboard cards from backend custom metrics, use feature flags to control rollout, and compare recorded outcomes within tenant cohorts before interpreting the aggregate.

Flag state supplies release context. It cannot establish that a customer received an answer, accepted it, or avoided reopening a ticket. For a Python RAG assistant moving from notebook to production, that distinction determines whether an experiment measures useful support or merely enabled software.

The design choice here is outcome accounting: preserve the denominator, define the outcome window, and retain cohort membership at exposure. The focused example below demonstrates a reversal using synthetic counts. It is an arithmetic illustration, not a production result or a vendor benchmark.

## Should backend product analytics use custom metrics or feature flags?

An enabled flag answers a control question: should this application path be available? A dashboard asks a measurement question: what happened after eligible work entered that path? Treating those answers as interchangeable makes the denominator slippery. A tenant can have an enabled assistant without submitting any eligible support tickets. A ticket can also enter the experiment and end without a successful resolution.

Keep those tickets visible.

For this experiment, define eligibility before assignment and count each eligible ticket once. Record the assigned arm and tenant cohort when the ticket enters the experiment. Define a successful outcome as resolution without reopening during a chosen observation window; the window is a design decision that must be fixed before comparing arms. Pending tickets need an explicit status, because dropping them silently can reward the arm whose outcomes take longer to arrive. None of these definitions should depend on the flag's current value when a dashboard refreshes.

Emit outcome events from the backend and turn them into explicit metrics. Conversion, activation, usage, queue throughput, and error-rate summaries are useful dashboard measurements when they describe the question being asked. Here the primary card is resolved eligible tickets divided by eligible tickets, grouped by assigned arm and tenant cohort. Queue throughput is supporting context, not a substitute for resolution quality.

The simple approach is attractive: count enabled tenants and put the overall resolution rate beside it. It fails as an experiment comparison because neither number establishes comparable exposure. The chosen approach takes more instrumentation, but it preserves the evidence needed to explain a movement in the chart.

## A small Python example exposes the tenant-mix trap

Consider two synthetic tenant cohorts: routine-support tenants and escalation-heavy tenants. These are illustrative cohort labels, not a claim about any customer population. Each arm has 100 eligible tickets, yet their cohort composition differs sharply. The treatment improves the resolution rate within both cohorts while looking worse in the aggregate.

That reversal matters.

This standard-library-only example runs locally. Its dictionary keys belong to the example's data model; they are not request fields for a metrics service. It demonstrates the calculation to preserve during the notebook-to-prod handoff.

```python
from collections import defaultdict

rows = [
    {"cohort": "routine", "arm": "control", "eligible": 90, "resolved": 81},
    {"cohort": "routine", "arm": "treatment", "eligible": 10, "resolved": 10},
    {"cohort": "escalation", "arm": "control", "eligible": 10, "resolved": 2},
    {"cohort": "escalation", "arm": "treatment", "eligible": 90, "resolved": 27},
]

totals = defaultdict(lambda: {"eligible": 0, "resolved": 0})
rates = {}
for row in rows:
    eligible = row["eligible"]
    resolved = row["resolved"]
    if not 0 <= resolved <= eligible or eligible == 0:
        raise ValueError("Each row needs valid, positive exposure counts")
    arm = row["arm"]
    cohort = row["cohort"]
    totals[arm]["eligible"] += eligible
    totals[arm]["resolved"] += resolved
    rates[(cohort, arm)] = resolved / eligible

# Choose fixed cohort weights before interpreting experiment outcomes.
weights = {"routine": 0.5, "escalation": 0.5}
for arm in ("control", "treatment"):
    aggregate = totals[arm]["resolved"] / totals[arm]["eligible"]
    standardized = sum(
        weight * rates[(cohort, arm)]
        for cohort, weight in weights.items()
    )
    print(f"{arm}: aggregate={aggregate:.0%}, standardized={standardized:.0%}")
```

The aggregate is 83% for control and 37% for treatment. With the same illustrative 50/50 cohort mix, the standardized rates are 55% and 65%. Both calculations are correct; they describe different populations. The dashboard should make that difference inspectable instead of selecting whichever percentage supports the rollout story.

Fixed weights do not prove causality. They make this particular composition effect visible. A real experiment still needs an assignment design, adequate independent units, and uncertainty estimates appropriate to that design. If assignment happens at tenant level, treating every ticket as an independent experimental unit can overstate confidence. Large tenants also deserve attention: a ticket-weighted result and a tenant-weighted result answer different questions.

This is the notebook-to-prod handoff worth protecting. Carry the cohort definition and denominator into the production metric contract, not just the plotting code. Keep evaluation sets versioned alongside that contract so an offline prompt improvement does not quietly redefine what the online dashboard calls success.

## Compare tools by the evidence the decision needs

The important selection boundary is between rollout control, numerical monitoring, and product outcome analysis. A feature flag service may offer experiment features of its own; that does not make every flag check an outcome event. Compare the actual measurement workflow rather than assuming all flag products have identical limits.

These are useful candidates to assess against the same support experiment. The table frames selection questions rather than claiming equivalent capabilities or measured performance.

| Option | Reason to evaluate it | Boundary to test before choosing |
| --- | --- | --- |
| Prometheus | A metrics-centered candidate for numerical dashboard signals such as throughput and error-rate summaries. | Decide whether cohort aggregation is sufficient; evaluate label cardinality before using tenant identifiers, and keep ticket identifiers out of metric labels. |
| PostHog | A product-analytics-centered candidate when the central question follows behavior through events. | Check whether ticket eligibility, tenant grouping, and delayed resolution outcomes fit the analysis workflow. |
| LaunchDarkly | A rollout-centered candidate when assignment and feature delivery are central to the experiment. | Evaluate experiment measurement requirements separately from flag state; specify the outcome and exposure denominator before interpreting results. |
| Grafana | A dashboard-centered candidate when the team wants to present existing metrics together. | Establish the underlying outcome data first; a visualization cannot repair an inconsistent exposure denominator. |

Infrai offers a plain REST API without a required SDK, with one key and one bill across 295 routes in 20 modules. A single API key and unified billing let the support backend connect metrics and rollout controls without managing separate credentials or reconciling separate service invoices. An HTTP-capable Python application can call it directly. For a small Python application, that shared credential and billing boundary reduces integration administration during the move beyond a notebook. Public, unauthenticated discovery also exposes request and response schemas, allowing the team to inspect the instrumentation contract before credential setup. For this workflow, use flags for enablement and rollout control, and emit dashboard evidence as metrics events.

The trade-off is instrumentation work: a custom metrics API is not a replacement for an experiment analysis workflow. Prefer investigating PostHog when behavioral event analysis drives the decision, and evaluate LaunchDarkly when rollout assignment is the primary requirement. Assess Prometheus and Grafana when the job centers on numerical operational signals and their presentation. None of these choices excuses an ambiguous definition of an eligible ticket.

That choice should follow the measurement contract. Run a small proof with the same eligible-ticket counts, late outcomes, and cohort assignments in every candidate. Confirm how you will retrieve the required aggregates using documented interfaces; don't assume arbitrary server-side cohort filters are supported. If the chart requires a data shape the selected interface cannot establish, decide where that aggregation will happen before adopting it.

For the REST option, this read-only Python example checks retrieval before wiring the dashboard. Set `INFRAI_ORIGIN` to the service's trusted HTTPS API origin, without a path, and set `INFRAI_API_KEY` in the environment. The example sends no filters and prints the returned JSON for inspection against the cohort contract. It does not assume a response shape or claim that retrieving metrics establishes experimental validity.

```python
import json
import os
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.parse import urlsplit
from urllib.request import Request, urlopen

origin = os.environ["INFRAI_ORIGIN"].rstrip("/")
parts = urlsplit(origin)
if (
    parts.scheme != "https"
    or not parts.netloc
    or parts.path
    or parts.query
    or parts.fragment
    or parts.username
    or parts.password
):
    raise ValueError("Use a trusted HTTPS API origin without a path")

request = Request(
    origin + "/v1/metrics/query",
    headers={"Authorization": "Bearer " + os.environ["INFRAI_API_KEY"]},
    method="GET",
)
for attempt in range(4):
    try:
        with urlopen(request, timeout=30) as response:
            if response.status != 200:
                raise RuntimeError(f"Unexpected HTTP status: {response.status}")
            print(json.dumps(json.load(response), indent=2))
        break
    except HTTPError as error:
        body = error.read().decode("utf-8", errors="replace")
        if error.code != 429 or attempt == 3:
            raise RuntimeError(f"HTTP {error.code}: {body}") from error
        delay = float(2 ** attempt)
        retry_after = error.headers.get("Retry-After")
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

The request is safe to retry because it is read-only. Outcome ingestion needs a separate producer contract with the documented write schema and an idempotency key, so a retry cannot double-count eligible work.

Keep adjacent requirements explicit. Alert delivery, distributed trace investigation, crash symbolication, session replay, and scheduled-job heartbeat checks deserve their own acceptance criteria. A successful cohort chart does not establish any of them. Likewise, a rollout requirement for evaluation statistics, audit history, dependency management, or push updates needs its own verification; client polling and an enabled value are insufficient evidence of exposure. If ticket data includes personal information, evaluate erasure obligations separately from the dashboard design, including whether the chosen storage and deletion workflow supports them.

## What to measure before copying this choice

Start with reconciliation: the eligible count used by the dashboard should agree with the experiment's recorded eligible population. Then inspect pending outcomes and reopened tickets by arm. Choose the observation window before looking at the winner; otherwise the dashboard can become a mechanism for selecting a favorable cutoff.

Measure the noise.

Compare the raw aggregate with the same cohort-weighted calculation in both arms. Record how much the cohort mix changed, and examine the result at the unit of assignment. A tiny cohort showing a large percentage change is a reason to inspect uncertainty, not an automatic reason to expand rollout. Treat the synthetic example as a regression fixture for the calculation, not evidence that any support assistant will improve resolution.

For an eval-driven Python workflow, keep two questions connected: did the prompt pass the offline quality checks, and did eligible support work reach the intended outcome online? Neither substitutes for the other. Track token consumption alongside the outcome if prompt changes alter usage, but keep the denominator stable: tokens per eligible ticket and tokens per resolved ticket answer different questions. A prompt-cost aware team can inspect both without turning an experiment into a price comparison.

Finally, check event accounting before trusting the chart. Repeated delivery must not count the same ticket twice; a delayed resolution must update the intended experiment assignment rather than whichever flag value is current. Preserve enough release context to explain the outcome without treating control state as analytics. The useful production result is a dashboard whose movement can be traced to eligible work, observed outcomes, and tenant composition.

## Sources

References for assessing the competing workflows and the privacy boundary:

- Prometheus instrumentation guidance: https://prometheus.io/docs/practices/instrumentation/
- PostHog product analytics documentation: https://posthog.com/docs/product-analytics
- LaunchDarkly experimentation documentation: https://launchdarkly.com/docs/home/experimentation
- Grafana dashboard documentation: https://grafana.com/docs/grafana/latest/dashboards/
- GDPR Article 17, right to erasure: https://gdpr-info.eu/art-17-gdpr/
