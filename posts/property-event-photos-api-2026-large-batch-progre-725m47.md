# Property Event Photos API 2026: Large-Batch Progress and Cancellation Semantics

**TL;DR:** For a property manager publishing hundreds of photos after a resident event, the least complex useful design is one immutable original, a small set of measured delivery variants, and a durable batch ledger. Optimize the bytes that tenants repeatedly download, record progress from completed items rather than elapsed time, and make cancellation stop new work while preserving an auditable result for every photo.

The bill is mostly shaped by bytes stored and bytes delivered. Transformation compute matters, but a derivative downloaded 10,000 times has a different economic weight from a source decoded once. Start with a worksheet: for each variant, multiply encoded bytes by expected deliveries; separately total original and derivative retention. Those two numbers reveal whether quality tuning, cache policy, or deletion will move the dominant term.

## What actually moves the bill?

Consider a planning example, not a benchmark: 2,000 event photos, averaging 8 MB as uploaded, produce a 400 KB gallery image and a 90 KB thumbnail. One full gallery view transfers about 800 MB; the thumbnails add about 180 MB. The retained originals occupy about 16 GB, while one copy of both delivery variants occupies about 980 MB. If galleries are viewed often, shaving bytes from the gallery variant compounds on every delivery. If they are rarely revisited, original retention may dominate.

Measure both.

Averages can lie. A dark ballroom frame, a detailed group portrait, and a flat sponsor slide respond differently to the same quality setting. Sample by content class and inspect the tails of the encoded-size distribution. The decision rule is deliberately plain: choose the lowest-byte setting that passes a documented visual review at the actual display sizes. No universal quality number survives every event set.

Format choice belongs inside content negotiation and fallback policy. MDN documents broad JPEG support and transparency support in PNG, while newer formats have different compression and feature trade-offs. Keep the source asset independent of the delivery format so a browser capability change does not require another upload from staff.

## How should an API process a large batch of event photos?

Progress is a count of terminal item outcomes, not a timer. A batch with 2,000 inputs should expose totals for queued, running, succeeded, failed, and canceled items; those counts must reconcile to the batch total once processing stops. A percentage alone hides the difference between useful output and terminal failure.

Cancellation is also a state transition, not deletion. Accept the request idempotently, mark the batch `cancel_requested`, stop leasing new items, and let workers finish or abort their current item at an explicit checkpoint. Retried cancellation should return the same state. The controller may mark untouched queued items as canceled, but it must not rewrite successful work as if it never happened.

There is an awkward edge: an encoder can finish just as cancellation arrives. Resolve it with a conditional state update. The worker publishes a derivative only when it can atomically move that item from `running` to `succeeded`; the cancellation path can move eligible items to `canceled`. Store the generated object under a job-scoped temporary key until that transition succeeds, then promote it or schedule it for cleanup.

Exactly one transition wins.

Communication-system habits transfer well here. An accepted request is not proof of delivery, and an accepted cancel request is not proof that every in-flight operation vanished. Name the states honestly. Operators should be able to explain a partial gallery without reading worker logs.

## A small control plane in Python

The following example keeps image encoding behind a generic callable. Its job is narrower: cap concurrency, publish truthful counters, and cooperate with cancellation between expensive operations. The durable store must implement compare-and-set transitions; an in-memory dictionary is not a production substitute.

```python
from concurrent.futures import ThreadPoolExecutor, as_completed
from dataclasses import dataclass
from threading import Event, Lock
from typing import Callable, Iterable


@dataclass(frozen=True)
class Photo:
    photo_id: str
    source_key: str


class Progress:
    def __init__(self, total: int) -> None:
        self.counts = {
            "total": total,
            "succeeded": 0,
            "failed": 0,
            "canceled": 0,
        }
        self._lock = Lock()

    def record(self, outcome: str) -> None:
        with self._lock:
            self.counts[outcome] += 1

    def snapshot(self) -> dict[str, int]:
        with self._lock:
            return dict(self.counts)


def process_batch(
    photos: Iterable[Photo],
    transform: Callable[[Photo], None],
    cancel: Event,
    workers: int = 6,
) -> Progress:
    items = list(photos)
    progress = Progress(len(items))

    def run(photo: Photo) -> str:
        if cancel.is_set():
            return "canceled"
        try:
            transform(photo)
            return "succeeded"
        except Exception:
            return "failed"

    with ThreadPoolExecutor(max_workers=workers) as pool:
        futures = {pool.submit(run, photo): photo for photo in items}
        for future in as_completed(futures):
            progress.record(future.result())

    return progress
```

This sample does not claim hard cancellation of a running encoder. Python threads cannot make that promise here, and killing arbitrary work risks leaving partial files. Production workers need a bounded execution time, checkpoints where the underlying library supports them, and reconciliation for jobs whose lease expires. The concurrency value `6` is an example configuration, not a throughput claim; tune it against CPU, memory, decoder limits, and storage latency.

For large imports, avoid submitting the entire collection to an executor at once as the compact example does. Lease a bounded page from durable storage, renew leases for live work, and stop leasing when cancellation is visible. That keeps memory bounded and lets another worker recover abandoned items.

## Quality gates before rollout

Build a representative fixture set with faces, fine building textures, low-light rooms, text on banners, transparency, and rotated phone captures. Keep the set free of resident data unless consent and retention rules permit its use. Compare candidate variants at their intended CSS dimensions on common display densities, then record the chosen width, format, and quality as a versioned policy.

The operational checks are short but unforgiving:

- Reject decompression bombs and enforce decoded-pixel, file-size, and dimension limits before expensive work.
- Correct orientation intentionally, strip metadata that is not needed for delivery, and decide whether color profiles must be preserved.
- Make item writes idempotent with a key derived from source identity plus policy version.
- Emit batch age, queue depth, item latency, failure reason, encoded bytes, and cancellation lag. Do not put addresses, resident names, or access tokens in labels.
- Retry only failures classified as transient, with a cap and backoff. A corrupt source will not improve on attempt nine.

Test the races. Request cancellation before leasing, during download, during encode, and after upload but before commit. Then crash a worker at each boundary and verify that reconciliation reaches a terminal count without duplicate published objects. A polished progress bar is secondary; state convergence is the contract.

## Retention is a recovery decision

Keep originals long enough to cover review, correction, and policy changes, then delete them according to a written schedule. Derived assets can usually be regenerated while the original exists, so retaining every historical variant indefinitely adds storage and makes deletion requests harder to honor. Record the source checksum, policy version, output checksum, dimensions, byte size, and terminal state in the ledger.

What should be deliberately discarded? Superseded temporary objects, failed partial encodes, stale derivatives outside the supported policy window, and originals past the approved retention deadline. The cost is real: after an original is deleted, a bad crop, an overly aggressive quality setting, or a newly preferred format cannot be repaired from that source. Recovery then means asking staff to upload again, if the file still exists. Make that trade-off visible to the property team before automating deletion.

Keep less, knowingly.

## Further reading

- https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types
- https://owasp.org/www-community/attacks/Decompression_Bomb
- https://www.rfc-editor.org/rfc/rfc9110.html
