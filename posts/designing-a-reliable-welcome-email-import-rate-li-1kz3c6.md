# Designing a Reliable Welcome Email Import: Rate-Limited Transactional Batch Sends

A user import can create demand faster than a transactional email system can safely accept it. **Short answer: put every welcome email into a durable queue, drain it with a bounded worker pool, honor the sender's rate limit, and retry only transient failures with jitter.** Keep import completion separate from delivery completion, and make the send operation idempotent.

This is the same operational shape as sending a media order receipt after payment settles. A payment event or imported row creates an email job; it should never trigger an unbounded burst of network calls. The simple approach is a loop that sends immediately and retries any exception. It looks fine in a notebook. In production, it couples throughput to an external service, repeats permanent failures, and makes a partial batch hard to resume.

The chosen design has one boring property that matters: every recipient has an inspectable state.

## How should a user import batch send bulk transactional welcome email with rate limits?

Treat the import as a producer, not as the sender. Validate and normalize each row, assign an immutable message key, store the intended template version and recipient, then enqueue a reference to that record. A separate delivery worker claims jobs in bounded batches. The rate limiter governs when attempts begin; concurrency governs how many attempts can be in flight. Those are different controls, and a system often needs both.

The message key can be derived from stable business identity, such as `welcome:user_123:template_7`. For the media receipt variant, it might be `receipt:order_456:settled`. A uniqueness constraint on that key prevents two import runs or two payment-settled events from creating two logical sends. The transport request should carry the same key if the selected transport offers idempotency; regardless, the application database remains the record of intent and attempt history. Don't mark an imported user as "welcomed" before delivery has been accepted. Record states such as `pending`, `in_flight`, `accepted`, and `terminal_failure`, plus the attempt count and next eligible time. A worker lease needs an expiry so another worker can reclaim a job after process termination. Now consider the awkward case: the worker sends `receipt:order_456:settled`, loses its connection before recording the response, and restarts. The database still says `in_flight`, but blindly issuing a new logical message risks a duplicate receipt. The expired lease should make that job eligible for reconciliation under the same key, while the attempt history preserves the uncertainty for an operator. This is ordinary job processing, but it prevents email-specific code from swallowing the hardest failure mode: an unknown result after the request left the process. Google's sender guidelines belong in the acceptance criteria, not in a launch-week cleanup list. Authentication, subscription handling, message format, and sender reputation affect whether an accepted request becomes useful mail. The exact requirements vary by sending profile and can change, so verify the current guideline before deployment rather than freezing a remembered threshold in code.

Queue first.

| Approach | Integration effort | Best fit | Main limitation |
| --- | --- | --- | --- |
| Immediate loop | Low | Tiny, manually recoverable sends | Import latency and delivery latency are coupled |
| Bounded synchronous batch | Moderate | Small batches with explicit partial-failure handling | The caller remains responsible for retries and resumption |
| Durable queue and workers | Higher | Bursty imports and payment-triggered receipts | Requires job storage, leases, and operational monitoring |

## A focused bounded-worker example

The following Python sketch isolates policy from transport. `send_message` is an injected adapter, so the same worker can be exercised with a deterministic fake in an eval harness and connected to any standards-compliant delivery service later. The token bucket controls starts per second, while the semaphore caps simultaneous requests.

```python
import asyncio
import random
import time
from dataclasses import dataclass
from typing import Awaitable, Callable


@dataclass(frozen=True)
class EmailJob:
    message_key: str
    recipient: str
    template_version: str


@dataclass(frozen=True)
class SendResult:
    accepted: bool
    retryable: bool
    retry_after_seconds: float | None = None


class TokenBucket:
    def __init__(self, starts_per_second: float, capacity: int) -> None:
        self.rate = starts_per_second
        self.capacity = float(capacity)
        self.tokens = float(capacity)
        self.updated_at = time.monotonic()
        self.lock = asyncio.Lock()

    async def acquire(self) -> None:
        while True:
            async with self.lock:
                now = time.monotonic()
                elapsed = now - self.updated_at
                self.tokens = min(self.capacity, self.tokens + elapsed * self.rate)
                self.updated_at = now
                if self.tokens >= 1:
                    self.tokens -= 1
                    return
                wait_seconds = (1 - self.tokens) / self.rate
            await asyncio.sleep(wait_seconds)


async def deliver(
    job: EmailJob,
    send_message: Callable[[EmailJob], Awaitable[SendResult]],
    limiter: TokenBucket,
    concurrency: asyncio.Semaphore,
    max_attempts: int = 5,
) -> SendResult:
    for attempt in range(1, max_attempts + 1):
        await limiter.acquire()
        async with concurrency:
            result = await send_message(job)

        if result.accepted or not result.retryable or attempt == max_attempts:
            return result

        base_delay = min(30.0, 2 ** (attempt - 1))
        delay = result.retry_after_seconds
        if delay is None:
            delay = random.uniform(0.0, base_delay)
        await asyncio.sleep(delay)

    raise AssertionError("the attempt loop must return")
```

This function deliberately doesn't own database state transitions. The queue consumer should claim a lease, call `deliver`, and commit the resulting state in its own persistence boundary. Sleeping inside a worker is acceptable for a small batch, but a large backlog should store `next_attempt_at` and release the worker slot; otherwise delayed jobs consume capacity while doing no work. That distinction is easy to miss when a notebook test has ten recipients and no competing traffic.

The retry classification also belongs in the adapter, where transport responses can be mapped without leaking vendor details into job logic. Throttling and temporary network failures are candidates for retry. Invalid recipients, malformed requests, and rejected policy decisions are terminal until input or configuration changes. I'm not sure what retry window is right for every publication because the business value decays differently: a welcome message may tolerate a delay that an order receipt should surface to operations much sooner. An explicit delivery objective resolves that choice.

## Retries need budgets, not optimism

Exponential backoff is only half of retry design. Add jitter so a fleet doesn't wake together, enforce a maximum attempt count or time window, and cap retry traffic as a share of available capacity. Otherwise an unhealthy dependency can turn yesterday's backlog into today's load.

Be strict here.

The useful unit for idempotency is the logical message, not an individual HTTP attempt. Persist the message key before enqueueing, preserve it across retries, and never generate a fresh key merely because a worker restarted. If the outcome of an attempt is unknown, reconciliation is safer than an immediate duplicate send: inspect the application's state and, where the transport supports it, query by the same idempotency key.

Batch progress should remain monotonic even when individual rows fail. One bad address must not roll back 9,999 valid enqueue operations, while one database transaction per row may be unnecessarily expensive. Commit manageable chunks, capture row-level validation errors, and publish a final import summary containing counts by state. The number `9,999` is an illustration of partial progress, not a throughput claim; actual chunk size belongs to a load test against your database and queue.

## What should the eval harness measure before rollout?

Start with behavior, not provider response time. A deterministic transport fake can return accepted, retryable, and terminal results in a scripted order. With a fake clock and seeded randomness, tests can assert that a retry never exceeds the budget, a terminal result is attempted once, two jobs with the same message key produce one logical record, and concurrent workers don't exceed the configured start rate.

Then run a staging batch using non-production recipients and inspect four operational signals: queue age, attempts per outcome, acceptance latency, and terminal-failure reason. Queue depth alone is misleading because a growing queue may be expected during import; the age of the oldest eligible job tells you whether workers are actually falling behind. Keep message content and addresses out of general-purpose logs. Correlate with the opaque message key instead.

Prompt-cost awareness still matters in an AI-assisted media workflow. If a model generates subject lines or personalized copy, generate and approve that artifact before the delivery queue, pin the prompt and model configuration with the template version, and cache the result on the message record. A retry should resend the exact approved payload rather than spend tokens again or produce different wording. Evaluate content quality separately from delivery reliability; combining them makes a transport regression look like a prompt regression and vice versa.

For an order receipt, add a business invariant: only a settled payment can create the receipt job, and replaying the settlement event must resolve to the existing message key. For a welcome campaign created from an import, record consent provenance and suppression status before enqueueing. These checks sit upstream of rate limiting because a perfectly paced message can still be the wrong message.

## The decision boundary

**Use the queue-and-worker design when delivery is asynchronous, bursts are plausible, and duplicate or lost messages carry real support cost.** It gives the team replay, visibility, controlled throughput, and a clean place to enforce retry policy without tying import or payment latency to an email transport.

The catch is added infrastructure and delayed completion. It is not suitable when the application must synchronously confirm delivery rather than merely accept a job, and it may be too much machinery for a tiny internal tool with a handful of manually recoverable messages. In that case, stick with a bounded synchronous sender, retain stable message keys, and report partial failures explicitly. If an existing job system already provides leases, delayed retries, and uniqueness, use those primitives instead of building an email-specific queue.

SMS is not a drop-in retry channel. Consent, content, and cost semantics differ, and message encoding affects segmentation: the referenced SMS guidance explains the distinct GSM-7 and UCS-2 limits. Choose SMS as a separately designed notification path, not as an automatic response to an email failure.

Before copying this choice, measure peak enqueue rate, sustainable accepted-send rate, oldest-job age under retries, duplicate logical messages, and terminal-failure volume. Those numbers determine worker count, rate policy, and alert thresholds. Your mileage may vary, especially when imports and payment receipts share capacity, but the evaluation method stays stable.

## Sources

- https://support.google.com/a/answer/81126
- https://www.twilio.com/docs/glossary/what-sms-character-limit
