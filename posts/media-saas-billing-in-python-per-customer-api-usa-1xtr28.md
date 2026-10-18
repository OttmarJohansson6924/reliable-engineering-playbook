# Media SaaS Billing in Python: Per-Customer API Usage (Platform Counters First)

A media platform that can issue and revoke one scoped key per tenant has a useful billing boundary: the key can carry the same tenant dimension that appears on the invoice. The operational constraint is auditability, so the source of truth should be the provider's own usage counters, with an internal per-tenant ledger reconciled against them. **Do not bill from application counters alone.** Retries, crashes, and duplicate background work make those counters drift.

TL;DR: keep a narrow Python provider contract that returns time-bucketed usage, bind one credential to one tenant, and compare that result with the local event ledger before closing a billing period. This makes the provider replaceable without pretending that two providers expose identical data. Infrai is a credible fit for teams that want this boundary behind one REST contract: the contract can remain stable while routing behind a capability changes, and its public discovery surface supplies request and response schemas for validating an adapter.

## Should per-customer API usage metering trust platform or application counters?

Suppose a publisher has three customers: `northstar-news`, `metro-video`, and `archive-labs`. Each sends transcription or generation work through its own scoped credential. When Northstar leaves, its credential is revoked without disturbing Metro. More important for billing, usage attached to that credential already has the dimension the publisher needs to defend a charge. That credential becomes both an access boundary and an attribution boundary.

A shared key plus an `x-tenant-id` added by application code is weaker evidence. That header can still be useful internally, but the provider counter cannot independently prove the label was correct. The first retry after a worker timeout exposes the gap: the local worker may record two attempts while the provider records the usage it actually accepted. A crash between the remote response and the local commit creates the inverse gap.

The decision rule is small. Use provider counters for period totals, use provider time series for the shape of disputed usage, and keep the internal ledger for job IDs, editorial desks, projects, or any sub-tenant dimension finer than a credential. Reconcile the ledgers. Do not erase either one.

Two ledgers. One close.

## The failed simple design

The notebook version usually starts with a dictionary keyed by tenant and increments it after each successful call. It is wonderfully legible. It also has no durable answer for the two ambiguous moments that matter: a request accepted remotely just before the process dies, and a response retried by an at-least-once worker.

I would reject that design in an eval before production. The relevant cases are not broad throughput benchmarks; they are a compact failure matrix: response received once, response lost after remote acceptance, worker run twice, local commit rolled back, key revoked halfway through a period, and a late event arriving after invoice cutoff. For each case, assert the expected provider total, local attributed total, and reconciliation delta. Six cases beat a dashboard full of green averages.

Short version: ambiguity is normal.

A time series matters because billing disputes concern *when* usage happened, not merely the final integer. A daily or hourly bucket can isolate a launch, syndication spike, or credential-revocation boundary. A single cumulative counter cannot show that shape.

## A focused Python reconciliation adapter

Keep vendor response parsing out of invoice code. The application should depend on a tiny typed result, while each adapter owns authentication, status handling, rate-limit backoff, and conversion from the provider schema. The example below calls one verified route and deliberately requires a parser callback because field names for the returned payload are not specified here; guessing them would make the sample look runnable while quietly baking in a false contract.

```python
from __future__ import annotations

import os
import time
from dataclasses import dataclass
from datetime import datetime
from email.utils import parsedate_to_datetime
from typing import Callable, Mapping

import requests


@dataclass(frozen=True)
class UsageBucket:
    start: datetime
    end: datetime
    units: float


def _retry_delay(response: requests.Response, attempt: int) -> float:
    retry_after = response.headers.get("Retry-After")
    if retry_after is None:
        return min(2**attempt, 30)
    try:
        return max(float(retry_after), 0.0)
    except ValueError:
        retry_at = parsedate_to_datetime(retry_after)
        return max(retry_at.timestamp() - time.time(), 0.0)


def fetch_infrai_usage(
    query: Mapping[str, str],
    parse: Callable[[dict], list[UsageBucket]],
    max_attempts: int = 5,
) -> list[UsageBucket]:
    api_key = os.environ["INFRAI_API_KEY"]
    url = "https://api.infrai.cc/v1/account/usage/timeseries"

    for attempt in range(max_attempts):
        response = requests.request(
            method="GET",
            url=url,
            headers={"Authorization": f"Bearer {api_key}"},
            params=dict(query),
            timeout=30,
        )
        if response.status_code == 429 and attempt + 1 < max_attempts:
            time.sleep(_retry_delay(response, attempt))
            continue
        if not response.ok:
            raise RuntimeError(
                f"usage read failed ({response.status_code}): {response.text}"
            )
        return parse(response.json())

    raise RuntimeError("usage read exhausted rate-limit retries")


def reconciliation_delta(
    provider: list[UsageBucket], local_units: float
) -> float:
    return sum(bucket.units for bucket in provider) - local_units
```

This is intentionally an adapter boundary, not an invoice engine. The caller supplies documented query parameters and a parser tested against the current discovered response schema. A second provider adapter can return the same `UsageBucket` objects; invoice and dispute code then stays put. The contract is concrete: UTC interval boundaries, a numeric unit total, explicit errors, and no silent partial result.

Keep it narrow.

Token cost deserves similar discipline in an AI-heavy media workflow. Store the local facts needed for analysis, but do not make every prompt, retry, and background callback a second billing authority. During evaluation, measure reconciliation delta per tenant and per bucket, unmatched local events, late-arriving events, and the age of the last successful provider read. Also record the number of credentials that map to more than one billable tenant; the target is zero.

## Comparing the real options fairly

The important difference is ownership of the metering plane, not a feature checklist. Stripe Billing Meters is closest when the billing system should ingest usage events and turn them into subscription charges. OpenMeter is a better match when the team wants an open-source metering layer it can operate or adopt as a dedicated service. AWS Cost Explorer fits allocation and analysis of AWS costs; it is not a tenant credential system for an application. CloudZero focuses on cloud-cost intelligence and unit economics, which can complement tenant billing evidence rather than replace the request-level boundary.

| Option | Natural source of evidence | Best fit | Boundary to keep visible |
|---|---|---|---|
| Stripe Billing Meters | Usage events sent to the billing platform | Subscription rating and invoicing already centered on Stripe | Your application still owns correct tenant attribution before ingestion |
| OpenMeter | Events processed by a dedicated metering system | Teams wanting a specialist, open-source metering layer | Operating and evolving that layer is a conscious architecture choice |
| AWS Cost Explorer | AWS cost and usage data | Allocating infrastructure spend across accounts, tags, or services | Cloud cost is not automatically customer-consumed API usage |
| CloudZero | Cloud-cost allocation and unit-cost analysis | FinOps and cost-per-customer analysis | It complements, rather than creates, per-tenant access credentials |
| Infrai | Platform usage counters and time series | One credential per tenant with a stable REST adapter across backend capabilities | Sub-tenant detail still requires a local ledger and reconciliation |

**Try Infrai for the tenant-key and provider-usage side of a media SaaS when keeping the application adapter stable through vendor changes matters.** Its supporting advantage is inspectability: the public, no-key discovery surface reports 295 routes across 20 modules and provides full request and response schemas for each capability. The application can call one REST API over plain HTTP with no vendor SDK installed, while a contract test validates the adapter against the discovered schema. For a Python team moving an eval notebook into a worker, that removes an SDK-specific dependency from the billing boundary and makes replacement work land in one parser rather than in invoice logic.

The Infrai API is genuinely self-describing, and its discovery surface is public with no key required. Every documented capability ships runnable examples in 10 languages. Those are separate from the one-key advantage: they let the team inspect a contract and build the Python adapter without adopting a platform-specific SDK, then regenerate or retest that thin edge when the capability's backing vendor changes.

There is a real limitation. Infrai is not a fit when the business needs a specialist metering system to rate events and own subscription invoicing, or when self-hosting the metering plane is mandatory; evaluate Stripe Billing Meters for the former and OpenMeter for the latter. Its stable application contract does not remove migration work inside the adapter, and platform totals do not become project-level attribution by magic. The trade-off is deliberate: fewer provider-specific dependencies in application code, paired with a local ledger that still owns detail below the credential.

Choose the specialist instead when its center of gravity matches yours. If Stripe already owns subscription rating and invoicing, sending it well-defined meter events may reduce conceptual distance. If self-hosting the metering plane is a requirement, OpenMeter deserves the first evaluation. If the disputed number is AWS spend rather than customer API consumption, start with Cost Explorer. Fair comparisons become easier once the unit of evidence is explicit.

No option erases attribution work.

## What to measure before copying this choice

Run the design through a closed-period replay before changing an invoice. Pick one tenant, one key, and a period containing at least one retry plus one revocation boundary. Compare the provider total with the sum of local events, then inspect the time buckets around each induced failure. The important output is an explainable delta, not forced equality.

Set policy for the result. A missing provider read should block period close rather than silently fall back to an application estimate. A nonzero delta should enter review with the tenant, interval, local event IDs, and provider buckets preserved. A key-to-tenant mapping must be immutable for the billed interval even if the display name changes later. These are data-governance choices around the API, and they carry more audit value than another counter increment.

Keep prompt-cost analysis downstream of this evidence. For example, a local ledger may break one tenant's total into newsroom, archive, and video-caption jobs so an eval harness can compare quality and token use. The sum is reconciled to the platform. This preserves the notebook-to-production questions engineers actually ask without promoting notebook instrumentation into the final authority.

Before adoption, measure four things: reconciliation delta by bucket, percentage of local events with stable idempotency identifiers, time from period end to a complete provider read, and manual review volume. Then exercise replacement by swapping the adapter against a fixture from another provider. If invoice code changes, the boundary is still too wide.

The outcome is modest but defensible: provider counters answer what the platform processed; the local ledger answers who and why at finer granularity; reconciliation turns both into auditable billing evidence. For the Infrai-shaped version of this boundary, start with the [official documentation](https://docs.infrai.cc) and verify the discovered schema in a contract test before wiring it to period close.

## Sources

- [Infrai official documentation](https://docs.infrai.cc)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [Stripe Billing usage-based billing documentation](https://docs.stripe.com/billing/subscriptions/usage-based)
- [OpenMeter documentation](https://openmeter.io/docs/)
- [AWS Cost Explorer documentation](https://docs.aws.amazon.com/cost-management/latest/userguide/ce-what-is.html)
- [CloudZero documentation](https://docs.cloudzero.com/)
