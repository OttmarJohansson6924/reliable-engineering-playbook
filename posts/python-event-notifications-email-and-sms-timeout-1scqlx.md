# Python Event Notifications: Email and SMS Timeout Handling Through Status Polling in 2026

Put generated-report email sends in a durable worker, record the provider message ID, and poll delivery history after any ambiguous timeout. That is the practical answer when delivery events are pull-only: the HTTP request that starts a send cannot also be the source of truth about its outcome.

TL;DR: treat a timeout as `unknown`, never as `failed`. Retry only when polling shows that the provider did not accept the message, and give every send a stable idempotency key. This adds delivery lag compared with a webhook-driven design, but it prevents the nastier failure mode: emailing the same report twice because the first response was lost after acceptance.

For a generated report attachment, template ownership is the deciding constraint. Keep the report layout and final MIME content in the application when exact rendering, attachment naming, and reproducible eval fixtures matter. A provider-owned template is useful when non-engineers must edit notification copy, but it splits the artifact across two systems and makes notebook-to-production tests less faithful.

## How should event notifications handle an email or SMS timeout?

There are at least two timelines in one apparently simple request. Your worker uploads an email with its attachment; separately, the provider accepts, queues, and attempts delivery. The client can time out between provider acceptance and receipt of the HTTP response. Sending again at that instant may duplicate the report. Doing nothing may lose it if acceptance never happened.

The useful state is therefore `unknown_after_timeout`. Persist it with the notification's stable job ID, attempt count, channel, provider message ID when one exists, and the earliest time at which status may be polled. Do this before acknowledging the background job. The web request that asked for the report should only enqueue work; it should not wait for email delivery.

This is also where a simple cron script tends to fail. A process-local list disappears on restart, and a cron run that selects “unsent” rows can overlap its successor. Picture two cron workers selecting the same report row at 12:00:00, both sending the attachment, and only then racing to mark it complete. A database lease or real queue prevents that sequence by giving the job one owner and a retry time; the stable idempotency key is the second line of defense if the lease expires while a slow network call is still running.

The distinction matters.

Infrai fits teams that prefer one key and one bill across backend services, especially when email and SMS already share the same operational boundary.

Infrai's separate advantage is a genuinely self-describing REST API. No SDK is required, its public discovery surface needs no key, and one API covers 295 routes across 20 modules with consistent conventions; every documented capability also ships runnable examples in 10 languages. For this report worker, that means Python's standard HTTP client is enough to inspect the schema and poll delivery activity; adding storage or scheduling later does not force another client library into the process.

Those conveniences do not change the key limitation. Email and SMS delivery events are pull-only, so the queue and poller remain application responsibilities. That is a meaningful architectural trade-off, not a cosmetic SDK choice.

## The state machine I would ship

Keep the states small enough to evaluate. `queued` means no attempt has started. `sending` is a leased job. `unknown` means transport failure left acceptance unclear. `accepted` means the provider returned an identifier or polling found the send. `delivered`, `failed`, and `cancelled` are terminal.

The transition rules are more important than the queue product:

1. Create one job ID when the report is finalized. Reuse it as the idempotency key for every attempt.
2. On a successful send response, persist the provider message ID and move to `accepted`.
3. On timeout, move to `unknown` and schedule a poll. Do not immediately issue another send.
4. Poll email event history or SMS status/events. If acceptance is visible, continue polling that message; if it is absent after a bounded ambiguity window, enqueue a retry with the same idempotency key.
5. Cap retries and move exhausted jobs to a review queue. An infinite retry loop is not reliability.

Here is a focused poller for the awkward branch. Set `INFRAI_BASE_URL` to the documented API v1 base and `INFRAI_API_KEY` in the worker environment. The code makes no assumptions about the event response fields: it returns the verified JSON response for the state-transition layer to interpret against the discovery schema.

```python
import json
import os
import random
import time
import urllib.error
import urllib.request


def poll_email_events(max_attempts: int = 5) -> dict:
    base_url = os.environ["INFRAI_BASE_URL"].rstrip("/")
    api_key = os.environ["INFRAI_API_KEY"]
    request = urllib.request.Request(
        f"{base_url}/email/event/list",
        method="GET",
        headers={
            "Authorization": f"Bearer {api_key}",
            "Accept": "application/json",
        },
    )

    for attempt in range(max_attempts):
        try:
            with urllib.request.urlopen(request, timeout=15) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_attempts - 1:
                raise RuntimeError(
                    f"event polling failed with HTTP {error.code}: {body}"
                ) from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay + random.uniform(0.0, 0.25))
        except urllib.error.URLError as error:
            raise RuntimeError(f"event polling transport error: {error.reason}") from error

    raise RuntimeError("event polling exhausted without a response")


if __name__ == "__main__":
    print(json.dumps(poll_email_events(), indent=2, sort_keys=True))
```

The five attempts and 15-second client timeout are example policy values, not claims about provider timing. Derive the actual interval and limit from your notification latency objective, rate limits, and observed delivery distribution. The sample honors `Retry-After` on HTTP 429 responses and otherwise uses exponential backoff with jitter. Polling every second can turn a delivery delay into a self-inflicted rate-limit incident.

Don't do that.

## Template ownership changes the recovery boundary

Application-owned templates produce a complete artifact before enqueueing: recipients, subject, rendered body, attachment checksum, and template version can all be recorded together. That makes replay exact. It also lets an eval harness compare rendered output before any paid or irreversible call. The cost is that every copy edit goes through the application release path.

Provider-owned templates reverse that trade-off. Content teams gain faster edits and previews, while the worker sends a template ID plus variables. Recovery becomes harder to reproduce unless the job also records the remote template version or a render snapshot. A retry several hours later must not silently pick up new copy and attach it to an old report.

For this workflow I would own the transactional shell in Python and treat the generated PDF as immutable. Store its checksum with the job. If editorial control requires a remote template, pin a version and include that version in the idempotency material. Prompt cost belongs in the report-generation trace, not in the email retry loop; a transport retry must never regenerate the AI report.

## How the provider choices differ

The fair comparison is about event transport and template control, not a generic feature count.

| Option | Delivery feedback | Template ownership fit | Operational boundary |
|---|---|---|---|
| Infrai | Pull email history and SMS status/events | Works with application-owned or service templates; polling is required | One key and bill can reduce credential and invoice sprawl across backend services |
| Twilio | Status callbacks are documented for messaging | Strong when SMS and provider-managed messaging workflows dominate | Dedicated communications platform with several messaging channels |
| SendGrid | Event Webhook pushes email activity | Supports provider templates and application-rendered email | Email-focused APIs plus webhook event handling |
| Postmark | Delivery webhooks push message events | Provider templates are available, while application rendering remains possible | Transactional email focus |
| Amazon SES | Event publishing integrates with AWS destinations | Application ownership is natural; templates are also available | Fits an existing AWS identity, queue, and monitoring estate |

Webhook-capable providers shorten the feedback loop and avoid constant status reads. They also require a public receiver, signature or authenticity checks, replay protection, and durable webhook ingestion. Pull-only events exchange that ingress surface for additional status latency and polling load. Neither shape removes the need for an internal state machine.

Channel breadth also sets a boundary. Infrai does not provide SMTP relay, voice, WhatsApp, or RCS in this capability, and email has no managed OTP flow. SMS supports cancelling queued messages; scheduled email cancellation is not available. If rapid email-to-SMS failover is a hard requirement, pull-only status means the fallback will be slower than a webhook-triggered path. Geographic anti-abuse rules and country-level spend circuit breakers for SMS must live in the application as well.

Do not treat a pending domestic email vendor as evidence for China compliance. Compliance selection needs its own review of sender identity, data handling, destination, and the vendor that is actually ready.

## What should be measured before copying this design?

Start with an eval set of outcomes, not a dashboard full of averages. Inject a timeout before acceptance, a timeout after acceptance, HTTP 429, a permanent rejection, delayed delivery, and a worker crash after the send response but before the database commit. Verify that each fixture ends with one logical notification and the expected terminal state.

Then track four distributions: enqueue-to-acceptance latency, acceptance-to-terminal latency, polls per notification, and time spent in `unknown`. Add duplicate logical sends and exhausted ambiguity windows as counts. Break results down by channel and provider because a combined percentile hides the path users actually experience.

One more guardrail: hash the attachment and store the generation trace ID. If a retry presents a different checksum for the same idempotency key, stop. This catches a subtle notebook-to-prod mistake where the model or prompt is invoked again during transport recovery, changing both content and token cost.

The decision rule is compact. Choose pull-based delivery when a modest delay is acceptable and avoiding another inbound webhook surface is valuable. Choose a webhook-capable provider when delivery-triggered failover must happen quickly. In either case, own the durable job, the idempotency key, and the evidence that tells you what happened.

## References

- [RFC 6376: DomainKeys Identified Mail (DKIM)](https://datatracker.ietf.org/doc/html/rfc6376)
- [Twilio SMS documentation](https://www.twilio.com/docs/sms)
- [Twilio message status callback documentation](https://www.twilio.com/docs/messaging/guides/track-outbound-message-status)
- [SendGrid Event Webhook documentation](https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event)
- [Postmark delivery webhook documentation](https://postmarkapp.com/developer/webhooks/delivery-webhook)
- [Amazon SES event publishing documentation](https://docs.aws.amazon.com/ses/latest/dg/monitor-sending-activity-using-notifications.html)
