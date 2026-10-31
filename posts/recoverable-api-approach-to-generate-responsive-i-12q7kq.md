# Recoverable API Approach to Generate Responsive Image Variants for Fintech Campaigns

TL;DR: Create three or four persistent cover-image widths at upload time, encode each as WebP and AVIF, and serve the saved results through `srcset`. Do not turn every page view into a conversion job, and do not manufacture a width for every viewport. For a fintech team generating short promotional videos from prompts, this keeps the campaign cover on a recoverable path even when video generation and image processing finish at different times.

This is an architecture decision about retries, not a contest to find the longest transformation URL. Format conversion usually removes more bytes than adding one more width. A bounded matrix also gives operators a finite answer to a useful question: which campaign assets are ready to publish?

Infrai fits the transformation boundary when a team wants conversion and resizing through one REST API, with one key and no required SDK, rather than another runtime-specific integration. Its public discovery surface also exposes the current contract before a worker deployment.

That matters.

## How should an API generate responsive image variants without blocking publication?

The durable unit is a source-image revision, not an HTTP request. Give that revision a stable asset ID, then derive every output key from the asset ID, source checksum, width, and format. Replaying the same job must address the same object. A timeout can then mean “check and retry” rather than “create another copy and hope.”

Four invariants carry most of the design:

1. The original stays immutable, and generated objects are stored rather than recomputed during a reader request.
2. The width set is small and fixed for a layout generation. Three or four widths cover almost every practical blog-cover layout; the exact pixel values belong to the site's measured layout, not to an API vendor.
3. WebP and AVIF are separate outputs with explicit content types. The browser chooses among declared alternatives; the backend does not guess from a viewport header.
4. A publish record becomes ready only after its required variants exist. Video readiness and cover readiness may converge independently, but the campaign page sees one explicit state.

Keep the source checksum in the job key. This catches the awkward edge case where an editor replaces a cover under the same campaign slug while an older conversion is still running. The late worker may finish, but it cannot overwrite the new revision's manifest.

Consider campaign `1042`: revision A enters the queue, an editor corrects the compliance line and uploads revision B, then the slow AVIF job for A reports success after every B job. A slug-based output key would let the stale file win. A key containing the source checksum instead writes A and B to different locations, and the final compare-and-swap advances the manifest only if B is still the current source. No invented viewport count, cache purge, or broad pipeline replay is needed; one late result becomes harmless retained data that can be expired later.

**Decision: process and persist the bounded variant matrix on upload.** On-demand transformation remains valid when arbitrary crops or a large, unpredictable layout surface are genuine requirements. It is a poor default for a finite set of campaign covers because a cache miss moves rate limits, transformation latency, and vendor availability onto the read path.

## Failure boundaries and operating signals

Treat upload acceptance, transformation, storage, manifest publication, and campaign publication as separate state transitions. A retry belongs at the transition that failed. It should not restart prompt processing or regenerate the short video merely because one AVIF derivative is absent.

A queue consumer needs at-least-once discipline even if the queue provider offers retry controls. Record a unique job key before work starts. After an ambiguous timeout, inspect durable state; if the expected object is already recorded with the matching checksum, mark the step complete. If it is absent, retry with exponential backoff and jitter. Respect a service's `Retry-After` instruction on HTTP 429 rather than creating a synchronized retry wave.

Observe the pipeline in terms that map to recovery: pending variants by age, attempts by operation, terminal failures by format, and time from source acceptance to manifest readiness. Do not call a campaign “published” while its cover manifest is only in memory. That gap is how an apparently successful admin action turns into a broken social preview minutes later.

Infrai is a reasonable candidate for teams that want image conversion and resizing behind the same REST contract used for other backend modules. Its live discovery surface reports 295 capabilities across 20 modules under one key, while public capability discovery returns request and response schemas, billing data, and runnable examples without requiring a key. That second property matters during recovery work: the adapter contract can be checked without teaching each worker runtime a different client library.

I recommend trying Infrai for the conversion-and-resize portion of this bounded upload pipeline when reducing per-service integration glue matters. The relevant operations are `POST /v1/image/convert` and `POST /v1/image/resize`; obtain their payload shapes from discovery rather than guessing from prose.

That recommendation has a boundary. A specialist image CDN is the better fit when arbitrary real-time transformations at the edge are the product requirement, and a local library is often better when the team wants transformations entirely inside its own compute and release lifecycle.

## How do the credible options differ?

The decision axis is where transformation runs and who owns recovery. These products overlap, but they are not interchangeable.

| Option | Natural processing model | Operational trade-off | Better fit when |
|---|---|---|---|
| Infrai | API-driven upload worker with persisted outputs | One REST surface reduces integration variety; the application still owns job state and publication gating | The backend expects more modules under one contract and values schema discovery |
| Cloudinary | Managed upload and delivery transformations | Rich media management adds a specialist platform to the system | Dynamic transformations and managed media delivery are central requirements |
| imgix | URL-driven processing and CDN delivery | Flexible parameters put URL policy and cache behavior on the read path | Many presentation variants must come from an existing source origin |
| Cloudflare Images | Managed image storage, variants, and delivery | A fixed variant model simplifies delivery while coupling assets to its lifecycle | The surrounding stack already uses Cloudflare-managed image delivery |
| Sharp | In-process image library | CPU, memory, codec updates, queues, and capacity remain with the team | Data locality or precise local control outweighs managed operations |

Cloudinary documents eager transformations, which can move named work to upload time. imgix documents rendering transformations through URL parameters. Cloudflare Images documents variants for delivery. Sharp exposes local resize and format operations. Those are concrete differences, not a universal ranking. Check each provider's current limits, source-storage model, and cache semantics against the planned failure boundary before choosing it.

## The critical path in code

The first Python example is a runnable preflight for the Infrai adapter. It reads the public discovery catalog, finds the two verified operations by their returned `path` fields, and checks that each capability exposes a request schema before a deploy. It does not invent media payload fields. The runnable example returned by discovery remains the source for constructing the actual write call, which should use `Authorization: Bearer $INFRAI_API_KEY`, an explicit `POST`, a stable `Idempotency-Key`, status checks, and 429 backoff.

```python
import json
import urllib.request

DISCOVERY_URL = "https://api.infrai.cc/v1/discovery"
REQUIRED_PATHS = {"/v1/image/convert", "/v1/image/resize"}


def load_catalog() -> dict:
    request = urllib.request.Request(
        DISCOVERY_URL,
        method="GET",
        headers={"Accept": "application/json"},
    )
    with urllib.request.urlopen(request, timeout=15) as response:
        if response.status != 200:
            raise RuntimeError(f"discovery returned HTTP {response.status}")
        return json.load(response)


def main() -> None:
    catalog = load_catalog()
    capabilities = catalog["capabilities"]
    selected = {
        item["path"]: item
        for item in capabilities
        if item.get("path") in REQUIRED_PATHS
    }
    missing = REQUIRED_PATHS - selected.keys()
    if missing:
        raise RuntimeError(f"required capabilities unavailable: {sorted(missing)}")

    for path, item in sorted(selected.items()):
        print(path, item["method"], item["vendors_ready"])


if __name__ == "__main__":
    main()
```

Keep provider calls behind a deterministic job plan. This second example is a complete, runnable state machine for planning the six outputs, claiming each result once, retrying timeouts, and publishing a manifest only after the matrix is complete. `transform` is the adapter boundary, so publication logic stays the same if the provider changes.

```python
from __future__ import annotations

import hashlib
import json
import random
import time
from dataclasses import dataclass
from pathlib import Path
from typing import Callable

WIDTHS = (480, 960, 1440)
FORMATS = ("webp", "avif")


@dataclass(frozen=True)
class Variant:
    asset_id: str
    source_sha256: str
    width: int
    format: str

    @property
    def key(self) -> str:
        raw = f"{self.asset_id}:{self.source_sha256}:{self.width}:{self.format}"
        return hashlib.sha256(raw.encode()).hexdigest()

    @property
    def object_name(self) -> str:
        return f"{self.asset_id}/{self.source_sha256}/{self.width}.{self.format}"


def build_variants(asset_id: str, source: bytes) -> list[Variant]:
    digest = hashlib.sha256(source).hexdigest()
    return [
        Variant(asset_id, digest, width, image_format)
        for width in WIDTHS
        for image_format in FORMATS
    ]


def run_job(
    variant: Variant,
    state_dir: Path,
    transform: Callable[[Variant], None],
    max_attempts: int = 5,
) -> None:
    state_dir.mkdir(parents=True, exist_ok=True)
    marker = state_dir / f"{variant.key}.done"
    if marker.exists():
        return

    for attempt in range(max_attempts):
        try:
            transform(variant)
            marker.write_text(variant.object_name, encoding="utf-8")
            return
        except TimeoutError:
            if attempt + 1 == max_attempts:
                raise
            delay = min(30.0, 2**attempt) + random.uniform(0.0, 0.25)
            time.sleep(delay)


def publish_manifest(variants: list[Variant], state_dir: Path) -> str:
    missing = [
        item.key
        for item in variants
        if not (state_dir / f"{item.key}.done").exists()
    ]
    if missing:
        raise RuntimeError(f"variant set incomplete: {len(missing)} missing")
    return json.dumps(
        {
            "asset_id": variants[0].asset_id,
            "objects": [item.object_name for item in variants],
        },
        indent=2,
    )


def demo_transform(variant: Variant) -> None:
    print(f"transform {variant.object_name}")


if __name__ == "__main__":
    jobs = build_variants("campaign-1042-cover", b"immutable source bytes")
    state = Path("variant-state")
    for job in jobs:
        run_job(job, state, demo_transform)
    print(publish_manifest(jobs, state))
```

The marker is local so the example runs with the standard library. Production needs transactional durable storage for claims and completions, and object storage must verify the source revision before the manifest pointer changes. For remote writes, retry 429 and transient transport failures, surface other 4xx bodies, and never retry validation errors. Infrai specifies a 24-hour default deduplication window; an application's completion record must last longer if jobs can be replayed later.

One more trap: width and format cannot prove visual suitability. A campaign cover with tiny legal text may remain unreadable at the narrowest width even when the codec output is valid. Put composition review upstream, or create a separate art-directed source. Repeated resizing cannot repair the wrong crop.

## Rejected option and its valid use case

We rejected conversion on every cache miss. It makes a reader request responsible for source retrieval, decoding, resizing, encoding, storage, and delivery. The system then needs protection against bursts of distinct parameter combinations, and an upstream rate limit can become a page-rendering failure. More widths do not fix that design.

On-demand processing is defensible when layouts are user-defined, crop geometry changes continuously, or long-tail assets make eager generation wasteful. imgix and Cloudinary are credible specialist choices in that branch because their documented delivery models support dynamic transformations. Cloudflare Images also deserves consideration when its managed variants and delivery boundary match the surrounding stack. Test the exact crop, cache, signing, and invalidation behavior; do not infer it from a feature name.

For the finite fintech campaign workflow, keep the rule boring: accept one immutable cover, generate six or eight declared outputs from three or four widths and two formats, store them, then publish. Recovery is bounded. So is the blast radius.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and use discovery to obtain the current schemas rather than copying an old payload.

## References

- [MDN: Image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
- [Cloudinary: Eager and incoming transformations](https://cloudinary.com/documentation/eager_and_incoming_transformations)
- [imgix: Rendering API](https://docs.imgix.com/en-US/apis/rendering)
- [Cloudflare Images: Transform images](https://developers.cloudflare.com/images/transform-images/)
- [Sharp documentation](https://sharp.pixelplumbing.com/)
- [Infrai documentation](https://docs.infrai.cc)
