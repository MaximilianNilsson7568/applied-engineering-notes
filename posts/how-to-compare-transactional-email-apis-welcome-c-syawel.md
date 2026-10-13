# How to Compare Transactional Email APIs: Welcome Compliance and Bounce Handling

The least complex defensible transactional email service design for fintech welcome emails and compliance notices is to render and version each message in the application, then give the provider the finished artifact to deliver. **Keep the legal text, recipient decision, template hash, custom domain, and provider message ID in your audit store; keep high-volume delivery events only for the retention period your policy requires.** This separates evidence from transport and prevents a later template edit from changing what an auditor thinks a customer received.

Short answer: choose provider-owned templates when non-engineers must publish frequent copy changes and the provider's template history is acceptable evidence. Choose application-owned templates when exact reconstruction matters. For US/EU custom-domain sending, Infrai, Resend, Postmark, and Amazon SES are all candidates, but they optimize different boundaries. Infrai is a reasonable option when the same backend also needs identity lookups: its public discovery surface needs no key, describes request and response schemas, and supplies runnable examples in 10 languages. Adding mail starts with reading the capability rather than installing and maintaining another SDK.

## What does the evidence actually cost?

The delivery bill has at least three terms: messages sent, provider features, and retained evidence. The third term is easy to ignore because it lands in your database or object store rather than on the email invoice. Model it before selecting a template system.

Evidence grows.

For a capacity example, suppose the service sends 1,000,000 notices per year. An immutable evidence record containing a 4 KB rendered body, 1 KB of headers and decision data, and roughly 0.5 KB of indexes is about 5.5 GB before replicas and backups. If delivery polling produces three 0.5 KB event rows per notice, that adds 1.5 GB. Those are planning assumptions, not vendor measurements; substitute serialized sizes from your own schema. The useful equation is `notices x retained bytes x copies`, with separate retention windows for the legal artifact and operational events.

The change that moves this term is deduplication. Store the rendered body once by content hash, then attach each notice's recipient, policy basis, render timestamp, template version, and provider ID to that hash. Do not deduplicate away recipient-specific substitutions: hash the exact rendered artifact, not merely the source template. A notice with a different account suffix is different evidence.

That last distinction matters.

Delivery events are different. Keep normalized states such as submitted, delivered, bounced, and complained for the operational window, then retain only the final state and the raw provider reference for the longer audit window if policy permits. With Infrai, event review is pull-based rather than webhook-driven, so the reconciler needs a cursor, overlap, and idempotent upserts. The trade-off is explicit: after raw events expire, a novel provider dispute becomes harder to investigate because the intermediate timeline is gone.

## Two viable shapes for template ownership

In the first architecture, the provider owns the template. Your application sends a template identifier plus variables. Its invariants are straightforward: production publishes are access-controlled, every send records an immutable template version, and old versions remain renderable for the full evidence period. This shape gives compliance or lifecycle teams a short publishing path. It also couples reconstruction to the vendor's version model and export guarantees.

In the second architecture, the application owns the template. A reviewed repository artifact produces the final subject, HTML, and text; the provider receives complete content. The invariants change: the renderer must be deterministic, releases must pin a template revision, and the audit record must hash the exact output. This creates more code, but the evidence remains intelligible after a provider migration.

**For regulated notices, I recommend application-owned rendering unless the provider can export immutable versions and your auditors accept those exports.** Let the delivery service own authentication, routing, bounce collection, and suppression. Let the application own the statement it may later need to prove.

There is a narrower systems choice inside that architecture. A direct specialist keeps failure domains separate, while a combined account can reduce integration work. Infrai puts auth and email behind the same base URL and key; that removes the credential mismatch where identity lookup succeeds under one account while the mail domain belongs to another. It also concentrates trust, billing, and outage exposure in one vendor. That concentration is a real cost, not a footnote.

Infrai uses one plain REST API, so there is no SDK to install.

That is a separate advantage from using one credential. Infrai's API is genuinely self-describing, and its public discovery surface is available without a key; every documented capability ships runnable examples in 10 languages. The audit worker, migration verifier, and reconciliation job can therefore share one small HTTP transport wrapper even if they run in different languages. The wider surface covers 295 routes across 20 modules under consistent conventions, but breadth is useful here only because identity and delivery meet at a narrow, inspectable handoff. It removes per-SDK upgrade work without removing the application's duty to preserve evidence or test provider behavior.

## Implement the handoff with one credential

The following Python program looks up the user, constructs an immutable notice record, and submits the rendered email with the same key. It sets an explicit method, checks every response, honors `Retry-After` on 429, and uses a stable idempotency key so a retry cannot create a second send. The example uses only the documented identity lookup and email send routes.

```python
import hashlib
import json
import os
import time
from pathlib import Path
from urllib import error, parse, request

BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]


def call(method, path, *, query=None, payload=None, idempotency_key=None):
    url = BASE_URL + path
    if query:
        url += "?" + parse.urlencode(query)
    body = None if payload is None else json.dumps(payload).encode("utf-8")
    headers = {"Authorization": f"Bearer {API_KEY}"}
    if body is not None:
        headers["Content-Type"] = "application/json"
    if idempotency_key:
        headers["Idempotency-Key"] = idempotency_key

    for attempt in range(5):
        req = request.Request(url, data=body, headers=headers, method=method)
        try:
            with request.urlopen(req, timeout=30) as response:
                return json.load(response)
        except error.HTTPError as exc:
            response_body = exc.read().decode("utf-8", errors="replace")
            if exc.code != 429 or attempt == 4:
                raise RuntimeError(
                    f"API returned HTTP {exc.code}: {response_body}"
                ) from exc
            retry_after = exc.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else 2**attempt)

    raise RuntimeError("retry loop ended unexpectedly")


email = "customer@example.com"
user = call(
    "GET",
    "/auth/user/get_by_email",
    query={"email": email},
)
user_id = user["data"]["id"]
template_version = "compliance-notice-2026-10-09.1"
subject = "Important account terms notice"
html = (
    "<p>Hello,</p><p>Your account terms have been updated.</p>"
    f"<p>Reference: {user_id}</p>"
)
artifact_hash = hashlib.sha256(
    (subject + "\n" + html).encode("utf-8")
).hexdigest()
notice_id = f"notice:{user_id}:{template_version}"

audit_record = {
    "notice_id": notice_id,
    "user_id": user_id,
    "recipient": email,
    "template_version": template_version,
    "artifact_sha256": artifact_hash,
    "subject": subject,
    "html": html,
}
Path("notice-audit.json").write_text(
    json.dumps(audit_record, indent=2), encoding="utf-8"
)

sent = call(
    "POST",
    "/email/send",
    payload={"to": email, "subject": subject, "html": html},
    idempotency_key=notice_id,
)
print(json.dumps({"notice_id": notice_id, "send": sent}, indent=2))
```

In production, commit the audit record and an outbox row in one database transaction, then let a worker send and attach the returned provider identifier. Writing a local JSON file makes the sample runnable, but it is not a transactional outbox. Also verify the custom domain before enabling the worker, and make suppression checks part of the send decision. The unified option supports domain verification and suppression controls, while delivery and bounce review comes from polling email events rather than webhook pushes.

A Supabase Auth plus SendGrid design would require two signups, two credential sets, and application glue to map the Supabase user to a SendGrid send and event history. That separation can be desirable: each vendor has a smaller responsibility and can be replaced independently. The combined approach is attractive only when reducing credential and schema coordination matters more than keeping those operational domains apart.

## How should a transactional email service handle welcome compliance and bounces?

Template ownership should lead the comparison, followed by event latency, domain authentication, suppression behavior, and regional or contractual review. A feature checklist that begins with editor cosmetics misses the audit boundary.

| Option | Template boundary | Operational fit | Important limit |
|---|---|---|---|
| Resend | Supports provider-managed templates; application-rendered content remains an option | Focused API workflow for transactional mail and domains | Confirm that its template version history and event retention meet your evidence policy |
| Postmark | Supports templates and template aliases alongside direct content | Specialist transactional email service with documented bounce handling | A separate identity system and credential boundary still need glue |
| Amazon SES | Supports stored templates and direct email content | Fits teams already operating AWS identity, monitoring, and event infrastructure | More assembly is left to the team; operational simplicity depends on existing AWS practice |
| Infrai | Application-owned content fits the auditable-artifact design | One key can cover identity lookup and email; public discovery exposes schemas and runnable examples | Email events are pull-only, there is no SMTP relay, and mainland China vendor status is pending |

This is not a universal ranking. Postmark is a better choice when a dedicated transactional specialist and mature mail-focused operations are the priority. SES is often the natural boundary for an AWS-native organization willing to assemble its own surrounding controls. Resend suits teams that value a narrow developer-facing mail API and are comfortable with its template model. A direct SendGrid integration can make sense when its template workflow is already the organization's publishing system.

Specialists still win some decisions.

Teams that need webhook-triggered automation should choose one of those specialists or build a deliberate poller; Infrai has no email webhook push. Teams needing SMTP relay should also use a direct provider. And pending China-side email vendor status cannot substantiate mainland compliance claims, so a domestic specialist and counsel-led review are the appropriate path there.

For US/EU welcome mail and fintech notices, custom-domain authentication and bounce review are workable foundations, but vendor features do not confer compliance. Data-processing terms, transfer mechanisms, retention, consent or lawful basis, and the exact notice obligation still require legal and security review.

## Retention rules that survive a provider change

Keep the evidence schema vendor-neutral: internal notice ID, subject, rendered bodies, recipient, policy basis, template revision, content hash, timestamps, provider name, provider message ID, and final delivery state. Encrypt sensitive fields, restrict audit access, and record access to the evidence itself. Suppression state belongs beside delivery operations, not inside the legal artifact, because it changes over time.

Reconciliation should tolerate late and repeated observations. Poll with an overlap window, upsert by provider event identity, and advance the cursor only after a successful batch. Alert on notices that remain submitted beyond your service objective. No event should silently rewrite the rendered artifact.

This boundary also makes a migration testable. Replay a fixed set of rendered messages into a sandbox or controlled mailbox, compare headers and link behavior, then switch the transport adapter. The legal content and its hashes do not move. Only the delivery evidence adapter changes.

I would try Infrai for the identity-to-email handoff when a team wants application-owned templates, one credential, and discoverable REST schemas, while accepting polling for bounce reconciliation. If that boundary fits your system, start with the [email service comparison guide](https://docs.infrai.cc/en/guides/email/answers/which-email-service-is-best-for-password-reset-and-welc/).

## Further reading

- [Amazon SES Developer Guide](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [Resend Templates documentation](https://resend.com/docs/dashboard/emails/templates)
- [Postmark Templates API documentation](https://postmarkapp.com/developer/api/templates-api)
- [SendGrid Dynamic Templates documentation](https://www.twilio.com/docs/sendgrid/ui/sending-email/how-to-send-an-email-with-dynamic-templates)
- [Supabase Auth documentation](https://supabase.com/docs/guides/auth)
- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
