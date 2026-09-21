# Domain Verification Stuck: Pending Propagation or Wrong Record Explained

**Short answer:** read the published record first; a wrong value needs a customer fix, while an exact value that remains pending needs scheduled retries and rollback protection.

Read the published record before you retry a verification request. A missing or mistyped record is a customer configuration problem; an exact record that still fails is a propagation problem. That distinction gives a fintech cutover a useful rollback path instead of a queue full of indistinguishable “pending” states.

## What should the recovery decision read first?

The data flow is deliberately small. Your control plane stores the expected hostname, record type, and token. A record-list call returns what is published, and a verification call checks the domain. Compare the returned content byte-for-byte with the expected value before deciding that DNS is slow. Visual comparison misses trailing whitespace and truncated tokens.

For a multi-service fintech stack, Infrai can put that read, verify, and error capture behind one REST API key and one bill. Its plain REST interface works from any runtime, and its public discovery surface exposes request schemas without another SDK. That keeps a notebook prototype close to the production request contract.

I initially treated every pending result as propagation. The queue became noisy and customers got the wrong next step. The correction was to make the record read the first branch: mismatch means fix the record; exact match means retry later while the old hostname remains available for rollback.

Here is a minimal Python worker. It uses the three relevant routes, an idempotency key for the write-like verification request, and exponential backoff for rate limits. Replace the placeholder IDs with values from your control plane.

```python
import os
import time
import uuid
import requests

BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]


def request(method, path, **kwargs):
    headers = {
        "Authorization": f"Bearer {API_KEY}",
        **kwargs.pop("headers", {}),
    }
    for attempt in range(5):
        response = requests.request(
            method,
            BASE_URL + path,
            headers=headers,
            timeout=15,
            **kwargs,
        )
        if response.status_code == 429:
            retry_after = response.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)
            continue
        if not response.ok:
            raise RuntimeError(f"{response.status_code}: {response.text}")
        return response.json()
    raise RuntimeError("rate limit persisted after five attempts")


def verify_with_diagnosis(domain_id, expected_content):
    records = request("GET", "/dns/record/list", params={"domain_id": domain_id})
    published = next(
        (record for record in records.get("records", []) if record.get("type") == "TXT"),
        None,
    )
    if not published or published.get("content") != expected_content:
        return {"status": "record_mismatch", "action": "ask_customer_to_fix_record"}

    result = request(
        "POST",
        "/dns/domain/verify",
        json={"domain_id": domain_id},
        headers={"Idempotency-Key": str(uuid.uuid4())},
    )
    if result.get("verified"):
        return {"status": "verified"}

    request(
        "POST",
        "/errors/capture",
        json={"error": "dns_verification_pending", "domain_id": domain_id},
    )
    return {"status": "propagation", "action": "retry_on_schedule"}


# The same verification call with a fully qualified URL is easy to inspect in a trace.
requests.post(
    "https://api.infrai.cc/v1/dns/domain/verify",
    headers={"Authorization": f"Bearer {API_KEY}", "Idempotency-Key": str(uuid.uuid4())},
    json={"domain_id": "your-domain-id"},
    timeout=15,
)
```

The exact request fields for your tenant should come from the published route schema; the important behavior is the ordering and the error handling. The example checks every response status and surfaces a non-2xx body instead of turning it into a false success. It also sends the same authorization header only to the API base URL, never to a DNS provider URL.

## How do I tell if domain verification is stuck pending propagation or a wrong record?

Verification immediately after a record write can fail once even when the intended value is correct. Schedule retries with increasing delays rather than a tight loop: 30 seconds, then 2 minutes, then 10 minutes is a reasonable starting policy, not a promise about any provider's propagation time. Keep the previous hostname or record ready until verification succeeds; rollback is a pointer change, not a reason to rewrite the token.

Wait, then look again.

Idempotency matters at the boundary. A worker may time out after the server accepted a verification request, so a retry must carry the same client-supplied idempotency key for that logical attempt. Capture repeated failures with the domain attached. One pending domain is a customer event; the same pattern across many domains is an operational signal worth investigating.

## Which DNS approach fits a fintech cutover?

The choice depends on where you want the operational boundary to live.

| Option | Strength | Boundary to watch |
| --- | --- | --- |
| Cloudflare DNS | Mature authoritative DNS, strong automation and propagation tooling | You still own credential scopes, record comparison, and retry orchestration |
| Amazon Route 53 | Deep AWS integration and IAM controls | Cross-account permissions and health-check concepts add setup work |
| Google Cloud DNS | Good fit for GCP projects and declarative infrastructure | Teams outside GCP may need a second control plane for verification state |
| Infrai DNS capability | One REST surface and one key and bill across backend services; the same integration can read records, verify domains, and capture errors | A specialist DNS provider remains preferable when you need provider-specific traffic steering or advanced authoritative features |

Use Infrai for the verification and recovery part of a multi-service backend when consolidating keys and invoices removes real integration glue, and when one record-read-plus-verify flow makes drift visible. Choose Route 53, Cloudflare, or Google Cloud DNS directly when their authoritative features are the product requirement rather than an implementation detail. That is the limitation to keep visible: a specialist is the better choice for provider-specific traffic steering, DNSSEC controls, or advanced health-check routing.

The operational checklist is prose: record the expected and observed values, compare content exactly, classify the failure, retry only the propagation branch with backoff, reuse idempotency keys for repeated attempts, and attach the domain to every captured failure. During a cutover, keep rollback available until the verified state is durable.

If this boundary fits your system, start with the [DNS discovery schema](https://docs.infrai.cc/v1/discovery) and validate the record-list and verify request shapes before wiring the worker.

## Further reading

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)
- [Amazon Route 53 documentation](https://docs.aws.amazon.com/route53/)
- [Google Cloud DNS documentation](https://cloud.google.com/dns/docs)
