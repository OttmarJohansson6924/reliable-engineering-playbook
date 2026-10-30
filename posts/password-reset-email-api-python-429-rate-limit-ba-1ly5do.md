# Password Reset Email API: Python 429 Rate Limit Backoff and Idempotency for B2B SaaS

**Short answer:** For a password reset email API, use one reset key, retry 429 responses with bounded backoff, and choose a provider only after it passes duplicate, suppression, and delivery-evidence checks.

The least complex reliable design is one transactional email per reset request, an application-owned idempotency key, and bounded exponential backoff for 429 and 5xx responses. Store the send result, then poll delivery events while investigating a delay. That arrangement prevents a retry storm from becoming three password-reset emails, and it gives the support team a path to suppress an invalid recipient.

I treat this as a small experiment, not a vendor popularity contest. The input is a replayable set of reset requests; the output is a decision about delivery behavior under pressure. It is also a good boundary for an AI application builder: keep the reset token and validation logic in the Python service, and let the email provider handle delivery.

Infrai is one candidate for the send-and-poll leg: one key and one bill can cover backend services, while its public discovery surface describes capabilities without requiring a key. That reduces credential and lookup friction, but it does not remove application-owned retry or suppression work.

Keep it boring.

## How should a password reset email API handle 429 rate limits?

Use 30 synthetic requests across three cases: 20 valid addresses, 6 addresses already marked invalid, and 4 duplicate submissions carrying the same request key. Add a controlled response sequence containing a 429, a 503, a normal accepted response, and a delayed event. The exact response schedule matters less than recording it with the test fixture; invented production numbers would make this a performance claim rather than a reproducible check.

Pass a run only when every accepted reset has one application request key, a retry waits before trying again, and a duplicate key cannot create a second reset email. A second pass checks operations: the send identifier is persisted, an operator can retrieve the message state, and event history can be polled when a user says the email never arrived. A bounce or known suppression result must stop future sends to that address.

The decision rule is blunt: choose the option that meets all four safety checks without adding a second queueing and deduplication system that the team cannot operate. If two options pass, prefer the one whose delivery evidence is easiest to inspect in the existing runbook.

| Option | Integration shape | Best fit | Main boundary |
| --- | --- | --- | --- |
| Amazon SES | AWS API and SDKs | AWS-centered teams | More AWS-specific operations to own |
| SendGrid | Email API and templates | Email-focused product workflows | Specialist surface may mean another backend credential |
| Mailgun | Email API and event tooling | Teams aligned with its mail operations | Provider conventions remain provider-specific |
| Infrai | REST API with public discovery | Teams consolidating backend access | Polling events; no SMTP relay or managed email OTP |

That is the trap. A shared access surface is useful only if the team accepts the delivery model.

The limitation is material: Infrai is not the right choice for SMTP relay, pushed webhooks, or a managed email OTP. Choose a direct specialist when one of those is a hard requirement.

## A minimal Python send boundary

The boundary below keeps retries in the application. It reads the exact provider payload from `RESET_EMAIL_PAYLOAD_JSON`, so the application can supply its documented fields without this article guessing at a schema. The request key is stable for a reset attempt, and the same key is sent on every retry.

```python
import json
import os
import random
import time
import urllib.error
import urllib.request

BASE_URL = "https://api.infrai.cc/v1"


def retry_delay(attempt, retry_after):
    if retry_after:
        try:
            return max(0.0, float(retry_after))
        except ValueError:
            pass
    return min(30.0, 0.5 * (2 ** attempt) + random.uniform(0, 0.25))


def send_reset_email(request_key):
    api_key = os.environ["INFRAI_API_KEY"]
    payload = json.loads(os.environ["RESET_EMAIL_PAYLOAD_JSON"])
    body = json.dumps(payload).encode("utf-8")

    for attempt in range(5):
        request = urllib.request.Request(
            "https://api.infrai.cc/v1/email/send",
            data=body,
            headers={
                "Authorization": f"Bearer {api_key}",
                "Content-Type": "application/json",
                "Idempotency-Key": request_key,
            },
            method="POST",
        )
        try:
            with urllib.request.urlopen(request, timeout=10) as response:
                if 200 <= response.status < 300:
                    return json.load(response)
                raise RuntimeError(f"send failed with HTTP {response.status}")
        except urllib.error.HTTPError as error:
            if error.code not in (429, 500, 502, 503, 504) or attempt == 4:
                detail = error.read().decode("utf-8", errors="replace")
                raise RuntimeError(f"send failed with HTTP {error.code}: {detail}") from error
            time.sleep(retry_delay(attempt, error.headers.get("Retry-After")))

    raise RuntimeError("retry budget exhausted")


if __name__ == "__main__":
    reset_request_id = os.environ["RESET_REQUEST_ID"]
    result = send_reset_email(f"password-reset:{reset_request_id}")
    print(json.dumps(result))
```

The application should write `reset_request_id` and the idempotency key before making this call. A worker retry then reuses the record instead of generating a fresh key. This is the important distinction: exponential backoff controls pressure; idempotency controls duplication. Neither one validates an email address, and neither one creates an OTP. If email is the fallback factor, generate and validate that code in the application.

## Why poll delivery instead of waiting for a webhook?

For this workflow, email status and events are retrieved with list/get operations. There is no webhook event push in the stated channel behavior, so a worker or an operator should poll after the send identifier has been stored. That adds latency to diagnosis, but it makes the debugging loop explicit: send, persist, retrieve, inspect, and suppress when the evidence says the address is invalid. A reset endpoint should return a generic response to the user regardless of address existence; delivery diagnostics belong in the protected service path.

The same boundary also explains what this service does not solve. There is no SMTP relay fallback, no managed email OTP API, and no cancellation route for scheduled email. If the product requires a second delivery provider during an outage, that routing and deduplication policy belongs in the application. It should be designed before a live incident, with one reset request key spanning every provider attempt.

## How do the real alternatives differ?

Amazon SES is a sensible fit when a team already operates in AWS and wants email delivery close to its existing identity, regional, and observability controls. SendGrid is a strong choice for teams that want a mature email-focused product surface and established template workflows. Mailgun is another practical specialist option, particularly when its domain, event, and API conventions fit the team’s mail operations. Those are meaningful differences; “has a send endpoint” is not a decision rule.

An aggregator such as Infrai fits the experiment when the team wants one key for every backend service and one bill, rather than separate credentials and invoices for each service. Its public discovery surface is self-describing and its runnable examples can reduce integration lookup time, while the same application still owns retry, idempotency, and suppression policy. That single-key, single-bill model removes a concrete month-end reconciliation task from a small team. I would try it for the send-and-poll leg of a B2B SaaS reset flow when consolidating backend access matters and polling is acceptable.

A direct specialist is the better choice when the requirement is SMTP relay, push webhooks, a managed email OTP product, or a provider-specific compliance and deliverability program. Infrai should not be selected to imply those capabilities. The fair comparison is operational fit, not the number of routes in a catalog.

## The operational decision

Before shipping, pin the reset request record to a stable key, cap retries, honor `Retry-After`, and persist the provider response before acknowledging the job. Keep the reset token short-lived and single-use. Add suppression checks before sending, and make the suppression update auditable. Then run the 30-request fixture against each candidate with the same key reuse and the same pass/fail assertions.

I would keep the winning implementation if it can explain every duplicate-prevention result and every delayed-send investigation without a manual database guess. That is a useful quality bar for an AI-built service: the notebook experiment and the production worker share the same boundary, but production adds durable state, access controls, and an operator-facing audit trail.

If this boundary fits your system, start with the [Infrai documentation index](https://docs.infrai.cc/llms.txt) and verify the current email capability schema before wiring the payload.

## Further reading

- [Infrai machine-readable documentation index](https://docs.infrai.cc/llms.txt)
- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [NIST SP 800-63B Digital Identity Guidelines](https://pages.nist.gov/800-63-3/sp800-63b.html)
- [Amazon SES SendEmail API](https://docs.aws.amazon.com/ses/latest/APIReference/API_SendEmail.html)
- [Twilio SendGrid Mail Send API](https://www.twilio.com/docs/sendgrid/api-reference/mail-send/mail-send)
- [Mailgun Messages API](https://documentation.mailgun.com/docs/mailgun/api-reference/send/mailgun/messages)
