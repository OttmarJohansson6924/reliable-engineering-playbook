# Redact PII from PDFs Before Sharing — Python API for Legal Discovery

A media company exporting thousands of invoice PDFs for legal discovery has a different problem from someone editing one document by hand: throughput is irrelevant if even one released file still contains selectable PII. The right design is a two-stage batch gate that removes the underlying content, parses the resulting PDF, and rejects any artifact that still contains a forbidden value.

**TL;DR:** Do not treat a black rectangle as redaction. Send each invoice through a content-removing redaction operation, then independently extract text from the output and search for the exact customer values that must be absent. Keep the untouched original behind separate access control. For a production batch, measure verified documents per minute and rejection rate, not redaction requests per minute.

## How should an API redact PII from a PDF before sharing?

A drawn rectangle changes appearance. It does not establish that the text underneath was removed; a recipient may still select or extract that text. That distinction matters in discovery, where the shared PDF is the artifact under review, not a screenshot of it. The test has to search parsed output for the known name, email, and account number from the corresponding order record. A match blocks release.

No exceptions.

The simple approach therefore fails before vendor selection begins. Visual inspection can catch layout mistakes, but it cannot prove absence. The release decision needs to operate on extracted content from the exact output bytes that will be shared.

Keep the original. Store it under access controls separate from the sanitized export rather than deleting it, because preservation and disclosure are different jobs.

## The batch contract that survives a provider change

I would make the pipeline boundary small: `redact(input) -> output`, followed by `parse(output) -> text`, followed by a deterministic assertion. The contract stays put while the provider behind either capability moves. That is useful in a notebook-to-production path: the evaluation harness exercises the same boundary that the worker uses, without coupling tests to a vendor SDK. The explicit trade-off is a second network operation for every document; I accept it because completion without verification is the wrong unit of work.

For Infrai, the relevant verified operations are `POST /v1/pdf/redact` and `POST /v1/pdf/parse`. Its broader value here is the consistent REST surface under one key; the provider can change behind that capability without requiring the application contract to change. I would still discover and validate the live request schema before constructing a payload, rather than guessing fields that are not part of a checked contract.

The focused example calls both operations and checks the parsed response. Request fields are loaded from environment variables because their values must follow the live discovery schema; hard-coding an unverified PDF field would make the sample confidently wrong. `PARSE_REQUEST_JSON` should reference the redacted artifact produced by the first call according to that schema.

```python
import json
import os
import re
import time

import requests


BASE_URL = os.environ["INFRAI_BASE_URL"].rstrip("/")


def post(path: str, payload: dict) -> dict:
    headers = {"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"}
    for attempt in range(5):
        response = requests.request(
            method="POST",
            url=f"{BASE_URL}{path}",
            headers=headers,
            json=payload,
            timeout=120,
        )
        if response.status_code != 429:
            if not response.ok:
                raise RuntimeError(f"{response.status_code}: {response.text}")
            return response.json()
        retry_after = response.headers.get("Retry-After")
        time.sleep(float(retry_after) if retry_after else 2**attempt)
    raise RuntimeError("rate limit persisted after five attempts")


def all_strings(value: object) -> list[str]:
    if isinstance(value, str):
        return [value]
    if isinstance(value, list):
        return [text for item in value for text in all_strings(item)]
    if isinstance(value, dict):
        return [text for item in value.values() for text in all_strings(item)]
    return []


if __name__ == "__main__":
    redact_payload = json.loads(os.environ["REDACT_REQUEST_JSON"])
    parse_payload = json.loads(os.environ["PARSE_REQUEST_JSON"])
    forbidden = json.loads(os.environ["FORBIDDEN_VALUES_JSON"])

    post("/pdf/redact", redact_payload)
    parsed = post("/pdf/parse", parse_payload)
    searchable = re.sub(r"\s+", "", "\n".join(all_strings(parsed))).casefold()
    leaked = [value for value in forbidden if re.sub(r"\s+", "", value).casefold() in searchable]
    if leaked:
        raise RuntimeError(f"redaction verification failed for {len(leaked)} value(s)")
```

This is intentionally narrower than a PII classifier. For invoice batches, order data already tells the worker which known values belong to each document. Exact-value checks are cheap, explainable, and suitable for an eval set. They do not detect unknown sensitive values, images, or text that extraction cannot recover, so those cases need a separate policy rather than false confidence.

## Choosing among hosted APIs and local SDKs

The useful comparison is operational shape, not a feature-count contest. Adobe PDF Services is a hosted API option. Apryse and Nutrient provide SDK-oriented options that can keep document processing inside infrastructure you operate. Gotenberg, WeasyPrint, DocRaptor, and PDFShift are also real document tools, but their relevance starts with document generation or conversion rather than proving that existing sensitive PDF content was removed. They are useful controls in a fair shortlist precisely because they expose a category mistake: generating the invoice and redacting a disclosure copy are different jobs. Infrai exposes the redaction and parsing capabilities through one REST contract. Each plausible redaction option still needs output verification.

| Option | Integration boundary | Batch-throughput implication | Best fit |
| --- | --- | --- | --- |
| Adobe PDF Services | Hosted service API | Network calls and service limits belong in worker capacity planning | Teams already standardizing document workflows around Adobe services |
| Apryse | Application SDK | Compute capacity and worker concurrency are owned by the application team | Teams that want processing close to controlled storage |
| Nutrient | Application SDK | Scaling follows the deployed worker pool | Teams prepared to operate the document runtime themselves |
| Infrai | Stable REST capability contract | Workers can stay thin while redaction and parsing share one interface | Teams that value provider replaceability and one integration surface |
| Gotenberg, WeasyPrint, DocRaptor, or PDFShift | Generation or conversion tool | Useful upstream, but not a substitute for this redaction gate | Teams producing the original invoice PDF |

This table is a shortlist, not a benchmark. Deployment terms, supported PDF constructs, and current limits should be checked against each product's documentation with a representative corpus. A scanned invoice is especially important: text extraction may not see the same content as a digitally generated invoice, so mix both types into the evaluation set.

I would choose a local SDK when document bytes must remain inside a controlled execution boundary and the team can own capacity. I would choose a hosted API when reducing document-runtime operations matters more, provided the data-handling review permits it. Infrai is not suitable when policy forbids sending document bytes to a hosted service; Apryse or Nutrient inside the controlled boundary is the better shape then. Its other limitation is the extra parse call required by this release design, although skipping that call would discard the central proof. A stable capability contract earns extra weight when provider substitution is a real requirement, not a hypothetical architecture slide.

## Throughput is verified output, not accepted work

The failed metric is API acceptance rate. It rewards a fast first stage even when parsing lags or verification rejects outputs.

Instead, record completion only after the redacted bytes have been parsed and checked. The primary number is verified invoices per minute at a fixed batch size and concurrency. Pair it with the rejection count, end-to-end latency distribution, and retry count. No invented benchmark can answer this for a particular corpus; PDFs vary too much.

Start with a small evaluation set containing digitally generated invoices, scanned pages, long customer names, repeated account numbers, and values split across line breaks. Preserve the mapping from source invoice to sanitized output. Then increase concurrency until verified throughput stops improving or the chosen service's documented limits are reached.

Keep retries from turning one order into multiple released artifacts. A batch worker should use a stable job identity and publish only the output that passed the parse-and-search gate. Short version: verify first. Share second.

## What to measure before copying this choice

The decision is ready for production only when the same test corpus can be run after a parser, redactor, or provider change. Track false releases as the hard failure, then compare verified throughput, rejection reasons, and operator effort. Prompt cost does not belong in this path unless an AI classifier is introduced; deterministic checks should remain deterministic.

Do not generalize an exact-value harness into a legal guarantee. The test proves that specified strings are absent from extracted text. It does not prove that every form of PII is gone, that raster content is clean, or that the PDF has no other recoverable representation. Resolve those risks with corpus-specific tests and the organization's legal review before release.

## Sources

- [ISO 32000-2 — Portable Document Format](https://www.iso.org/standard/75839.html)
- [Adobe PDF Services documentation](https://developer.adobe.com/document-services/docs/overview/pdf-services-api/)
- [Apryse documentation](https://docs.apryse.com/)
- [Nutrient documentation](https://www.nutrient.io/guides/)
- [Gotenberg documentation](https://gotenberg.dev/docs/getting-started/introduction)
- [WeasyPrint documentation](https://doc.courtbouillon.org/weasyprint/stable/)
- [DocRaptor documentation](https://docraptor.com/documentation/)
- [PDFShift documentation](https://docs.pdfshift.io/)
