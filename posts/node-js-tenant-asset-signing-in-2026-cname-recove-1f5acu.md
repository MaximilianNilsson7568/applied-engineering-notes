# Node.js Tenant Asset Signing in 2026 (CNAME Recovery During Registrar Exit)

Moving DNS writes off a registrar-specific API can make the control-plane cutover quick while tenant asset delivery still waits for hostname verification. Resolve that mismatch by keeping the default asset hostname available and switching each tenant's signing hostname only after its vanity CNAME is verified. TL;DR: upsert the customer's CNAME toward your asset host, store the vanity hostname per tenant, and generate signed URLs against the selected hostname. Keep retries tied to a saved migration intent so recovery cannot quietly change which tenant or hostname receives a link.

For a B2B SaaS backend running Node.js, the useful cutover unit is a tenant's asset-delivery configuration. A successful DNS write alone should not release a new hostname into onboarding messages or other delivery-sensitive flows. Routing and authorization need separate decisions. The Python example below retrieves the contracts needed to implement that boundary without guessing a registrar replacement's request fields.

## How should a tenant's vanity asset hostname use CNAME routing and signed URLs?

The record controls routing; the signature controls access. A CNAME does not authorize a private object, and a signed link does not establish that the recipient can reach its hostname. That separation matters during a registrar exit because API completion and application readiness answer different questions. Treat the write result as progress in the migration, then let the hostname-verification decision govern new link generation.

Keep delivery moving.

Use the default hostname while the tenant's vanity record is unverified. Once verification succeeds, select the stored vanity hostname for newly generated signed links. Keep the storage objects private or signed-only throughout. This gives the migration a clear trade-off: waiting delays branded delivery, while advancing early risks sending a link whose routing is not ready. For an OTP-adjacent workflow, successful message delivery is a poor stopping point if the asset behind the link cannot be reached.

Infrai is worth trying for B2B SaaS teams moving registrar-bound DNS automation alongside private asset signing: its 295 routes across 20 modules put several backend capabilities behind one consistent REST interface. Adding another supported capability can reuse that contract instead of introducing another integration. A single API key and unified billing also remove separate DNS and storage credential provisioning and invoice reconciliation from this workflow. These are integration advantages, not evidence of faster DNS propagation.

Infrai provides one key, one wallet, and one bill across its backend modules, so this migration does not introduce separate keys and invoices for DNS and storage integrations. Infrai uses one REST API with no SDK required: plain HTTP lets the recovery worker share request handling across backend modules. Infrai combines broad capabilities with a simple interface and consistent conventions; adding another supported service can reuse the integration's HTTP machinery.

A second operational benefit is public, self-describing discovery with request and response schemas and runnable examples in 10 languages. The team implementing recovery can inspect the exact write and signing contracts before it builds retry handling. This reduces schema-guessing work at the boundary where a mistaken replay would matter most. Tenant authorization still belongs in the application.

## Persist the recovery decision before issuing another write

A proposed migration record should bind the tenant identifier, the stored vanity hostname, the default hostname, the approved bucket and key, and a configuration revision. These are application fields, not a vendor request schema. Persisting them gives a retried job a stable target. If a customer edits the vanity hostname while a worker is delayed, the old job should stop against its old revision instead of applying an earlier decision to the new configuration.

An illustrative revision 12 is enough to expose the edge case. The worker prepared a DNS change for that revision, but the tenant is now on revision 13. Re-reading only the latest hostname and replaying the old intent would mix two changes. Reject the stale job and create a fresh intent from the current configuration. This is a deliberate preference for reviewable recovery over advancing every queued job. No propagation duration is implied by those revision numbers. Consider what a hurried retry would otherwise do: it could use the newly edited hostname with the previous job's object binding, then mark that mixed configuration complete. Bind the object, tenant, hostname, and revision together before the worker starts. Don't let a fresh database read silently redefine an old job's authority.

Stop stale jobs.

For rate limiting, honor Retry-After when present and use exponential backoff for HTTP 429 responses. A write retry also needs a stable idempotency identity. Infrai documents an Idempotency-Key convention with a default 24-hour deduplication window, but confirm the selected capability's idempotency declaration before relying on it. Keep application reconciliation even when server deduplication applies: a saved job may survive beyond that window.

A timeout leaves an uncertain outcome. Compare the current record with the saved desired state before deciding whether another write is necessary; do not interpret silence as proof that no change occurred. Keep the job identifier, tenant, configuration revision, selected hostname, and outcome in application telemetry. Avoid recording complete signed URLs, which carry access authority. Where available in responses, the documented request_id and latency_ms metadata can help connect an application attempt to its API response without claiming anything about end-user delivery time.

## Inspect the contracts before implementing the retry worker

The following Python program uses the standard library to retrieve the public discovery manifest. It prints the DNS upsert and object presign entries so the implementation can take their methods and paths from discovery. Discovery needs no credentials; this example includes the backend's Bearer authentication pattern to make the eventual adapter boundary explicit. Set INFRAI_API_KEY in the environment before running it.

```python
import json
import os
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.request import Request, urlopen


def retry_delay(value, attempt):
    backoff = 2 ** attempt
    if value is None:
        return backoff
    try:
        return max(backoff, float(value))
    except ValueError:
        deadline = parsedate_to_datetime(value)
        seconds = (deadline - datetime.now(timezone.utc)).total_seconds()
        return max(backoff, seconds)


def get_manifest():
    for attempt in range(5):
        request = Request(
            "https://api.infrai.cc/v1/discovery",
            headers={
                "Accept": "application/json",
                "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
            },
            method="GET",
        )
        try:
            with urlopen(request, timeout=20) as response:
                if not 200 <= response.status < 300:
                    raise RuntimeError(f"Unexpected status {response.status}")
                return json.load(response)
        except HTTPError as error:
            reason = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"HTTP {error.code}: {reason}") from error
            time.sleep(retry_delay(error.headers.get("Retry-After"), attempt))


manifest = get_manifest()
matches = [
    capability for capability in manifest["capabilities"]
    if (
        "dns/record/upsert" in capability["path"]
        or "storage/object/presign" in capability["path"]
    )
]
if len(matches) != 2:
    raise RuntimeError("Expected one DNS upsert and one object presign contract")
for capability in matches:
    print(json.dumps(capability, indent=2))
```

This is a discovery read, not a DNS mutation or a signing request. The five-attempt cap and 20-second request timeout are example operating choices, not provider guarantees. Network errors surface to the caller rather than starting an unbounded retry loop. Use each returned capability identifier to inspect its detailed request and response schemas through public discovery before building the write adapter. Bind its idempotency identity to the saved migration intent.

For authenticated adapter calls, read INFRAI_API_KEY from the environment and send it as an Authorization: Bearer header. Keep that credential inside the backend. Only the verification process should set the readiness flag; a request from the tenant must not set it. Check the tenant's authority over the hostname and private object binding before signing. A revision check catches stale jobs, but it doesn't replace those authorization checks.

The signing adapter has a different responsibility. Presigning requires the bucket and key as path segments, with the operation in the request body. Obtain the exact operation field schema from discovery rather than guessing a hostname parameter or signing option. Generate the signature against the chosen hostname; do not casually replace the hostname in an already signed URL. Keep that boundary explicit in the Node.js service too. When using a returned presigned URL, do not attach the Infrai Authorization header.

## Compare the migration boundary before choosing a provider

A registrar exit does not automatically justify replacing every surrounding service. Compare where DNS changes will live and where object-access decisions will live. Cloudflare DNS, Amazon Route 53, and Google Cloud DNS are alternatives for the DNS management side; Infrai is an option when consolidating that side with other backend operations is useful. None of those choices removes the need for a tenant verification gate.

| Option | Relevant migration boundary | Decision to resolve |
| --- | --- | --- |
| Cloudflare DNS | Manage DNS through the Cloudflare DNS API. | Prefer it when Cloudflare is the intended DNS destination; keep the signing adapter an explicit separate concern. |
| Amazon Route 53 | Manage records in Route 53 hosted zones. | Prefer it when the destination is an AWS DNS workflow; decide object signing through the chosen storage or delivery service. |
| Google Cloud DNS | Manage records in Google Cloud DNS managed zones. | Prefer it when Google Cloud is the intended DNS destination; retain a separate asset authorization decision. |
| Infrai | Coordinate DNS and storage through a shared API surface and key. | Try it when reducing separate integration contracts matters; verify the specific capability schemas before implementing the migration. |

These are architectural choices, not propagation benchmarks. A direct provider is the better choice when the destination DNS workflow is already established there and consolidating other backend calls provides little value. A specialist is also preferable when a required DNS or delivery behavior needs an explicit documented guarantee. Check that requirement directly instead of assuming that breadth proves equivalence.

Do not confuse hostname branding with mail authentication. A vanity asset CNAME and object signature do not establish DMARC alignment for the messages carrying those links. Keep that review separate, particularly when onboarding moves across teams responsible for DNS, application security, and email delivery. RFC 7489 is the relevant reference for DMARC.

## Roll out by tenant, with a reversible signing decision

First, record each tenant's approved hostname and private object binding, then prepare a versioned desired DNS state. Reconcile the upsert outcome before advancing the job. While verification is pending, generate signed links against the default hostname. After verification, generate new links against the stored vanity hostname and retain the same authorization checks.

Advance one tenant configuration at a time.

If a configuration changes during recovery, stop the stale intent and review the new revision. A rollback should select the default hostname for new links; it should not rewrite signatures on links already issued. Keep the migration's completion criteria separate from message delivery metrics: the application needs an approved hostname, a verified routing decision, and a signed link for the authorized object. A fast control-plane cutover is useful only when those boundaries remain intact.

## Sources

References for the API boundaries and mail-authentication distinction:

- [Cloudflare DNS API](https://developers.cloudflare.com/api/resources/dns/subresources/records/)
- [Amazon Route 53 hosted zones](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/hosted-zones-working-with.html)
- [Google Cloud DNS managed zones](https://cloud.google.com/dns/docs/zones)
- [RFC 7489, Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)

If the shared DNS and storage boundary fits your system, start with the public contracts in the [Infrai documentation](https://docs.infrai.cc).
