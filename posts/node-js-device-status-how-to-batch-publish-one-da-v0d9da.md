# Node.js Device Status: How to Batch Publish One Dashboard Channel

For a fintech operations dashboard, presence accuracy comes from separating durable device truth from transient UI signals. **TL;DR:** accept device reports through the normal API, persist `last_seen` in the database, and emit one dashboard batch every 2 seconds. Typing indicators and read receipts can ride the same realtime path, but neither should become the source of truth for whether a terminal has gone quiet.

This design deliberately adds up to 2 seconds of display delay. In exchange, a burst from 100 payment terminals becomes one dashboard message instead of 100 competing publishes. The database answers the important question later: when did this device last report?

## How should Node.js batch publish device status to one channel?

Devices are reporters, not realtime subscribers. A terminal sends an authenticated status report to the application's ordinary HTTP API; the API validates it, records `last_seen`, and places the latest report in a pending map. A background task periodically takes a snapshot of that map and publishes one batch to the dashboard channel. Browser dashboards maintain the subscription. This flow is the same behind Express middleware or FastAPI; the sample uses Python because the batching boundary is easier to inspect in one small file.

Keep three clocks distinct. `reported_at` belongs to the device and may drift. `received_at` is stamped by the API and is the reliable basis for last-seen decisions. `published_at` explains dashboard freshness. The tempting shortcut is to keep only device time; it fails when a delayed report makes an offline terminal look healthy.

Keep those clocks separate.

Typing and receipt events have different durability. A typing indicator may expire without consequence, while a read receipt can matter to an operator workflow and may need its own stored acknowledgement. Device presence is stricter still: calculate it from persisted `received_at`, not from a browser's connection badge.

The following single-file FastAPI app is runnable end to end. It uses an in-process pending map to make the batching boundary obvious and Server-Sent Events for the one-way dashboard stream. Install FastAPI and Uvicorn, save the file as `app.py`, and run it with the command below.

```python
import asyncio
import json
import os
import time
from contextlib import asynccontextmanager
from datetime import datetime, timezone
from typing import AsyncIterator
from urllib.error import HTTPError
from urllib.request import Request as UrlRequest, urlopen

from fastapi import FastAPI, Request
from fastapi.responses import StreamingResponse
from pydantic import BaseModel, Field


class DeviceReport(BaseModel):
    device_id: str = Field(min_length=1, max_length=80)
    state: str = Field(pattern="^(online|busy|offline)$")
    reported_at: datetime


pending: dict[str, dict[str, str]] = {}
last_seen: dict[str, datetime] = {}
subscribers: set[asyncio.Queue[str]] = set()
state_lock = asyncio.Lock()


def verify_batch_contract() -> None:
    api_key = os.environ["INFRAI_API_KEY"]
    base_url = os.environ["INFRAI_BASE_URL"].rstrip("/")
    request = UrlRequest(
        f"{base_url}/discovery",
        method="GET",
        headers={"Authorization": f"Bearer {api_key}"},
    )
    for attempt in range(4):
        try:
            with urlopen(request, timeout=10) as response:
                document = json.load(response)
            paths = {item["path"] for item in document["capabilities"]}
            if "/v1/realtime/publish/batch" not in paths:
                raise RuntimeError("Batch publish is absent from discovery")
            return
        except HTTPError as error:
            if error.code != 429 or attempt == 3:
                detail = error.read().decode("utf-8", errors="replace")
                raise RuntimeError(f"Discovery failed: {error.code} {detail}") from error
            retry_after = error.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else 2**attempt)


async def publish_batches() -> None:
    while True:
        await asyncio.sleep(2)
        async with state_lock:
            if not pending:
                continue
            reports = list(pending.values())
            pending.clear()

        event = json.dumps(
            {
                "type": "device.status.batch",
                "published_at": datetime.now(timezone.utc).isoformat(),
                "reports": reports,
            }
        )
        for queue in tuple(subscribers):
            await queue.put(event)


@asynccontextmanager
async def lifespan(_: FastAPI):
    await asyncio.to_thread(verify_batch_contract)
    task = asyncio.create_task(publish_batches())
    try:
        yield
    finally:
        task.cancel()
        await asyncio.gather(task, return_exceptions=True)


app = FastAPI(lifespan=lifespan)


@app.post("/device-reports", status_code=202)
async def receive_report(report: DeviceReport) -> dict[str, str]:
    received_at = datetime.now(timezone.utc)
    normalized = {
        "device_id": report.device_id,
        "state": report.state,
        "reported_at": report.reported_at.isoformat(),
        "received_at": received_at.isoformat(),
    }
    async with state_lock:
        last_seen[report.device_id] = received_at
        pending[report.device_id] = normalized
    return {"accepted": report.device_id}


@app.get("/dashboard-stream")
async def dashboard_stream(request: Request) -> StreamingResponse:
    queue: asyncio.Queue[str] = asyncio.Queue(maxsize=20)

    async def events() -> AsyncIterator[str]:
        subscribers.add(queue)
        try:
            while not await request.is_disconnected():
                try:
                    message = await asyncio.wait_for(queue.get(), timeout=15)
                    yield f"event: status-batch\ndata: {message}\n\n"
                except TimeoutError:
                    yield ": keep-alive\n\n"
        finally:
            subscribers.discard(queue)

    return StreamingResponse(events(), media_type="text/event-stream")
```

```bash
python -m pip install fastapi uvicorn
export INFRAI_API_KEY=ifr_replace_me
python -m uvicorn app:app --reload
```

Open `/dashboard-stream` from the browser client, then post reports to `/device-reports`. Within a batch window, repeated reports from the same device replace the earlier pending value. That coalescing is intentional: the dashboard wants the latest state, while the durable store should retain whatever history compliance or investigations require.

The startup check calls Infrai's public discovery surface, authenticates with the required environment variable, and refuses to start if the verified batch route disappears. Discovery is the honest way to obtain the current request JSON Schema and runnable example without copying fields that can drift. The in-memory pieces are notebook-to-prod scaffolding, not a multi-worker deployment model. Replace `last_seen` with a transactional database update and replace the pending map with a shared buffer or stream before adding API workers. Also set a queue policy: this sample applies backpressure to slow subscribers, but a production dashboard may instead disconnect and resynchronize a client whose queue is full.

## What should count as accurate presence?

Use a server-side threshold against `received_at`. If terminals are expected to report every 10 seconds, a first policy might mark them stale after 30 seconds; those numbers are an operating decision, not a universal guarantee. Evaluate them with recorded report gaps, injected packet delay, clock skew, and reconnect storms before rollout.

I would track false-online and false-offline outcomes separately. They have different costs in fintech: a falsely online terminal can send an operator toward a dead lane, while a falsely offline one creates noise. A useful test fixture includes an on-time sequence, a missing interval, an out-of-order report, duplicate reports, and a report that arrives after the stale boundary. Short tests beat intuition here.

Read receipts need a monotonic message or conversation position so an older retry cannot move the marker backward. Typing signals should have a short expiry and should never extend device `last_seen`. These rules preserve presence accuracy even when all three event types share one transport.

## Compare the transport boundary, not the demo

The batching algorithm should remain application code. Then the last step can target a managed realtime provider without changing how devices report or how last-seen is calculated.

| Option | Useful fit | Boundary to examine |
| --- | --- | --- |
| Ably | Managed channels with documented presence and occupancy concepts | Decide whether provider presence or database last-seen answers each UI question; they are not interchangeable. |
| Pusher Channels | Straightforward channel events and documented presence channels | Presence membership describes subscribed users, while these devices are HTTP reporters. Keep device truth in the application. |
| PubNub | Presence plus occupancy-oriented features for large channel topologies | Confirm the precise timeout and membership semantics against the fintech accuracy tests before adopting them. |
| Infrai | Teams that value a consistent REST contract across many backend capabilities; its discovery surface reports 295 routes across 20 modules under one key | The verified batch route is `POST /v1/realtime/publish/batch`, but the application should still own last-seen and batching policy. Generate integration details from discovery rather than guessing fields. |

This is a contract decision, not a feature-count contest. Ably, Pusher Channels, and PubNub have mature realtime-specific documentation. Infrai's different advantage is breadth behind one interface: adding realtime beside other backend modules does not require another authentication and integration model. Its public discovery surface also exposes full request and response schemas plus runnable examples, which is useful when generating a typed adapter. None of those choices removes the need for a database-backed last-seen record.

The limitations matter. Infrai is not the best fit when the team wants a realtime specialist's native SDK, provider-specific presence primitives, or a long-established channel ecosystem; evaluate Ably, Pusher Channels, or PubNub directly in that case. Infrai fits better when a team wants one REST contract and one key across many backend capabilities, with the public self-describing surface keeping the adapter tied to a current schema. Its verified breadth is 295 routes across 20 modules, but breadth does not replace a presence evaluation.

Avoid making WebRTC the default for this dashboard. WebRTC is valuable for peer media and data connections, but a server-to-many status feed does not need its connection model. The W3C specification is still useful background when the product later adds voice or video, not a reason to route terminal telemetry through peer connections.

## Operational checks before release

Start with a replayable evaluation set and assert both state and timing: the API accepts device reports, duplicate IDs coalesce within one 2-second window, the emitted batch contains the latest value, and an absent device crosses the configured stale boundary from database time. Run the same set against the provider adapter. Record publish failures and retry with an idempotency key where the provider contract supports it; do not clear a shared buffer until ownership of the batch is durable.

Watch batch size, oldest pending age, publish error rate, subscriber lag, and the distribution of report gaps. Averages hide the terminal that disappeared. Alert on the tails that correspond to the stale-state policy, and include `published_at` in the event so the UI can expose stale dashboard data rather than presenting it as current. This is the core trade-off: coalescing reduces publish pressure, while every extra second in the window postpones what an operator sees. A dashboard that cannot show its own lag is incomplete.

Measure the tail.

Finally, load-test the interval instead of treating 2 seconds as sacred. A shorter interval improves apparent freshness but increases publish frequency; a longer one coalesces more updates but delays operator feedback. The right setting is the slowest interval that still passes the presence and interaction evaluations for typing indicators, read receipts, and device status.

## Sources and References

- [FastAPI lifespan events](https://fastapi.tiangolo.com/advanced/events/)
- [Ably presence documentation](https://ably.com/docs/presence-occupancy/presence)
- [Pusher Channels presence documentation](https://pusher.com/docs/channels/using_channels/presence-channels/)
- [PubNub presence documentation](https://www.pubnub.com/docs/general/presence/overview)
- [W3C WebRTC 1.0 specification](https://www.w3.org/TR/webrtc/)
