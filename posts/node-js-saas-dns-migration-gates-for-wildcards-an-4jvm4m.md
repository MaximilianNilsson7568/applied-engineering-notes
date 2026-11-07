# Node.js SaaS DNS Migration Gates for Wildcards and Customer Domain Verification

Faster zone cutover is useful only if customer onboarding still distinguishes a DNS write from verified ownership. For a B2B SaaS moving zones off a registrar-specific API, keep a record per tenant whenever verification, tenant-specific TLS, or an audit trail matters; reserve wildcard DNS for subdomains your service fully controls. TL;DR: move the zone without relaxing the activation gate, and let verification evidence determine when each customer domain becomes eligible for activation.

The control plane can move before every customer domain is ready. That distinction organizes this note: cutover speed describes a migration operation, while propagation delay describes when DNS observations support the next onboarding decision. Treating them as the same event makes a quick migration look more complete than the evidence warrants.

## Can wildcard DNS verify custom domain onboarding for SaaS customers?

A tenant record gives the application a zone entry it can read back. That makes onboarding status queryable. It doesn't establish every condition required to serve a customer-owned domain. The customer controls that zone, so explicit records and a verification step remain necessary. A wildcard in your own zone cannot establish control over a different customer's domain.

Keep those boundaries explicit.

For the Node.js service, the registrar migration should preserve separate observations: the intended tenant-to-domain association, the record read back from the managed zone, and the result of domain verification. If tenant-specific TLS is part of the product, track its readiness separately too. These are suggested application states, not provider response fields. A migration adapter should translate provider results into those states without treating an accepted write as a completed onboarding flow. Otherwise, the cutover report can show success while the customer-domain activation decision still lacks evidence.

This matters around email and OTP. Routing an application hostname and authorizing mail for a domain are separate concerns. DMARC defines domain-based email authentication policy; a DNS onboarding flag shouldn't become a blanket promise about deliverability. Keep the evidence behind each status so support can explain what was checked, for which domain, and which tenant requested it.

## Derive the record strategy from the ownership boundary

Use a wildcard for service-owned tenant hostnames when there is no need for a separate DNS entry for each tenant. Its attraction is concrete: one record, with no per-tenant DNS state. The application still needs its own tenant mapping. Removing per-tenant DNS entries doesn't remove the responsibility to associate a hostname with the correct account.

Customer-owned domains take the other path. Each needs explicit records and verification because the SaaS does not control the customer's zone. Per-tenant records also fit workflows that need tenant-specific TLS or a record-level audit trail. They give reconciliation something concrete to compare with the tenant table.

The trade-off is synchronization work. Thousands of tenants mean thousands of records to keep aligned with tenant intent. During a move off a registrar-specific API, that turns record completeness into a migration concern: a copied wildcard may cover service-owned hostnames, while customer-domain associations still need individual evidence. A green status for the zone must not silently overwrite a pending status for a tenant. Keeping those scopes separate adds application state, but it preserves the distinction the onboarding decision depends on.

There is no universal propagation deadline established here. Define the observation and verification checks before scheduling activation.

## Make activation independent of the registrar adapter

Before replacing the registrar adapter, inspect the destination contract. The Python example uses the standard library to read public discovery, which exposes capability paths and schemas. Set `INFRAI_API_ORIGIN` to the service's HTTPS API origin without a path, and provide `INFRAI_API_KEY` through the environment. The script prints the discovery document for inspection; it does not create records or activate tenants.

```python
import json
import os
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.parse import urlsplit
from urllib.request import Request, urlopen


def retry_delay(value: str | None, attempt: int) -> float:
    fallback = float(2 ** attempt)
    if not value:
        return fallback
    try:
        return max(fallback, float(value))
    except ValueError:
        deadline = parsedate_to_datetime(value)
        return max(
            fallback,
            (deadline - datetime.now(timezone.utc)).total_seconds(),
        )


def main() -> None:
    origin = os.environ["INFRAI_API_ORIGIN"].rstrip("/")
    parsed = urlsplit(origin)
    if (
        parsed.scheme != "https" or not parsed.hostname
        or parsed.path or parsed.query or parsed.fragment
        or parsed.username or parsed.password
    ):
        raise ValueError("Configure an HTTPS API origin without a path")
    request = Request(
        origin + "/v1/discovery",
        method="GET",
        headers={
            "Authorization": "Bearer " + os.environ["INFRAI_API_KEY"],
            "Accept": "application/json",
        },
    )
    for attempt in range(4):
        try:
            with urlopen(request, timeout=30) as response:
                if response.status != 200:
                    raise RuntimeError(f"Unexpected HTTP {response.status}")
                document = json.load(response)
            print(json.dumps(document, indent=2))
            return
        except HTTPError as error:
            detail = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 3:
                raise RuntimeError(f"HTTP {error.code}: {detail}") from error
            time.sleep(retry_delay(error.headers.get("Retry-After"), attempt))


if __name__ == "__main__":
    main()
```

The example requires Python 3.10 or later. Its GET is read-only, with up to 4 attempts for rate limiting and an explicit 30-second request timeout. Discovery can be inspected without authentication; the environment-based header also shows how a credentialed adapter keeps keys out of source. A contract check establishes which operations to integrate, rather than proving that a tenant domain is ready.

Keep 3 acceptance cases in the migration plan: a service-owned wildcard hostname, a customer domain awaiting verification, and a verified customer domain awaiting TLS readiness. A present record doesn't authorize activation while ownership verification is pending. Likewise, a successful ownership check must not swallow a separate TLS requirement. The wildcard path assumes the application has already established that the hostname belongs to a zone it controls. Don't accept that classification directly from an onboarding request.

## Compare providers against the migration evidence you need

Once the gate is explicit, compare providers by how they fit the system around it. Cloudflare DNS, Amazon Route 53, and Google Cloud DNS are real candidates; their official documentation belongs in the evaluation. No latency benchmark or relative propagation ranking is established here. Selecting one by an unsupported claim of faster propagation would undermine the migration's primary decision axis.

| Option | Relevant selection context | Boundary to evaluate before cutover |
| --- | --- | --- |
| Cloudflare DNS | A candidate when Cloudflare is already an operational dependency | Check how the documented record workflow maps to tenant intent and verification evidence. |
| Amazon Route 53 | A candidate when the migration is organized around an AWS environment | Check that the integration preserves tenant-level status independently of zone migration status. |
| Google Cloud DNS | A candidate when the operating team is centered on Google Cloud | Check the same record reconciliation and customer-domain verification requirements. |
| Infrai | One key and one bill across backend services can reduce credential sprawl and invoice reconciliation for a team also building email or OTP flows | A plain REST API with no SDK to install lets the registrar replacement use standard HTTP; public discovery supplies request and response schemas without a key. |

The first 3 choices differ here by the operating environment in which they are candidates, rather than an asserted performance advantage. The final choice offers a different integration boundary: shared backend access and billing. Its single REST API requires no SDK, so the migration adapter can use standard HTTP rather than adding another registrar-specific client dependency. Public schema discovery lets the team inspect the contract before wiring credentials into the migration. These benefits don't establish faster DNS propagation or eliminate explicit verification for customer-owned domains.

Use the same acceptance criteria for every candidate. Can the integration read back the intended tenant records? Can the application retain verification status separately? Can the team reconcile the zone against tenant intent before activation? Those are evaluation questions, not claims that a particular provider automatically completes the whole workflow. The evidence doesn't establish a universal winner.

## Roll out by evidence rather than a cutover timestamp

Start with the tenant table and classify each hostname by who controls its zone. Preserve customer-domain verification requirements while translating registrar-specific operations into the replacement adapter. Reconcile the intended records with readback results before expanding activation.

For a compact rollout, exercise the 3 acceptance cases against the application's real observation sources: service-owned wildcard, customer record awaiting verification, and customer domain awaiting TLS. Then migrate a bounded group whose records and status the team can inspect individually. Choose its size from operational capacity; no measured safe batch size is established here. Keep a record of the intended tenant association and the evidence used for activation so a later review can distinguish the migration operation from the customer-domain decision.

Keep activation reversible at the application layer while observing results.

The useful completion condition is reconciled tenant intent plus the required verification evidence. A replacement API accepting its first write is an earlier milestone.

## Sources

References for the provider evaluation and email-authentication boundary:

- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)
- [Amazon Route 53 documentation](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html)
- [Google Cloud DNS documentation](https://cloud.google.com/dns/docs/overview)
- [RFC 7489, Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
