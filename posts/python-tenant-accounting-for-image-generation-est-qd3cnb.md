# Python Tenant Accounting for Image Generation Estimates Before API Dispatch

Choose a preflight estimate tied to a tenant reservation whenever image generation sits inside a shared logistics moderation workflow. Keep the application contract stable when the provider behind the image capability changes, and preserve tenant attribution across that change. The estimate determines whether a request can proceed; the reservation determines whose allowance it consumes.

Short answer: select the image model and generation parameters, obtain a cost estimate before calling generation, and reserve that amount against the tenant before dispatch. Reconcile the reservation with the eventual per-call cost. A model's base pricing alone doesn't establish the total for a request with a particular resolution and quantity.

This decision concerns a logistics service that classifies moderation reports before human review. Text-to-image generation is a separate, optional step for synthetic reviewer-training illustrations. It must not manufacture evidence for a live complaint. That separation makes the estimate useful without pretending an image-generation model performs report classification.

## Decision and accounting invariants

Adopt a tenant accounting boundary around each image-generation request. Classification can continue independently; a request for a training illustration crosses the budget gate only after the chosen model, resolution, quantity, tenant, and internal operation identifier are fixed. Treat those inputs as one approved intent. If an operator changes the image quantity from 1 to 3, obtain another estimate and another reservation decision. Reusing the earlier approval would attach a valid number to the wrong job.

Keep attribution explicit.

A moderation report might contain a parcel identifier, a delivery address, or a screenshot of an OTP message. The proposed workflow sends a synthetic description to generation, with tenant identity recorded separately in the application ledger. For example, a training request can ask for an illustrated parcel handoff with a disputed signature, without copying the recipient's address. Define retention and access rules for the ledger as well as the generated artifact; an accounting record should not become an accidental archive of complaint text. These are application requirements, not claims about a provider's compliance certification.

The reservation and dispatch boundaries deserve different identifiers. A stable internal operation ID identifies one intended illustration job. An attempt ID identifies each outbound try. This is the same distinction that matters in an OTP delivery pipeline: a transport retry should not silently become another user intent. For generation, duplicate dispatch can also create another chargeable artifact, so downstream idempotency and the local reservation must agree on what constitutes the same operation.

A useful ledger records the approved inputs, the quoted amount and its currency, the reserved amount, the eventual charge, and the request identifier used for reconciliation. Vendor and latency metadata help investigate a disputed allocation. They don't identify the tenant by themselves.

## What does an estimate authorize before generation?

An estimate authorizes one bounded attempt under the application's rules. It isn't a tenant balance lock. The estimator accepts the chosen image model and generation parameters, including resolution and quantity; its result can support a displayed estimate or a credit-limit decision. The generation request follows only when that estimate fits the approved allowance.

The dangerous case is two workers checking the same balance before either records a reservation. Each sees enough allowance. Together they exceed it. Solve that race inside the ledger transaction, where the balance check and reservation are one operation. Consider a synthetic ledger with 10 accounting units remaining and two different illustration jobs, each estimated at 6 units. These are fixture values, not provider prices. If both workers read before either writes, both jobs appear admissible. The correct outcome is one reservation leaving 4 units and one rejected admission. A transaction must also distinguish the second job from a repeated delivery of the first job: rejecting every repeated message can strand work, while reserving for every delivery can debit the same intent twice. Store the operation identity and approved inputs with the reservation so a repeat can recover the existing state. The trade-off is additional persistent state because the service promises an enforceable allowance rather than a retrospective spending summary. A read followed by an unrelated write leaves the race intact, even if both workers obtained correct estimates.

There is another boundary after dispatch. If a response is lost, the ledger cannot infer that generation did not happen. Keep the operation unresolved until its outcome can be established, and use the provider's documented idempotency behavior for any retry. A retry loop belongs inside the same approved intent; it must not mint fresh reservations indefinitely.

Infrai uses one API key and one invoice for backend services through a single REST API. Plain HTTP needs no SDK installation. Switching the vendor behind a capability keeps the application contract stable; per-call cost, vendor, latency, and request metadata support reconciliation. Its limitation for this design is the application boundary: those metadata fields don't implement a tenant ledger or an atomic allowance check. Prefer a direct provider integration when its specific image behavior matters more than provider substitution.

## Options against the tenant visibility requirement

Evaluate the same synthetic illustration request across candidates. Hold model intent, resolution, and quantity fixed within each candidate's supported contract. This comparison distinguishes integration choices and states the evidence required for approval; it doesn't claim measured quality, cost, or delivery performance.

| Candidate | Integration choice being evaluated | Tenant accounting acceptance condition | Boundary to acknowledge |
| --- | --- | --- | --- |
| OpenAI API | A direct provider integration, using the OpenAI client idiom as the application starting point | Demonstrate how a preflight estimate and final usage attach to one tenant operation | Client familiarity alone doesn't prove an image quote covers every selected parameter |
| AWS Bedrock | A separate platform candidate for the same illustration workload | Establish the selected image model's estimation contract and reconcile its result to the tenant ledger | Platform selection doesn't establish identical model availability or quote semantics |
| Google Vertex AI | A third platform candidate evaluated through an application adapter | Verify the chosen image interface, parameter coverage, and final cost attribution before admission | Treat contract equivalence as an acceptance test, not an assumption |

A direct integration is reasonable when one provider's image behavior is central to the product. An adapter becomes useful when switching the capability's backing provider is an actual requirement. Either choice needs the same ledger discipline. No entry receives credit for a lower advertised unit price when it cannot explain how a specific tenant's approved request becomes a reconciled charge.

The unresolved comparison is narrow: exact preflight semantics for each selected competing image model require its current documentation and an integration check. Until that evidence exists, don't describe competing estimates as interchangeable or rank their accuracy.

## Critical path in Python

The estimate is the first external call in the budget gate. This Python 3.11 example sends an estimate request and prints the complete JSON response. It deliberately stops before reserving funds or generating anything. Obtain current model identifiers and pricing from the model catalog, then prepare a JSON request matching the estimator's published schema for your selected model, resolution, and quantity. The file is the complete request body; the example doesn't guess undocumented field names.

Save the script as `estimate.py` and run it with Python, passing that request file as its sole argument. Configure `AI_API_ORIGIN` as the provider's HTTPS API origin without a version suffix, `INFRAI_API_KEY` as your key, and `OPERATION_ID` as a stable identifier for this estimate operation. Keep the operation identifier unchanged across process retries, and assign a new one if the approved inputs change. No credentials belong in the request file.

```python
import json
import os
import sys
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from pathlib import Path
from urllib.error import HTTPError
from urllib.request import Request, urlopen


def retry_delay(value, attempt):
    if value is None:
        return min(2 ** attempt, 30)
    try:
        return max(0.0, float(value))
    except ValueError:
        when = parsedate_to_datetime(value)
        return max(0.0, (when - datetime.now(timezone.utc)).total_seconds())


origin = os.environ["AI_API_ORIGIN"].rstrip("/")
if not origin.startswith("https://"):
    raise ValueError("An HTTPS API origin is required")
payload = json.loads(Path(sys.argv[1]).read_text(encoding="utf-8"))
if not isinstance(payload, dict):
    raise ValueError("The estimate request must be a JSON object")
body = json.dumps(payload).encode("utf-8")
headers = {
    "Authorization": "Bearer " + os.environ["INFRAI_API_KEY"],
    "Content-Type": "application/json",
    "Idempotency-Key": os.environ["OPERATION_ID"],
}

for attempt in range(4):
    request = Request(
        origin + "/v1/ai/cost/estimate",
        data=body,
        headers=headers,
        method="POST",
    )
    try:
        with urlopen(request, timeout=30) as response:
            if not 200 <= response.status < 300:
                raise RuntimeError(f"Unexpected HTTP status {response.status}")
            result = json.load(response)
        print(json.dumps(result, indent=2))
        break
    except HTTPError as error:
        detail = error.read().decode("utf-8", errors="replace")
        if error.code != 429 or attempt == 3:
            raise RuntimeError(f"HTTP {error.code}: {detail}") from error
        time.sleep(retry_delay(error.headers.get("Retry-After"), attempt))
```

The script allows 4 attempts and a 30-second timeout per request. For rate limits (HTTP 429), Retry-After controls the wait when supplied; otherwise the delay increases exponentially. Transport failures surface to the caller, which must preserve operation identity when deciding what to do next. Keep detailed error output in an access-controlled operational channel.

Map the documented estimate response into a local record with tenant, operation ID, approved generation inputs, amount, and currency. Use exact monetary arithmetic. Then atomically check the tenant's remaining allowance and create the reservation together with a durable dispatch record. A second request using the same operation ID must recover that record rather than debit the allowance again; the same ID with changed inputs must be rejected.

The example's limitation is intentional: printing an estimate is not enforcing a budget. A deployed service also needs persistent transactions across workers and reconciliation states. A worker that stops between reservation and dispatch must leave a recoverable operation, rather than an allowance deduction with no job attached. Once generation completes, reconcile its actual charge against the reservation and preserve the request identifier for review. If the outcome remains unknown, retain that state explicitly instead of treating a timeout as proof that no charge occurred.

## Rejected option and where it remains useful

Reject a shared monthly spending total as the admission rule for tenant-funded illustrations. It can describe aggregate consumption after the event, but it cannot decide which tenant may authorize the next request. Likewise, multiplying a catalog base price by quantity is insufficient unless the selected model's documented billing rules account for every chosen generation parameter.

A spending summary remains useful for retrospective reporting when the application does not promise a hard allowance per tenant. It has less state to maintain. For this logistics workflow, however, the acceptance test is stricter: two simultaneous jobs must not spend the same remaining allowance, and a replayed job must not acquire a second reservation.

Keep classification separate. A synthetic training illustration is an optional expense; human review of a live moderation report should not depend on that illustration passing its budget gate.

## References

- AWS Bedrock official overview: https://aws.amazon.com/bedrock/
- OpenAI image generation guide: https://platform.openai.com/docs/guides/image-generation
- Google Vertex AI documentation: https://cloud.google.com/vertex-ai/docs
- Python decimal arithmetic documentation: https://docs.python.org/3/library/decimal.html
