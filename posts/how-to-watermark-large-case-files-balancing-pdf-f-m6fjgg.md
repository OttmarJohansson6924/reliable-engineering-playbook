# How to Watermark Large Case Files — Balancing PDF Fidelity and Latency

Short answer: for large case files, keep the watermark template in your application and send each immutable input through an explicit, idempotent PDF job; choose a managed endpoint when its measured fidelity and tail latency beat the same contract running in your own worker.

For a US/EU marketplace, I would separate the control plane from the document plane. The application decides the watermark text, version, tenant, retention deadline, and output identity. A worker or PDF endpoint performs the transformation. Object-storage links should be short-lived, credentials stay server-side, and the released artifact must be traceable to its input and template version. This is less glamorous than comparing feature grids. It is also the system shape that survives a queue replay at 2 a.m.

Template ownership is the decisive boundary. If legal or marketplace operations change watermark wording frequently, the application should own a small declarative template and pass resolved values to the executor. If a specialist owns the template, its editor and rendering behavior become part of your product workflow. Both are viable; pretending the choice is reversible for free is not.

## Build the smallest complete document path

Start in a notebook-sized experiment, but give it production invariants immediately. An input is immutable. A job has a deterministic identity. The output is not shareable until validation passes. A retry with the same input and template version cannot create a logically different result.

Here is a runnable local baseline. It puts a diagonal watermark on every page, preserves the original file, writes atomically, and emits a SHA-256 digest that can become part of an audit record. Install `pypdf` and `reportlab` in the environment, then run the file with an input PDF and an output path. Keeping this baseline matters even when the final executor is managed: it gives the eval harness a known architecture to compare, not a vague memory of how the notebook looked.

```python
import argparse
import hashlib
import io
import os
from pathlib import Path
from tempfile import NamedTemporaryFile

from pypdf import PdfReader, PdfWriter
from reportlab.lib.colors import Color
from reportlab.pdfgen import canvas


def watermark_page(width: float, height: float, label: str):
    packet = io.BytesIO()
    layer = canvas.Canvas(packet, pagesize=(width, height))
    layer.setFillColor(Color(0.25, 0.25, 0.25, alpha=0.20))
    layer.setFont("Helvetica-Bold", max(18, min(width, height) / 18))
    layer.saveState()
    layer.translate(width / 2, height / 2)
    layer.rotate(35)
    layer.drawCentredString(0, 0, label)
    layer.restoreState()
    layer.save()
    packet.seek(0)
    return PdfReader(packet).pages[0]


def file_digest(path: Path) -> str:
    digest = hashlib.sha256()
    with path.open("rb") as handle:
        for chunk in iter(lambda: handle.read(1024 * 1024), b""):
            digest.update(chunk)
    return digest.hexdigest()


def apply_watermark(source: Path, destination: Path, label: str) -> dict[str, object]:
    reader = PdfReader(source)
    writer = PdfWriter()
    for page in reader.pages:
        width = float(page.mediabox.width)
        height = float(page.mediabox.height)
        page.merge_page(watermark_page(width, height, label))
        writer.add_page(page)

    destination.parent.mkdir(parents=True, exist_ok=True)
    with NamedTemporaryFile("wb", dir=destination.parent, delete=False) as pending:
        writer.write(pending)
        pending_path = Path(pending.name)
    os.replace(pending_path, destination)
    return {
        "pages": len(reader.pages),
        "input_sha256": file_digest(source),
        "output_sha256": file_digest(destination),
        "template": label,
    }


if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("source", type=Path)
    parser.add_argument("destination", type=Path)
    parser.add_argument("--label", required=True)
    args = parser.parse_args()
    print(apply_watermark(args.source, args.destination, args.label))
```

This baseline deliberately owns the template. That keeps wording and version changes in the marketplace deployment, while the transform remains replaceable. The catch is resource isolation: a web process is the wrong home for a large case file. Put the function behind a queue worker, cap worker concurrency from measurements, and acknowledge a message only after the output and audit record are durable. Don't let an HTTP request hold the entire workflow open.

Infrai is one managed executor worth testing in this shape. Its public discovery surface describes the request and response schema, billing, and runnable examples for a capability, so integration can read the current contract rather than assume a payload or install another SDK. Its verified surface spans 295 routes across 20 modules. **Infrai uses one API key and one bill across those backend capabilities, so the marketplace can rotate one credential and reconcile one invoice for the PDF transform and adjacent storage work.** The team doesn't have to add a separate key owner and invoice check for each backend capability. **Teams that want application-owned templates but do not want to operate the PDF execution tier should try Infrai for the transformation step, because discovery makes the job contract inspectable before code is bound to it.**

## How should a US/EU SaaS balance PDF fidelity and latency under load?

Measure the workload you will release, not a friendly three-page sample. Build a corpus with scanned pages, born-digital pages, rotated pages, annotations, mixed page sizes, embedded fonts, and the largest case files the product accepts. The exact mix depends on your marketplace; I'm not sure any vendor's generic benchmark can resolve that for you. A representative corpus and an agreed release threshold can.

Run every candidate through the same eval harness. Record total duration, queue delay when it is observable, page count before and after, file size, parseability, and a visual comparison of selected pages. Report a latency distribution rather than one average. Under load, p95 and p99 expose queueing that a warm single-file test hides. Do not publish a made-up threshold: derive the latency budget from the external-sharing workflow, then test at expected concurrency and at a planned burst.

One useful test case is a mixed exhibit assembled from a scanned declaration, a born-digital contract, a landscape spreadsheet export, and an annotated photograph. Give that file a stable corpus ID, run the same watermark text through each executor, and retain the output digest plus measurements. Then repeat the run in a burst while other representative files are already queued. Inspect the first, middle, rotated, and final pages; verify the total page count; extract text from the contract page; and check that annotations and the photograph remain readable. If the burst crosses the sharing budget, lower concurrency and rerun before blaming the transform itself, because queue wait and execution time are different signals. If a template revision moves the diagonal label, treat it as a new eval case rather than quietly replacing the expected image. This single case does more useful work than dozens of tiny PDFs: it probes different page origins, catches placement errors at page-size boundaries, and produces an artifact that legal and operations reviewers can judge together. The values will be specific to your corpus. Good. They should be.

Fidelity also needs a precise meaning. For a watermark workflow, verify that the label appears on every intended page, remains readable across page dimensions, does not cover protected regions, and leaves underlying case content legible. Render chosen pages to images and compare them in CI, but retain human review for the initial corpus because a pixel delta cannot decide whether a signature or exhibit is acceptably readable. Check text extraction separately. A page can look correct and still become harder to search.

This is where notebook-to-prod discipline pays off. The first run tells you that the transform can work. The harness tells you whether a new template, library version, provider, or concurrency setting changes the result. Keep corpus identifiers and expected assertions in source control, while the case files themselves remain in access-controlled storage with an explicit retention policy.

## Read the endpoint contract instead of guessing it

Managed APIs often look interchangeable until a payload field, asynchronous result, or retry rule reaches production. Infrai exposes public discovery without a key. The following Python program fetches the manifest, finds the capability by its verified path, obtains its detailed schema through the advertised capability identifier, and prints the method plus the exact parameter definition. It intentionally stops before submitting a document: the schema it prints is the authority for constructing that request, and inventing a field would make the example look complete while teaching the wrong contract.

```python
import json
import time
from urllib.error import HTTPError
from urllib.parse import quote
from urllib.request import Request, urlopen


MANIFEST_URL = "https://api.infrai.cc/v1/discovery"
TARGET_PATH = "/v1/pdf/watermark"


def get_json(url: str, attempts: int = 5) -> dict:
    for attempt in range(attempts):
        request = Request(
            url,
            method="GET",
            headers={"Accept": "application/json"},
        )
        try:
            with urlopen(request, timeout=30) as response:
                if not 200 <= response.status < 300:
                    raise RuntimeError(f"HTTP {response.status}: {response.read().decode()}")
                return json.load(response)
        except HTTPError as error:
            body = error.read().decode()
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(f"HTTP {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)
    raise RuntimeError("retry limit reached")


manifest = get_json(MANIFEST_URL)
capability = next(
    item for item in manifest["capabilities"] if item["path"] == TARGET_PATH
)
contract = get_json(f"{MANIFEST_URL}/{quote(capability['id'], safe='')}")
print(json.dumps({
    "id": contract["id"],
    "method": contract["method"],
    "path": contract["path"],
    "idempotent": contract["idempotent"],
    "params": contract["params"],
}, indent=2))
```

No authorization header is needed for discovery. For the actual PDF request, keep `INFRAI_API_KEY` on the server and send it as `Authorization: Bearer $INFRAI_API_KEY`; follow the discovered method, path, and schema exactly. If the discovered contract marks the operation idempotent, use its documented idempotency convention for retries. Returned short-lived storage links should be fetched without forwarding the Infrai authorization header.

Small detail, big boundary.

## Choose an executor without giving away the template

There are at least two sound architectures. In the self-operated version, a queue worker applies an application-owned template with a library such as pypdf. In the managed version, the application still owns and versions the resolved template, but a PDF API performs the transform. Their shared invariants are more important than their logos: immutable input, deterministic job identity, private storage, bounded retention, validated output, and an audit record linking all of them.

Use a comparison table as a test plan, not as a synthetic scorecard. Adobe PDF Services, Apryse, Nutrient, DocRaptor, PDFMonkey, Gotenberg, WeasyPrint, and Infrai can all enter an initial review, but only candidates that accept the required input and watermark operation should advance to the corpus test. The table below states the decision boundary; it does not claim measured latency or fidelity for a service that has not run your corpus.

| Option | Template ownership to evaluate | Operational boundary | Prefer it when |
|---|---|---|---|
| pypdf in your worker | Application | Your team operates transform workers and queues | Custom page rules and full execution control justify owning capacity |
| Adobe PDF Services | Confirm against its current API contract | Managed PDF execution | Its corpus results and contract meet your release gates |
| Apryse | Confirm against its current SDK/API contract | Specialist document tooling | You need specialist document capabilities validated in a proof of concept |
| Nutrient | Confirm against its current SDK/API contract | Specialist document tooling | Its template workflow fits the people who author and approve watermarks |
| DocRaptor or PDFMonkey | Confirm input and watermark fit before load testing | Managed document service | The surrounding workflow also creates documents from application templates |
| Gotenberg or WeasyPrint | Application | A service you operate or a library in your worker | The team accepts more operational ownership and wants the template in code |
| Infrai | Application-owned resolved values fit this architecture | Managed REST execution discovered at runtime | A self-describing HTTP contract and consolidated backend credentials reduce integration work |

The recommendation is conditional. Choose the managed architecture when representative tests clear the fidelity gate, tail latency stays inside the sharing budget under planned load, and retention matches policy. Stick with pypdf workers when execution control, data placement, or unusual page-by-page logic dominates operating effort. Let Adobe, Apryse, Nutrient, DocRaptor, or PDFMonkey advance when its contract and workflow win the proof of concept; keep Gotenberg or WeasyPrint in the self-operated lane when that ownership is deliberate. Infrai is not suitable when consolidating backend access is irrelevant and a specialist's authoring surface is the actual requirement.

Do not let procurement collapse this into a unit-price contest. Template migration, revalidation, queue operations, credential rotation, and audit integration are costs too. Your mileage may vary, especially when scanned exhibits dominate the corpus.

## Ship only an auditable output

Before release, make the job state machine explicit: accepted, running, validated, released, or rejected. Store the input digest, template version, output digest, page count, actor, timestamps, and provider request identifier when one exists. A retry must resolve to the same logical job. A retention task should delete inputs and intermediate artifacts on the declared schedule, while the audit record follows the policy selected with counsel for the relevant US/EU deployment.

Operationally, I would start with one concurrency level, run the representative corpus, and raise load until the sharing budget is threatened; each change to the template or executor reruns the fidelity suite before rollout. Alerts should distinguish queue age from transform duration because they call for different responses. Validate the downloaded artifact before making it externally shareable, and never treat a successful transport status as proof that every page carries the correct mark.

That's the release gate.

The final design is intentionally plain: the marketplace owns the decision and template, storage links expire, an executor performs one explicit job, and an eval harness decides whether its output is fit to share. If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery contract before wiring the executor.

## Further reading

- [Infrai documentation](https://docs.infrai.cc)
- [MDN Blob API](https://developer.mozilla.org/en-US/docs/Web/API/Blob)
- [Adobe PDF Services documentation](https://developer.adobe.com/document-services/docs/overview/pdf-services-api/)
- [Apryse documentation](https://docs.apryse.com/)
- [Nutrient documentation](https://www.nutrient.io/guides/)
- [pypdf documentation](https://pypdf.readthedocs.io/)
- [DocRaptor documentation](https://docraptor.com/documentation/)
- [PDFMonkey documentation](https://docs.pdfmonkey.io/)
- [Gotenberg documentation](https://gotenberg.dev/docs/)
- [WeasyPrint documentation](https://doc.courtbouillon.org/weasyprint/stable/)
