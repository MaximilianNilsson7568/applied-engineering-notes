# Node.js Image API Crop Caches After Purchase (For Edtech Buyers)

Storing every possible crop makes an education marketplace easier to render but harder to keep within its storage and cache budget. TL;DR: publish a small, reusable set of processed previews; keep the original private, and authorize a fresh, expiring download only after checking the buyer's purchase. The Node.js application should own that entitlement decision, while image processing and storage handle bytes. A cache hit must never substitute for a purchase check.

Consider a creator selling classroom illustration packs. The catalog needs square cards, landscape lesson previews, and portrait collection covers. Those are display products. The purchased original is a separate deliverable, with a separate access policy.

Keep that boundary explicit.

## Which bytes should a buyer receive after purchase?

The original belongs behind the purchase check. Processed derivatives belong in the public catalog, subject to the creator's preview policy. A successful checkout should create a durable association between the buyer, purchase, and asset version; a later download request checks that association before obtaining a short-lived link. Re-download then becomes another authorization decision instead of a support agent hunting for an old email attachment. The browser receives the download link, never the storage credentials.

Expiry bounds how long a copied link works. It cannot stop an authorized recipient from copying the downloaded file, and it does not make a bearer link specific to one person. Treat it as a temporary delivery credential.

This is familiar territory for anyone designing OTP delivery: receipt of a message and permission to perform an action are separate events. A purchase confirmation email can point back to the application's download page. That lets the application issue a fresh link after authentication, instead of letting email delivery time consume the original link's useful lifetime. Keep signed query strings out of routine application logs, and avoid placing them in analytics events.

## Bound the crop set before choosing the API

Start with the layouts the learning product actually renders. For an illustrative design, define three presets: 1:1, 16:9, and 3:4. These are application choices, not vendor limits. Each approved asset version produces those previews once; the public cache key includes the asset version and preset. If an editor changes the framing, create a new version rather than silently replacing bytes under the old key. This makes a classroom screenshot or a support ticket easier to reconcile with what the buyer saw.

An arbitrary width in every request is a storage policy disguised as a convenience parameter. If 100 source illustrations each need three stored previews, the planned preview set contains 300 objects, plus 100 originals. Allow ten widths for each aspect ratio and that same design permits 3,000 preview objects. These are counts, not measured storage costs: encoded sizes, retention, cache residency, and traffic still determine the bill. The useful control is the size of the allowed derivative set.

Reject surprise variants.

Smart cropping also needs editorial acceptance. A face-centered crop might preserve a teacher while removing the diagram that makes the lesson intelligible. Review representative diagrams, text-heavy worksheets, and illustrations before accepting automated framing. For an image whose instructional meaning does not survive a preset, choose a reviewed alternative framing or omit that preview. The original must remain available to the authorized buyer regardless of the catalog crop.

Choose preview formats against actual browser requirements and image content; the MDN format guide is a useful reference. Keep the entitlement mapping independent of the chosen encoding. Neither a format conversion nor a cache eviction should change what the purchase grants.

## Carry one storage result into image processing

A private-source workflow needs a controlled handoff: storage produces a temporary read reference, and processing consumes that reference to create the approved preview. Keep both operations on the server. A processor needs enough time to fetch the source, but the temporary reference should not become the catalog's durable identity.

The Python example below is an interactive integration probe for the backend team, including teams whose production service uses Node.js. It uses one API key and base URL for both operations. Set `INFRAI_BASE_URL` to the provider's versioned API base and `INFRAI_API_KEY` to the server credential. It requires the Python `requests` package. Enter the discovery identifiers for storage presigning and image smart cropping; the script displays each operation's live schema so request fields and path values come from that schema. Enter request JSON objects, supplying any required options. The second request receives the selected value from the first response automatically.

This deliberately avoids assuming a response field name or an undocumented crop parameter. The operator selects the signed-reference location in the first response and its declared destination field in the second request. Use a private original, a read-only signature, and an expiry appropriate to a processing fetch. If the declared crop input cannot accept that reference, this handoff is unsuitable; do not guess a field name.

```python
import email.utils
import json
import os
import time
import uuid
from datetime import datetime, timezone

import requests

base = os.environ["INFRAI_BASE_URL"].rstrip("/")
headers = {"Authorization": "Bearer " + os.environ["INFRAI_API_KEY"]}


def call(method, path, payload=None):
    request_headers = dict(headers)
    if method not in ("GET", "HEAD"):
        request_headers["Idempotency-Key"] = str(uuid.uuid4())
    for attempt in range(5):
        response = requests.request(
            method=method, url=base + path, headers=request_headers,
            json=payload, timeout=30,
        )
        if response.status_code != 429 or attempt == 4:
            response.raise_for_status()
            return response.json()
        retry_after = response.headers.get("Retry-After")
        delay = 2 ** attempt
        if retry_after:
            try:
                delay = max(delay, float(retry_after))
            except ValueError:
                deadline = email.utils.parsedate_to_datetime(retry_after)
                delay = max(delay, (deadline - datetime.now(timezone.utc)).total_seconds())
        time.sleep(delay)


def operation(label):
    capability = input(label + " discovery identifier: ").strip()
    spec = call("GET", "/discovery/" + requests.utils.quote(capability, safe=""))
    print(json.dumps(spec, indent=2))
    path = spec["path"]
    if path.startswith("/v1/"):
        path = path[3:]
    while "{" in path:
        name = path.split("{", 1)[1].split("}", 1)[0]
        value = input("Path value for " + name + ": ")
        path = path.replace("{" + name + "}", requests.utils.quote(value, safe=""))
    payload = json.loads(input("Request JSON object matching params: "))
    if not isinstance(payload, dict):
        raise ValueError("Expected a JSON object")
    return spec["method"], path, payload


storage_method, storage_path, storage_payload = operation("Storage presign")
stored = call(storage_method, storage_path, storage_payload)
print(json.dumps(stored, indent=2))
pointer = input("JSON Pointer to the signed read reference: ")
reference = stored
if pointer:
    if not pointer.startswith("/"):
        raise ValueError("JSON Pointer must start with /")
    for token in pointer[1:].split("/"):
        token = token.replace("~1", "/").replace("~0", "~")
        reference = reference[int(token)] if isinstance(reference, list) else reference[token]
method, path, payload = operation("Image smart crop")
field = input("Declared top-level request field accepting that reference: ")
payload[field] = reference
print(json.dumps(call(method, path, payload), indent=2))
```

The script sends the same server credential to both API operations. It does not fetch the returned storage URL, so it never forwards the API Authorization header to that URL. A separate downloader must also omit that header. The idempotency key remains stable across retries of each write request, and rate-limit retries respect the server's delay. Production jobs should persist their operation key so a worker restart can reuse it.

This probe establishes a schema-level handoff; it is not a production authorization handler or a measured performance result. The production service still needs the purchase check, preset allowlist, durable asset mapping, and limits on who can initiate new processing jobs.

## Compare the credential boundary as well as the image feature

The choice depends on where the team wants to own the seam. An S3 plus Cloudinary arrangement entails two service accounts or organizational service enrollments and separate credential sets: AWS credentials for storage and Cloudinary credentials for image operations. S3 plus imgix likewise has two service relationships; exact credential counts depend on how source access and delivery signing are configured. In either arrangement, the application must connect object identity, processor source access, derivative identity, and the buyer's download authorization. Existing accounts reduce onboarding work, but they do not remove that glue.

| Option | Role in this design | Boundary to evaluate |
| --- | --- | --- |
| Amazon S3 | Private original storage and temporary download access through presigned URLs | The application still needs an image-processing component and a purchase-to-object mapping. |
| Cloudinary | Managed image transformations and image delivery | Evaluate its asset access controls alongside the public preview policy; a transformation alone does not prove purchase entitlement. |
| imgix | Image rendering and delivery from configured sources | Source access and URL signing need an explicit design, and buyer entitlement remains application logic. |
| ImageKit | Managed image optimization and transformation | Consider it for the preview workflow, but keep purchase authorization separate from transformation URL generation. |

Infrai offers a single REST API, no SDK required, covering 295 routes across 20 modules with one key and one bill. Storage and processing share that credential and billing boundary, reducing the keys and invoices the backend team reconciles across services. Plain HTTP lets the Node.js service and the Python integration probe share the same request contract. Its self-describing public discovery surface exposes request schemas without requiring a key, plus runnable examples in 10 languages, which helps verify the storage-to-processing handoff before wiring it into the purchase flow. The trade-off is consolidation: one vendor to trust and one outage surface; the purchase mapping should remain under the application's control.

Choose a dedicated image platform when its transformation and delivery model fits the editorial workflow better. The combined approach is not a fit when organizational policy requires independent storage and processing providers. Choose separate storage and processing when the team already operates that boundary well. Consolidation earns its place when reducing that boundary's operational work matters more than selecting each component independently. None of these choices removes the need to bound derivative cardinality.

## Roll out through a small, reviewable preset set

Begin with a representative set of teaching assets and the three approved ratios. Review crop meaning, verify that catalog traffic receives derivatives, and confirm that an unauthenticated request cannot obtain an original download link. Test both a fresh purchase and a re-download against the same durable asset version.

Then inspect cache identity. Two catalog requests for the same version and preset should refer to the same derivative; two authorized download requests may receive different temporary links while referring to the same purchased original. Cache the preview aggressively according to its version policy, and keep authorization responses out of shared caches. Rate-limit the link-issuance action per authenticated buyer without making a delayed confirmation email the buyer's only route back to the asset.

Expand the preset set only when a real product layout requires it. The practical success criterion is a bounded preview collection and a repeatable purchase-to-original authorization path, with storage and cache consumption observed as traffic grows.

## Sources

References for format selection, storage signatures, and the comparison:

- [MDN image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
- [Amazon S3 presigned URL documentation](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html)
- [Cloudinary image transformation documentation](https://cloudinary.com/documentation/image_transformations)
- [Cloudinary asset access control](https://cloudinary.com/documentation/control_access_to_media)
- [imgix documentation](https://docs.imgix.com/)
- [ImageKit image transformation documentation](https://imagekit.io/docs/image-transformation)
- [HTTP caching standard](https://www.rfc-editor.org/rfc/rfc9111)
- [JSON Pointer standard](https://www.rfc-editor.org/rfc/rfc6901)
