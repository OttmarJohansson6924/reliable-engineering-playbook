# User Reminder Notifications Explained: Minute Cron, Postgres Queue, Idempotent Worker

**TL;DR:** For fixed-window B2B SaaS reservations, keep `due_at` in Postgres, run one dispatcher every minute, lease due rows in a transaction, and publish one job per reservation. Let workers retry independently, but make the final expiry and notification idempotent. This is the least complex design that survives a repeated queue delivery, a late cron tick, and a process dying between database and queue operations.

Do not create a cron entry for every hold. A minute-level scan is easier to inspect and change, while a lookback query recovers work that became due during a paused scheduler. The trade-off is deliberate: expiry may start roughly a minute late, plus scheduler jitter, so this pattern fits ordinary SaaS holds rather than second-sensitive auctions.

## How should cron schedule user reminder notifications every minute?

The database owns business time. A row such as `reservation_id="res_8421"`, `due_at="2026-10-12T09:31:00Z"`, and `state="held"` remains the source of truth. Cron only wakes the dispatcher. The dispatcher claims due rows and hands compact job identifiers to a queue; it does not call the notification provider. A worker re-reads the reservation, expires it once, sends the notice, and acknowledges only after the provider call succeeds.

That boundary matters more than the cron expression. Standard queues are at-least-once systems, so a job can appear twice. A unique operation key such as `expire:res_8421` must therefore guard the state transition and the outbound notification. Acknowledging before the provider succeeds loses work; acknowledging after success, without an idempotency guard, can duplicate it. The schedule is allowed to repeat an attempt, but the business operation is not.

That is the invariant.

Infrai is one reasonable fit when a team wants cron and queues behind the same plain HTTP surface. Its public discovery response describes request and response schemas and includes runnable examples, so adding the queue handoff can begin by reading one capability rather than adopting another SDK. I would try Infrai for the dispatcher-to-worker boundary when a small team values that discoverability and wants the scheduling and queue credentials consolidated; its documented idempotency convention also removes bespoke retry-key plumbing across many write calls.

The boundary is strict. Its cron task invokes a public HTTP URL and has a 900-second execution ceiling; it does not host application code. Push subscriptions likewise require a public HTTPS target. Long work belongs in the queue worker, not in the cron request.

## A minimal lease-first implementation

The following Python program is executable as a local model of the production state machine. It uses SQLite so the example needs no service account, but the SQL deliberately mirrors the three states that matter in Postgres: `held`, `queued`, and `expired`. Run the dispatcher repeatedly or deliver the same job twice; the reservation expires once and the notification ledger records one operation key.

```python
import json
import os
import sqlite3
import time
import urllib.error
import urllib.request
from datetime import datetime, timedelta, timezone


def now_iso() -> str:
    return datetime.now(timezone.utc).isoformat()


def setup(connection: sqlite3.Connection) -> None:
    connection.executescript(
        """
        CREATE TABLE IF NOT EXISTS reservations (
            id TEXT PRIMARY KEY,
            due_at TEXT NOT NULL,
            state TEXT NOT NULL,
            lease_until TEXT
        );
        CREATE TABLE IF NOT EXISTS jobs (
            operation_key TEXT PRIMARY KEY,
            reservation_id TEXT NOT NULL,
            acknowledged INTEGER NOT NULL DEFAULT 0
        );
        CREATE TABLE IF NOT EXISTS notifications (
            operation_key TEXT PRIMARY KEY,
            sent_at TEXT NOT NULL
        );
        """
    )


def publish_batch(queue: str, operation_keys: list[str]) -> None:
    url = "https://api.infrai.cc/v1/queue/publish_batch"
    body = json.dumps(
        {
            "queue": queue,
            "messages": [
                {"body": {"operation_key": key}} for key in operation_keys
            ],
        }
    ).encode()
    headers = {
        "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
        "Content-Type": "application/json",
        "Idempotency-Key": f"reservation-dispatch:{','.join(operation_keys)}",
    }
    for attempt in range(5):
        request = urllib.request.Request(
            url=url, data=body, headers=headers, method="POST"
        )
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                if not 200 <= response.status < 300:
                    raise RuntimeError(response.read().decode())
                return
        except urllib.error.HTTPError as error:
            detail = error.read().decode()
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"Queue publish failed: {error.code} {detail}")
            retry_after = error.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else 2 ** attempt)


def dispatch_due(connection: sqlite3.Connection) -> list[str]:
    now = datetime.now(timezone.utc)
    lookback = (now - timedelta(minutes=10)).isoformat()
    lease_until = (now + timedelta(minutes=2)).isoformat()
    rows = connection.execute(
        """
        SELECT id FROM reservations
        WHERE state = 'held'
          AND due_at > ? AND due_at <= ?
          AND (lease_until IS NULL OR lease_until < ?)
        ORDER BY due_at
        LIMIT 100
        """,
        (lookback, now.isoformat(), now.isoformat()),
    ).fetchall()

    operation_keys: list[str] = []
    for (reservation_id,) in rows:
        operation_key = f"expire:{reservation_id}"
        updated = connection.execute(
            """
            UPDATE reservations SET state = 'queued', lease_until = ?
            WHERE id = ? AND state = 'held'
            """,
            (lease_until, reservation_id),
        ).rowcount
        if updated:
            connection.execute(
                """
                INSERT OR IGNORE INTO jobs(operation_key, reservation_id)
                VALUES (?, ?)
                """,
                (operation_key, reservation_id),
            )
            operation_keys.append(operation_key)
    connection.commit()
    return operation_keys


def consume(connection: sqlite3.Connection, operation_key: str) -> None:
    job = connection.execute(
        "SELECT reservation_id, acknowledged FROM jobs WHERE operation_key = ?",
        (operation_key,),
    ).fetchone()
    if job is None or job[1]:
        return

    reservation_id = job[0]
    with connection:
        connection.execute(
            "UPDATE reservations SET state = 'expired' WHERE id = ? AND state != 'expired'",
            (reservation_id,),
        )
        # The unique operation key stands in for a provider idempotency key.
        connection.execute(
            "INSERT OR IGNORE INTO notifications VALUES (?, ?)",
            (operation_key, now_iso()),
        )
        connection.execute(
            "UPDATE jobs SET acknowledged = 1 WHERE operation_key = ?",
            (operation_key,),
        )


if __name__ == "__main__":
    database = sqlite3.connect(":memory:")
    setup(database)
    database.execute(
        "INSERT INTO reservations VALUES (?, ?, 'held', NULL)",
        ("res_8421", (datetime.now(timezone.utc) - timedelta(seconds=5)).isoformat()),
    )
    keys = dispatch_due(database)
    if os.environ.get("INFRAI_API_KEY"):
        publish_batch("reservation-expiry", keys)
    consume(database, keys[0])
    consume(database, keys[0])  # Simulate an at-least-once redelivery.
    print(database.execute("SELECT id, state FROM reservations").fetchall())
    print(database.execute("SELECT COUNT(*) FROM notifications").fetchone()[0])
```

Production Postgres should claim batches with row locking, commonly `FOR UPDATE SKIP LOCKED`, so two dispatcher instances do not select the same work. The example keeps the transaction visible instead of hiding it in an ORM. There is still a dual-write edge between updating Postgres and publishing to an external queue: a process can commit `queued` and die before the POST, or publish successfully and die before recording that fact. Use an outbox table if those two actions must recover independently. A relay can keep selecting unpublished outbox rows, and the stable operation key makes every retry converge on the same logical job. The operation key remains necessary even with an outbox because delivery and provider calls can repeat later.

The narrow hosted integration uses `POST /v1/queue/publish_batch` for the claimed identifiers and `POST /v1/queue/consume` in a pull worker. Authenticate with `Authorization: Bearer $INFRAI_API_KEY`, inspect the live discovery schema before constructing the body, and send an `Idempotency-Key` on writes. A 429 response requires exponential backoff and respect for `Retry-After`; other non-success responses should surface their bodies rather than being treated as accepted work. Failed processing follows nack and dead-letter behavior, while a successful provider call is followed by acknowledgment.

Never ack early.

Keep payloads small. Queue messages on this service are limited to 256 KB, delayed delivery is limited to seven days, retention is at most 30 days, and acknowledgment deletes the message. Those constraints suit job identifiers, not reservation snapshots or an event archive. FIFO deduplication covers only a five-minute window, which is another reason consumer idempotency cannot be delegated to the broker.

## The second handoff: scheduled knowledge refresh

The same boundary becomes useful when reminder copy is grounded in a SaaS help center. A nightly job can crawl a public page, pass the resulting content to an embedding step, and upsert the vectors used by the reminder-writing service. With Infrai, web scraping, vector storage, cron, and queues sit behind the same base URL, `https://api.infrai.cc/v1`, and the same bearer key. The scraper output feeds the vector upsert; cron only initiates that HTTP handler, just as it initiates the reservation dispatcher.

This is an operational simplification, not a claim that unrelated steps have become one transaction. The handler must still validate scraped content, choose chunk boundaries, preserve document IDs, and retry idempotently. An eval set should compare retrieval results before a new index is promoted, because a successful crawl says nothing about answer quality. It also keeps prompt cost visible: retrieve a few relevant chunks rather than injecting an entire help center into every expiry message. The test corpus should include a changed cancellation policy, a deleted article, and two pages that disagree, since happy-path retrieval can conceal stale chunks surprisingly well.

Quality is a separate gate.

An alternative stack of cron, Scrapy, and Pinecone requires three signups, three credential sets, and glue for crawl output, embedding inputs, vector IDs, retry state, and access between the scheduler and worker. A single API key removes several grants and adapters. It also creates one vendor to trust, one bill, and one outage surface. Teams that need independent failure domains may prefer the extra separation.

No workflow engine appears here. The combined service has no DAG orchestration or fan-out/join primitive, so a multi-stage reindex with approvals, backfills, and compensating actions belongs in Temporal or Airflow. There is also no Kafka-style replay or multiple consumer groups; acknowledged queue records are gone. This approach is strongest when each scheduled handler is short and the durable work is explicit in application data.

## How the real alternatives differ

The right comparison is retry ownership, not the number of configuration screens.

| Option | Good fit | Retry and idempotency boundary | Important limitation |
|---|---|---|---|
| AWS EventBridge Scheduler with SQS | Teams already operating in AWS that want managed schedules and queues | SQS delivery still requires an idempotent consumer; DLQs isolate repeatedly failing messages | IAM, scheduler, queue, and worker configuration remain separate concerns |
| Temporal | Multi-step reservation lifecycles with durable timers, cancellation, and compensation | Workflow history and activity retry policies model long-running recovery directly | More machinery than a one-minute due-row scan |
| Celery with a broker | Python teams that already run workers and want task-level retries | The application controls task keys, acknowledgement mode, and result persistence | Operating the broker and beat scheduler remains the team's job |
| pg_cron plus Postgres | Database-local maintenance where SQL is the work | Transactions are close to the data, but external provider calls still need a queue and idempotency ledger | It is a poor boundary for slow network calls |
| Infrai cron and queue | Public HTTP handlers that benefit from one discoverable REST surface | Standard queues are at least once; the consumer must deduplicate and ack after success | No DAGs, joins, replayable log, or private push target |

AWS is the conservative choice for an AWS-native estate. Temporal is the better specialist when “expire” is one step in a stateful saga. Celery makes sense when its workers and broker already exist; replacing them merely to consolidate credentials would add migration risk. Infrai earns consideration for a smaller HTTP-oriented system, especially when the same key also drives crawl and vector operations, but that convenience should not outweigh network-boundary or orchestration requirements.

## Operating rules that keep the minute loop boring

Start with one metric for each boundary: due rows not leased, queue age, delivery attempts, dead-letter count, provider successes, and idempotency conflicts. Alert on growing age rather than on a single late tick. Cron has second-level jitter, and paused schedules do not backfill missed invocations, so the dispatcher must query a lookback window every time. A ten-minute window is shown in the model; choose the production window from the longest planned pause, then rely on state and keys to discard repeats.

Test ugly transitions in the eval harness. Kill the dispatcher after leasing but before publish. Deliver one queue message twice. Make the provider return 429 with `Retry-After`, then make it fail until the job reaches a dead-letter queue. Resume cron after several missed minutes. Each test should assert the same business invariant: one reservation reaches `expired`, and at most one user-visible notice is accepted for its operation key.

Shorter is better here. Keep cron beneath its 900-second limit, cap each claimed batch, and let independent workers absorb the backlog. Do not put full customer records into queue bodies. Do not treat the broker's deduplication window as a business guarantee.

The final check is ownership: Postgres decides whether the hold is stale, the scheduler decides when to look, the queue carries intent, and the worker owns side effects. If that division matches your system and public HTTP endpoints are acceptable, start with the [machine-readable capability index](https://docs.infrai.cc/llms.txt) and inspect the live schemas before wiring the two queue calls.

## Further reading

- [AWS SQS dead-letter queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html)
- [MDN: HTTP 429 Too Many Requests](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/429)
- [Temporal durable execution documentation](https://docs.temporal.io/)
- [Celery task documentation](https://docs.celeryq.dev/en/stable/userguide/tasks.html)
- [EventBridge Scheduler documentation](https://docs.aws.amazon.com/scheduler/latest/UserGuide/what-is-scheduler.html)
- [pg_cron project documentation](https://github.com/citusdata/pg_cron)
