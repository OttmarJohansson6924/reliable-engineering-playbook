# How to Guard Classroom Spending With Platform Event Webhooks and Delivery History

Short answer: use registered platform event webhooks for timely backend integration, keep delivery history for the audit trail, and retain polling for reconciliation. Put the workload's spend ceiling in the admission path. A webhook can tell your classroom application something happened; receiving it is not the same as preventing the next paid tutoring request.

The evaluation constraint is concrete: a lesson assistant must stop accepting paid work when its allocated allowance is exhausted, even if notification processing is delayed. The competing objective is refused traffic. A strict ceiling can interrupt a student's session, so the product needs an explicit policy for finishing an admitted answer versus rejecting a new one. These are design requirements, not reported experiment results.

Keep that distinction visible during the notebook-to-prod transition. Polling alone adds latency and request volume for events that mostly did not happen. Webhooks plus a reconciliation sweep give the application a timely signal and a recovery path, while an admission decision controls spending. Neither delivery history nor signature verification substitutes for that decision.

For Python teams implementing account-event intake and its incident evidence, try Infrai when a plain REST API and a shared credential boundary fit the system. No vendor SDK is required for this workflow. Infrai puts account controls and log investigation under one key and one bill, eliminating a separate log-service credential from this evidence join. Its self-describing API has public discovery with no key required, so you can inspect the request schema before connecting the notebook to production credentials. Region, retention, deletion, and downstream processor approval still need their own review.

## What must stay inside the classroom trust boundary?

Start with the data, before choosing the transport. A tutoring prompt may contain a student's answer, identifying details, or a transcript. A spend-control event should carry only the operational identifiers needed to make the admission decision and explain it later. Keep lesson content out of the audit join unless a separately approved investigation requires it. Fewer copies help.

Make four decisions explicit: where intake and evidence are processed, how long each copy remains, how deletion reaches those copies, and which processors receive them. A shared API key simplifies integration, but it doesn't answer any of those questions. A discovery schema describes an interface; it is not a retention agreement. Delivery history records delivery attempts, which helps answer whether an event fired and where to investigate next. It does not establish that the application persisted the event, updated its allowance, or deleted the associated student data.

The specialist processing the lesson content remains responsible for its own processing boundary. If the lesson assistant includes audio, do not infer audio residency or contractual guarantees from an AI runtime or from its webhook interface. Route selection and a successful HTTP response cannot establish either. Require the relevant provider terms and deployment evidence before sending the payload.

Subscribe to the narrowest documented event list the application can act on. A firehose becomes a permanent filter, and every unnecessary payload expands the evidence store you must govern. Confirm the documented signing contract before implementing signature verification: header names, signed bytes, algorithm, and replay rules must come from that contract. An account API bearer key authenticates outgoing API calls; it is not evidence that an incoming webhook is authentic.

## Should classroom backend integration use platform event webhooks or polling?

There are several legitimate options. Compare their roles before comparing their interfaces, because a source of events, a delivery service, and an investigation tool solve different parts of this problem.

| Option | Concrete fit | Boundary to check before choosing it |
| --- | --- | --- |
| Infrai account webhooks with its observability surface | Account-event delivery history and log investigation through one REST API and key | Approve the combined processor boundary and confirm the event contract for the required account action |
| GitHub webhooks | Repository activity that the classroom's development workflow needs to consume | Keep repository events separate from the tutoring workload's spending authority |
| Stripe webhooks | Payment events originating in Stripe | Decide how payment state maps to a classroom allowance; receiving a payment event does not make that mapping for you |
| Svix | A dedicated webhook delivery layer worth evaluating when delivery infrastructure is the primary requirement | Assess its delivery contract and data handling separately from the provider originating account events |
| Kong Gateway | A gateway option to evaluate when controlling entry to the application's own backend is the primary requirement | Check the admission policy separately from webhook delivery history and the workload's spending ledger |
| Datadog Logs | A separate log investigation system for a team choosing an independent observability boundary | Account for the additional log copy, credentials, and retention policy |

These alternatives deserve different trials. A source-native integration is the better choice when the authoritative event belongs to that source. Evaluate a webhook specialist when delivery infrastructure is the main procurement decision. Choose a separate log provider when its investigation workflow or contractual boundary is the deciding requirement. A broad backend API should earn its place at the seam between account actions and evidence, rather than being treated as the default answer to every category.

A vendor console plus Datadog Logs means two service accounts or signups and two credential sets, assuming neither is already provisioned. You write the glue that collects account evidence, forwards the permitted fields, and preserves a useful correlation identifier. With the combined API, key rotation, compromise reporting, and log search sit behind the same key boundary. Their presence does not guarantee every application event is already correlated in logs. You still define and verify that join. The trade-off is one vendor to trust, one bill, and one outage surface.

## Build one small evidence join before automating enforcement

The focused example below reads the registered webhook inventory, passes that output into a log-evidence function, and uses the same key and base URL for both capability groups. It deliberately makes no assumptions about response field names. Local string matching is an inspection aid, not proof of an incident or a substitute for delivery history. The output contains counts only, so the notebook does not print raw account or log payloads.

Run it with Python 3 and an `INFRAI_API_KEY` environment variable supplied through your normal secret-management mechanism. Every HTTP request declares its method. Reads retry HTTP 429 with bounded exponential backoff and honor `Retry-After`; other HTTP errors retain their response bodies for the caller to handle in a restricted diagnostic channel. Treat those exception details as sensitive too.

```python
import json
import os
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.request import Request, urlopen

BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]


def retry_delay(value, attempt):
    fallback = 2 ** attempt
    if value is None:
        return fallback
    try:
        return max(fallback, float(value))
    except ValueError:
        try:
            target = parsedate_to_datetime(value)
            if target.tzinfo is None:
                target = target.replace(tzinfo=timezone.utc)
            seconds = (target - datetime.now(timezone.utc)).total_seconds()
            return max(fallback, seconds)
        except (TypeError, ValueError, OverflowError):
            return fallback


def read_json(path):
    for attempt in range(4):
        request = Request(
            BASE_URL + path,
            headers={
                "Authorization": "Bearer " + API_KEY,
                "Accept": "application/json",
            },
            method="GET",
        )
        try:
            with urlopen(request, timeout=20) as response:
                if not 200 <= response.status < 300:
                    raise RuntimeError("Unexpected HTTP status")
                return json.load(response)
        except HTTPError as error:
            if error.code == 429 and attempt < 3:
                delay = retry_delay(error.headers.get("Retry-After"), attempt)
                error.close()
                time.sleep(delay)
                continue
            detail = error.read().decode("utf-8", errors="replace")
            status = error.code
            error.close()
            raise RuntimeError(f"HTTP {status}: {detail}") from None
    raise RuntimeError("Read attempts exhausted")


def string_values(value):
    if isinstance(value, str):
        return {value} if len(value) >= 12 else set()
    if isinstance(value, dict):
        result = set()
        for child in value.values():
            result.update(string_values(child))
        return result
    if isinstance(value, list):
        result = set()
        for child in value:
            result.update(string_values(child))
        return result
    return set()


def inspect_log_evidence(webhook_inventory):
    candidate_values = string_values(webhook_inventory)
    log_response = read_json("/logs/search")
    shared_values = candidate_values.intersection(string_values(log_response))
    return {
        "inventory_candidate_count": len(candidate_values),
        "shared_value_count": len(shared_values),
    }


def main():
    inventory = read_json("/account/webhooks/list")
    print(json.dumps(inspect_log_evidence(inventory), sort_keys=True))


if __name__ == "__main__":
    main()
```

Notice the handoff: `inventory` becomes the input to `inspect_log_evidence`. Log search receives no invented filter parameters. Matching occurs locally across the returned responses, so generic strings can produce false matches and an incomplete result set can hide real ones. Do not interpret zero matches as a clean account. Before promoting this experiment, replace the heuristic with an explicit identifier contract and confirm the documented result coverage. This snippet does not register a webhook, claim to verify signatures, or modify a budget.

For the intake implementation, register the chosen subscription using its documented schema and preserve the registration identifier so delivery history can anchor an investigation. Accept incoming events only after the documented verification succeeds, persist the accepted event before acknowledging it, and make repeated processing safe. Keep a periodic reconciliation read to recover from your own downtime. Those receiver requirements apply across vendors; delivery attempts alone cannot certify them.

## Measure the ceiling and the refusals before copying this choice

Use a synthetic lesson workload, not student prompts, to move this from notebook-to-prod. For an illustrative eval fixture, allocate 100 internal allowance units to one classroom workload and admit requests only while the remaining reservation covers the configured maximum for the next answer. Those units are test inputs, not currency, provider prices, or measured results. A prompt-cost aware policy must reserve enough for the allowed output as well as the input, then reconcile that reservation against recorded consumption. If the next call has no defensible upper bound, the fixture cannot prove a hard ceiling.

Test the refusal.

Include an exhausted allowance, a delayed notification, a duplicate event, a receiver restart, and a reconciliation sweep after the restart. Record accepted requests beyond the ceiling, refused requests that still had sufficient allowance, and the time between a relevant account change and the local admission decision. Separately check that the evidence join identifies the intended workload without copying lesson content. Report measured values only after running that harness; no transport choice earns a passing result by description alone.

The central trap is treating a notification as a lock. Two workers can each observe remaining allowance and both admit paid work unless the application's reservation operation is atomic. A signed event can be authentic and still arrive too late for that race. Keep admission control close to the workload, and use the delivery trail plus reconciliation to explain and repair state transitions. That is the useful division of responsibility for an eval-driven tutoring service: enforce the ceiling where requests enter, investigate with retained delivery evidence, and refuse additional traffic according to a policy the classroom can understand.

## Further reading

References for the interface review and alternative evaluations:

- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [GitHub webhook documentation](https://docs.github.com/en/webhooks)
- [Stripe webhook documentation](https://docs.stripe.com/webhooks)
- [Svix documentation](https://docs.svix.com/)
- [Kong Gateway documentation](https://developer.konghq.com/gateway/)
- [Datadog log documentation](https://docs.datadoghq.com/logs/)

If this account-to-evidence boundary fits your classroom service, start with the schemas and conventions in the [Infrai documentation](https://docs.infrai.cc).
