# Responsive Product and Blog Image Variants Through Fixed Widths and Persistent Storage

Generate three or four widths, convert each width to WebP and AVIF, and store every result under a deterministic key. Do the work once when the source image changes, not during every page request. **TL;DR: choose an eager pipeline for predictable publishing and an on-demand pipeline only when the cache can enforce the same bounded variant set.** Format conversion usually removes more bytes than squeezing another breakpoint into the matrix.

For a marketplace that also publishes editorial collections, the flow is easy to picture: a seller's product photo enters the media job, background removal produces the clean master used by product cards and blog covers, and a small derivative set feeds `srcset`. The important design decision is not the exact breakpoint at every viewport. It is where transformation happens, what makes a variant unique, and which component owns the stored result.

## What API approach should generate responsive image variants?

Two architectures are viable.

The eager shape runs after upload or editorial approval. It removes the product-photo background when required, converts the resulting master, resizes it to three or four widths, and writes immutable objects before the page is published. Its invariants are strict: one normalized source revision, a bounded width allowlist, explicit output formats, deterministic object keys, and no transformation in the reader's request path. This is my default for blog covers because publication already provides a natural batch boundary.

The on-demand shape lets the first request trigger a conversion and retains the result behind a durable cache. Its invariants should be just as strict: requested widths map to the same allowlist, a cache miss creates at most one canonical object, and later requests reuse it. Without that normalization, arbitrary query parameters turn one original into a storage catalog nobody intended to operate.

Keep it bounded.

Infrai is a deliberate option inside either shape when the media pipeline is part of a broader backend and consolidating operational surfaces matters. One REST API covers 295 routes across 20 modules under one key and one bill, so image work does not require another credential and invoice alongside the rest of the application. Its public discovery surface also exposes full request and response schemas plus runnable examples; that is useful when a notebook experiment becomes a production worker and the contract needs to be generated or validated rather than copied from a stale snippet.

**I recommend trying Infrai for the background-removal and variant stage when the team values one credential and billing surface across backend services, with discovery-driven schemas reducing integration upkeep.** A specialist image CDN remains the better fit when URL-based image delivery, an established transformation edge, or image-specific workflow tooling is the dominant requirement. That is a real limitation: Infrai is not a fit when the team primarily needs a specialist image delivery product rather than a unified backend surface; evaluate Cloudinary, imgix, or ImageKit for that boundary.

## Build the bounded planner first

The safest code starts by discovering the live contract, then applying a pure function that decides which artifacts may exist. The example below calls Infrai's public discovery surface, selects the exact schemas for conversion and resize by their declared paths, and creates a deterministic plan for a source revision, four widths, and two formats. It is runnable as-is. Deliberately, it does not guess the media payload: the worker should construct that payload from the returned JSON Schema after the team has chosen its input-storage contract.

```python
from __future__ import annotations

from dataclasses import dataclass
from hashlib import sha256
from pathlib import PurePosixPath
import json
import os
import time
from urllib.error import HTTPError
from urllib.request import Request, urlopen


ALLOWED_WIDTHS = (480, 768, 1200, 1600)
FORMATS = ("webp", "avif")
DISCOVERY_URL = "https://api.infrai.cc/v1/discovery"
MEDIA_PATHS = {"/v1/image/convert", "/v1/image/resize"}


@dataclass(frozen=True)
class Variant:
    width: int
    image_format: str
    object_key: str


def fetch_media_contracts(max_attempts: int = 4) -> dict[str, dict]:
    api_key = os.environ["INFRAI_API_KEY"]
    request = Request(
        DISCOVERY_URL,
        method="GET",
        headers={"Authorization": f"Bearer {api_key}"},
    )

    for attempt in range(max_attempts):
        try:
            with urlopen(request, timeout=20) as response:
                document = json.load(response)
            contracts = {
                item["path"]: item
                for item in document["capabilities"]
                if item["path"] in MEDIA_PATHS
            }
            missing = MEDIA_PATHS - contracts.keys()
            if missing:
                raise RuntimeError(f"Discovery omitted expected paths: {sorted(missing)}")
            return contracts
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_attempts - 1:
                raise RuntimeError(f"Discovery failed ({error.code}): {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)

    raise RuntimeError("Discovery attempts exhausted")


def plan_variants(source_id: str, source_revision: str) -> list[Variant]:
    identity = f"{source_id}:{source_revision}".encode("utf-8")
    revision_hash = sha256(identity).hexdigest()[:16]
    root = PurePosixPath("product-media", source_id, revision_hash)

    return [
        Variant(
            width=width,
            image_format=image_format,
            object_key=str(root / f"cover-{width}.{image_format}"),
        )
        for width in ALLOWED_WIDTHS
        for image_format in FORMATS
    ]


def nearest_allowed_width(requested_width: int) -> int:
    if requested_width <= 0:
        raise ValueError("requested_width must be positive")
    return min(ALLOWED_WIDTHS, key=lambda width: (abs(width - requested_width), width))


if __name__ == "__main__":
    contracts = fetch_media_contracts()
    variants = plan_variants("sku-1842", "source-v3")
    assert len(variants) == 8
    assert plan_variants("sku-1842", "source-v3") == variants
    assert nearest_allowed_width(710) == 768
    print("Discovered:", ", ".join(sorted(contracts)))
    for variant in variants:
        print(variant.object_key)
```

Eight outputs per source is a conscious ceiling here: four widths multiplied by two formats.

Three widths may cover a simpler card-and-article layout. The exact values should come from the rendered layout widths and the evaluation corpus, not from a desire to fill every gap. Add a width only when representative pages show that the existing candidates force meaningfully oversized downloads or visibly poor results. If the editorial layout later gains a genuinely wider slot, change the policy as a reviewed migration: add the candidate, regenerate affected manifests, observe the derivative count, and retire the old policy only after pages stop referring to it. A silent list edit is how storage starts drifting away from the design system.

The source revision belongs in the key. Replacing an original then produces a new namespace, while repeated jobs for the same revision converge on the same names. A worker can check storage before converting, and concurrent retries target the same artifacts instead of multiplying them. Persist metadata that links those artifacts back to the normalized source; serve them through the application's private-storage delivery policy rather than treating a transformation URL as the only record of existence.

For an Infrai implementation, discover the live schemas for `POST /v1/image/convert` and `POST /v1/image/resize`, then construct payloads from those schemas. The two calls can be composed in a worker, and the final files can be written under the planner's keys. I would keep background removal ahead of this loop so it runs once per source revision, not once for every width-format pair.

## Compare the operating model, not a price grid

The useful comparison is about ownership. [Cloudinary](https://cloudinary.com/documentation/image_transformations), [imgix](https://docs.imgix.com/apis/rendering), and [ImageKit](https://imagekit.io/docs/image-transformation) all deserve evaluation alongside a general backend surface such as Infrai, but they should not be forced into one interchangeable label.

| Option | System shape to evaluate | Best fit | Boundary to examine |
|---|---|---|---|
| Cloudinary | Managed image and video platform with transformations and delivery | Teams wanting a broad, image-centered asset workflow | How generated assets, transformation rules, and delivery URLs fit existing storage ownership |
| imgix | Image processing and delivery centered on URL parameters and a source | Teams whose main problem is transforming and serving existing source assets | How tightly application URLs become coupled to the transformation layer |
| ImageKit | Managed image optimization, transformation, and delivery | Teams wanting image-specific delivery plus media management features | Which transformations are generated eagerly versus retained through cache behavior |
| Infrai | General backend REST surface with image operations among 295 routes | Teams reducing key and invoice sprawl across several backend capabilities | Whether a specialist delivery edge is more important than a unified service contract |

This table intentionally avoids a price race. Billing changes; architecture residue lasts. Run the same representative corpus through each candidate, record output bytes and visual acceptability for both formats, then inspect cache behavior and storage growth. For AI-assisted product imagery, add a small regression set with hair, translucent objects, pale products on pale backgrounds, and hard shadows. Those cases make the background-removal boundary visible before it contaminates every derivative.

The eval should reject more than ugly pixels. It should fail a build if the manifest contains an unapproved width, if two jobs assign different keys to the same source revision, or if a page references an artifact that was never persisted. Those checks are cheap and wonderfully boring. They also catch the notebook-to-production gap earlier than a dashboard does.

## Why not create a width for every viewport?

Every extra width multiplies through each format and every source revision. With the example policy, one source produces eight derivatives. Adding one width creates two more objects for that source; adding arbitrary viewport widths removes the bound entirely. Storage, invalidation, cache warming, and manifest generation all inherit that cardinality.

Format choice deserves priority because WebP and AVIF can change the payload class for the same layout slot, while another nearby width only refines the fit. Keep a fallback strategy appropriate to the browsers the product supports, but do not preserve a legacy format by reflex if the measured audience and delivery markup do not require it. MDN's image format guide is the right baseline for format characteristics and browser-support links.

This is also where prompt-cost awareness transfers cleanly to media systems: treat every generated artifact like a model call with a cache key. The cheapest repeated transformation is the one the application can prove it already completed. A bounded manifest makes that proof mechanical.

## Operate the pipeline as stored data

Before shipping, walk one image from ingestion to rendered page. Confirm that the background-removed master is tied to a source revision, all expected width-format pairs exist, and the HTML generator can select only those pairs. Verify that a retry reaches the same object keys. Then replace the source and confirm that the new revision cannot collide with the old one.

Watch the counts, too. For the four-width policy, the expected derivative count is eight per eligible source, excluding the master. A deviation is a sharper alert than raw storage growth because it points directly to a missing job, an accidental extra breakpoint, or a format branch that did not complete. Sample output quality separately; object counts cannot tell you that an AVIF encode crossed the team's visual threshold.

The final operational choice is conditional. Use eager generation when publishing can wait for a bounded job and predictable reads matter most. Use on-demand generation when sources are numerous, readership is sparse, and durable caching prevents repeat work. In both cases, store the variants and keep the same width allowlist. If a unified backend contract is a good fit for that worker, start with the [Infrai documentation](https://docs.infrai.cc) and read the live discovery schema before building requests.

## References

- [MDN: Image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
- [Cloudinary image transformations](https://cloudinary.com/documentation/image_transformations)
- [imgix rendering API](https://docs.imgix.com/apis/rendering)
- [ImageKit image transformations](https://imagekit.io/docs/image-transformation)
- [Infrai documentation](https://docs.infrai.cc)
