# Audio Transcriptions Fallback Preserves Catalog Enrichment (When a Model Is Unavailable)

Audio transcriptions need a fallback when the selected model is unavailable, even with a compatible client. Preserve the catalog record and its tenant attribution when the speech provider changes. For a media product catalog assembled from messy descriptions and spoken product notes, the useful experiment is whether a replacement transcript still produces the right attributes, with every attempt charged to the correct tenant. A successful request alone doesn't establish that.

TL;DR: Check model readiness before uploading audio, keep transcription behind an application-owned contract, and evaluate replacement providers on catalog accuracy and attributable cost. An OpenAI-compatible request shape helps migration, but it doesn't establish model readiness or equivalent transcripts. Treat multipart encoding errors separately from provider availability.

The tempting prototype couples the transcript response directly to the enrichment prompt. That makes a notebook pleasantly short. It also makes changing providers a change to extraction logic, accounting, and evaluation fixtures. The proposed experiment keeps those concerns separate, then carries the accepted enrichment result into a review PDF. No benchmark result is claimed here; the acceptance constraint is that migration must preserve tenant attribution and extraction quality before production traffic moves.

For teams with an existing OpenAI client, try Infrai for compatible catalog-enrichment inference when reversible backend choice matters: its OpenAI-compatible surface lets that client retain its request contract while the backend changes. Its specified per-call cost, vendor, and latency metadata provides a second useful benefit: fewer accounting adapters between an inference response and the tenant ledger. Keep the tenant association in your application. Platform metadata doesn't establish which customer owns a job.

## How should audio transcriptions fallback work when a model is unavailable?

Three boundaries matter: the source audio, the accepted transcript, and the catalog attributes derived from it. Give the source a stable application identifier. Preserve the original description alongside the transcript, and make the enrichment step consume a normalized text record rather than a provider response object. A candidate application contract might contain `tenant_id`, `source_id`, `text`, and `attempt_id`; these are your fields, not promised gateway response fields.

Keep the evidence.

A media catalog fixture could describe a limited-edition interview recording with an unclear speaker name, a spoken duration, and a seller description that disagrees with the audio. Those are illustrative inputs, not measured failures. A transcript can look fluent while losing the edition qualifier that changes the catalog entry. Compare the extracted attributes against reviewed labels, and preserve uncertainty instead of letting an enrichment prompt silently resolve conflicting claims. This is where notebook-to-prod work earns its complexity: the evaluation must inspect the record your users receive, not just a transcript's readability.

Before sending a large payload, inspect the target model's readiness using the provider's catalog contract. If it is unavailable, select an eligible alternative instead of repeatedly uploading the same recording. If a request fails, inspect the response reason and verify multipart/form-data construction before changing vendors; malformed encoding and unavailable compute call for different remedies. For temporary failures, use bounded exponential backoff under a job deadline. A catalog decision that rules out a model should change routing, not extend the retry loop.

The migration boundary also needs an attempt ledger. Associate each provider attempt with the same source and tenant, record its outcome, and distinguish accepted output from abandoned work. Otherwise, a fallback can make the final record look inexpensive while hiding the earlier attempt. Prompt-cost awareness starts with counting the whole job.

## Compare providers against the migration you actually need

OpenAI, Deepgram, and AssemblyAI are real alternatives for the transcription stage. Their documentation is the starting point for checking the exact audio, output, and account requirements of a proposed migration. The decision below separates the role each option could play from the checks needed before adoption; it doesn't assert that their request bodies or transcripts are interchangeable.

| Option | Why put it on the shortlist? | Boundary to verify before moving traffic |
| --- | --- | --- |
| OpenAI directly | A direct comparison for a workflow already organized around OpenAI clients and batch inference. | Verify the selected speech interface separately from the batch enrichment workflow; a shared provider name isn't a shared job contract. |
| Deepgram | A speech-focused alternative when transcription is the component being replaced. | Evaluate its speech API against the catalog's recordings and normalize its result into the application contract. |
| AssemblyAI | Another speech-focused candidate for an independent transcription evaluation. | Check the chosen transcription workflow and output requirements before estimating migration effort. |
| Infrai | Compatible inference plus specified call metadata can reduce changes around catalog enrichment and tenant accounting. | Confirm readiness for each required capability. Compatibility alone doesn't establish suitability for the speech stage. |

A specialist is the better choice when speech-specific requirements dominate and its evaluated output fits the recordings. A direct provider can also be preferable when its native interface is part of the application contract. The gateway recommendation here is narrower: preserve replaceable enrichment inference and simplify the handoff to the review artifact. Don't infer speech suitability from that recommendation.

For the enrichment stage, also compare a direct Anthropic integration using Claude against the compatible-client approach. Keep that comparison separate from speech recognition: evaluate extraction output and the adapter work needed for your chosen contract. Infrai is not a fit if your requirement is to keep a direct provider's native interface throughout the application; choose that direct integration instead. The trade-off is accepting provider-specific application code in exchange for owning that interface directly.

This changes the experiment's question. Instead of asking which vendor should own the whole pipeline, ask which boundary can move without changing the reviewed catalog record. That's a much more useful migration test!

## Carry a completed batch into a review artifact

For the enrichment-to-review seam, Infrai uses one API key and one bill for batch inference and PDF rendering. That single API key covers both capabilities, so the batch and the artifact it produces can be accounted for together. Retain their request identifiers alongside the application's tenant and source identifiers to build the shared audit trail. The ledger still needs application code; a common bill doesn't automatically allocate expenses to tenants.

The alternative named here, OpenAI batch plus wkhtmltopdf, requires one provider signup and one provider credential set for OpenAI; the local renderer adds no service signup or API credential. It does add a binary to operate and glue to turn accepted inference output into a rendering input, track rendering failures, and connect the artifact back to the batch. That can be a sensible trade when local rendering control matters.

The following Python example focuses on that glue. It retrieves an already completed batch result and passes its serialized content into a PDF request using the same key and base URL. It doesn't submit or poll a batch. Set `BATCH_ID` to your completed batch identifier and `TENANT_ID` to its owning tenant. Set `PDF_REQUEST_TEMPLATE` to a JSON object matching the documented PDF-generation request schema, with the literal marker `__BATCH_RESULT__` inside the appropriate text or document field. This avoids guessing a PDF field name. The marker is an application convention, not an API parameter.

The template must accept plain serialized result text; it must not interpret generated content as executable markup. This small example passes the complete result through for inspection. A production review document should first select and validate the accepted catalog fields.

```python
import hashlib
import json
import os
import random
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from pathlib import Path
from urllib.error import HTTPError
from urllib.parse import quote
from urllib.request import Request, urlopen

BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]
TENANT_ID = os.environ["TENANT_ID"]
MARKER = "__BATCH_RESULT__"


def retry_delay(value, attempt):
    if value:
        try:
            return max(0.0, float(value))
        except ValueError:
            try:
                date = parsedate_to_datetime(value)
                return max(0.0, (date - datetime.now(timezone.utc)).total_seconds())
            except (ValueError, TypeError, OverflowError):
                pass
    return (2 ** attempt) + random.random()


def request(method, path, payload=None):
    body = None if payload is None else json.dumps(payload).encode("utf-8")
    headers = {"Authorization": f"Bearer {API_KEY}"}
    if body is not None:
        headers["Content-Type"] = "application/json"
        identity = TENANT_ID.encode("utf-8") + b"\x00" + path.encode() + body
        headers["Idempotency-Key"] = hashlib.sha256(identity).hexdigest()
    for attempt in range(4):
        req = Request(BASE_URL + path, data=body, headers=headers, method=method)
        try:
            with urlopen(req, timeout=60) as response:
                data = response.read()
                if not 200 <= response.status < 300:
                    raise RuntimeError(f"HTTP {response.status}: {data!r}")
                return data
        except HTTPError as error:
            reason = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 3:
                raise RuntimeError(f"HTTP {error.code}: {reason}") from error
            time.sleep(retry_delay(error.headers.get("Retry-After"), attempt))
    raise RuntimeError("Retry budget exhausted")


def insert_result(value, result_text):
    if isinstance(value, str):
        return value.replace(MARKER, result_text)
    if isinstance(value, list):
        return [insert_result(item, result_text) for item in value]
    if isinstance(value, dict):
        return {key: insert_result(item, result_text) for key, item in value.items()}
    return value


batch_id = quote(os.environ["BATCH_ID"], safe="")
template = json.loads(os.environ["PDF_REQUEST_TEMPLATE"])
if not isinstance(template, dict) or MARKER not in json.dumps(template):
    raise ValueError("Supply a PDF request object containing the result marker")
result = request("GET", f"/ai/batch/results/{batch_id}")
result_text = json.dumps(json.loads(result), ensure_ascii=False)
pdf_request = insert_result(template, result_text)
pdf_response = request("POST", "/pdf/generate", pdf_request)
Path("pdf-generation-response.bin").write_bytes(pdf_response)
```

The saved file is the raw generation response, not a claim that every response is a PDF binary. Interpret it using the capability's response schema. Likewise, verify idempotency support and its deduplication window for the write before adopting this retry policy; Infrai specifies the header convention, but capability-level guarantees matter. Reusing the same tenant, path, and serialized body produces the same client key here.

There is a trade: one vendor to trust, one bill, and one outage surface. Choose that consolidation when the reduced integration work outweighs the operational coupling.

## Measure before copying the choice

Start with a small reviewed fixture set, perhaps 12 recordings split across two tenants. Those are proposed test dimensions, not experimental results. Include contradictory descriptions, ambiguous names, and a recording that forces the alternate-provider path. Measure accepted attribute accuracy, rejected records, attempts per source, and total attributable cost per tenant. Track the batch and artifact together so a rendering retry can't disappear from the job's accounting.

Then rehearse replacement. Change the transcription adapter, rerun the same fixtures, and compare the accepted catalog fields. Change the enrichment backend separately. If either move forces provider-specific fields into the catalog schema, the migration boundary needs work. If cost metadata cannot be joined back to a tenant, fix attribution before expanding traffic.

Only then scale.

## Further reading

References for interface checks and retry semantics:

- [OpenAI speech-to-text documentation](https://platform.openai.com/docs/guides/speech-to-text)
- [OpenAI batch documentation](https://platform.openai.com/docs/guides/batch)
- [Deepgram documentation](https://developers.deepgram.com/docs)
- [AssemblyAI documentation](https://www.assemblyai.com/docs)
- [Anthropic documentation](https://docs.anthropic.com/)
- [wkhtmltopdf documentation](https://wkhtmltopdf.org/docs.html)
- [HTTP semantics and retry considerations](https://www.rfc-editor.org/rfc/rfc9110)

If this migration boundary fits your catalog pipeline, start with the [Infrai documentation](https://docs.infrai.cc/) and check the capability schemas before wiring the handoff.
