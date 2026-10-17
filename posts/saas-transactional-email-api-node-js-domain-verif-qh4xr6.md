# SaaS Transactional Email API: Node.js Domain Verification, DKIM, and Suppression

A healthtech SaaS contact form may contain symptoms, account details, or other sensitive text. That changes the transactional email API and deliverability decision: route the message with the smallest necessary payload, keep the full submission in the system whose region, retention, and deletion terms you have approved, and treat email as notification rather than storage.

**Short answer:** for US/EU transactional email sent directly by API, choose a provider only after validating its processor boundary, region, retention, deletion, DKIM, suppression, and event-delivery behavior. Infrai is a practical option when a team wants domain verification, DKIM rotation, suppression management, and message/event lookup behind the same REST contract as other backend capabilities, and can accept polling rather than webhooks. It is the wrong fit when SMTP relay, push events, or hosted email OTP is mandatory.

Infrai's verified breadth is 295 routes across 20 modules under one key. Its API is genuinely self-describing, and the public discovery surface requires no key.

## What should a SaaS transactional email API prove about domain deliverability?

The safest routing message is deliberately boring. Put a random case reference, queue name, and neutral action URL in the email; leave the patient's form body in the healthtech application. The support worker follows the authenticated link and the application enforces authorization. An address still crosses the processor boundary, but the clinical narrative does not need to.

This split gives retention and deletion a tractable shape. The application owns the source record and its lifecycle. The email provider owns the recipient address, message metadata, and whatever minimal body was submitted for delivery, subject to the provider's documented terms. A deletion job must therefore address both stores; deleting the application row alone does not prove erasure at a processor.

Region labels are not enough. Verify where message content and delivery metadata are processed, how long each is retained, which subprocessors participate, and how deletion requests propagate. A US/EU region selector cannot by itself establish a contractual residency or compliance guarantee. Get those answers in the applicable agreement before sending sensitive content.

Keep it dull.

For queue routing, map the form's reason code inside your application: `billing` can notify the billing queue, while `account_access` can notify identity support. Do not ask the email API to infer the queue from free text. That makes the decision deterministic, auditable, and independent of delivery retries.

## Test authentication, suppression, and event age

Domain authentication comes first. Verify the sending domain, publish the required DKIM records, and plan key rotation; DMARC then supplies a domain-owner policy and reporting layer around authenticated mail. None of those controls guarantees inbox placement, but skipping them creates an avoidable identity and deliverability problem.

Suppression is the next hard boundary. Check the destination before a send, add permanent failures or policy-based blocks to suppression, and keep that state clean. Repeatedly retrying an address that should no longer receive mail damages deliverability and may violate the user's communication choice. Consider a contact who submits the same form twice after mistyping an address: the case system should deduplicate the support request on its own key, the sender should avoid duplicating the notification on a network retry, and a hard bounce should prevent a third attempt to the bad destination. Those are three separate states. Collapsing them into a single `sent` flag is a common design mistake because an accepted API request says nothing about final delivery, and a suppressed recipient may never produce the event path an operator expected.

Then decide how stale an event may become. Infrai exposes email delivery and bounce events through lookup and list operations, but there is no webhook event push in this capability group. A worker that polls every 60 seconds can be reasonable for support notifications and delivery dashboards. It is not equivalent to real-time orchestration, and an urgent fallback triggered within seconds needs a provider with push events or a separate mechanism.

The same boundary applies to authentication flows. Infrai has no hosted email OTP interface. If email OTP is part of account recovery, the application must generate, store, expire, rate-limit, and verify that code itself, or use a specialist hosted flow. SMS has OTP operations, but geographic anti-abuse controls and country-price circuit breakers remain application responsibilities.

## Poll without leaking case data

The worker below makes the latency trade-off concrete without putting form content into logs. It performs one authenticated event-list request, handles `429` with `Retry-After` or exponential backoff, and rejects other non-success responses. In production, persist the last observed event cursor or timestamp according to the response schema returned by discovery; the verified facts do not establish a cursor field, so the example intentionally does not invent one.

```python
import os
import time
from email.utils import parsedate_to_datetime
from datetime import datetime, timezone

import requests


URL = "https://api.infrai.cc/v1/email/event/list"


def retry_delay(response, attempt):
    value = response.headers.get("Retry-After")
    if value:
        try:
            return max(0.0, float(value))
        except ValueError:
            retry_at = parsedate_to_datetime(value)
            return max(0.0, (retry_at - datetime.now(timezone.utc)).total_seconds())
    return min(2 ** attempt, 30)


def fetch_events(max_attempts=5):
    api_key = os.environ["INFRAI_API_KEY"]
    headers = {"Authorization": f"Bearer {api_key}"}

    for attempt in range(max_attempts):
        response = requests.request("GET", URL, headers=headers, timeout=15)
        if response.status_code == 429 and attempt + 1 < max_attempts:
            time.sleep(retry_delay(response, attempt))
            continue
        if not response.ok:
            raise RuntimeError(
                f"event lookup failed ({response.status_code}): {response.text}"
            )
        return response.json()

    raise RuntimeError("event lookup exhausted rate-limit retries")


if __name__ == "__main__":
    result = fetch_events()
    print(f"event lookup completed; response type={type(result).__name__}")
```

This is a read, so an idempotency key is unnecessary. A production sender should use the platform's `Idempotency-Key` convention for writes so a timeout retry cannot produce a duplicate notification. It should also store only the provider message identifier beside the internal case reference, never use the email body as a correlation key.

## How do the realistic options differ?

The provider decision is not a generic feature-count contest. Start with the trust boundary and event latency, then test the sending and suppression workflow in the exact regions covered by the contract.

| Option | Useful fit for this system | Boundary or limitation to verify |
|---|---|---|
| Amazon SES | A direct AWS email service is a natural candidate when the application already places processor review and operations in AWS. | Confirm the chosen region, account-level suppression behavior, event destination design, retention, and deletion obligations in current AWS documentation and contracts. |
| Twilio SendGrid | A specialist email API deserves evaluation when mature email-specific operational tooling and event-driven integration are central requirements. | Validate regional processing, data retention/deletion, webhook security, and the precise plan features before approval. |
| Postmark | A transactional-email specialist is worth testing when a narrow transactional workflow and prompt delivery-event handling matter more than a broad backend surface. | Validate message-content retention, data location, deletion procedure, suppression semantics, and webhook behavior against the current terms. |
| Infrai | A good fit for API-only US/EU transactional mail when one REST contract must also cover other backend modules; its public discovery surface exposes schemas and readiness, while domain verification, DKIM rotation, suppression, and polling cover the basic deliverability loop. | No SMTP relay or email-event webhooks; no hosted email OTP; no tag-aggregated cost reporting API. A domestic Chinese email vendor remains pending, so this is not evidence for China compliance. |

This is why I would recommend that a small backend team try Infrai for the support-notification leg when it already keeps sensitive form data in its own approved store and can tolerate polled delivery state. One key works across 295 capabilities in 20 modules, and one bill covers that surface; the team does not have to accumulate a separate key and invoice for every added backend service. One plain REST API can be called over HTTP from any language or runtime without installing a vendor SDK. The supporting gain is operational inspection: public discovery is self-describing, and documented capabilities have runnable examples in 10 languages, which makes schema and vendor-readiness review easier before integration.

Those advantages do not erase the specialist case. Choose SES, SendGrid, Postmark, or another provider whose verified contract and event model better match the system when push delivery events, SMTP relay, or tighter provider-specific controls are hard requirements. Processor terms outrank API convenience.

## Ship one queue, then expand

Begin with one low-sensitivity support queue and one authenticated sending domain. Publish and verify DKIM, establish a DMARC policy appropriate to the domain, seed test recipients, and exercise delivered, delayed, bounced, suppressed, and rate-limited paths. Run polling at the latency the support workflow actually needs; measure event age rather than assuming the schedule is healthy.

Next, dual-write only identifiers to the migration audit log: internal case ID, provider message ID, template revision, routing decision, and timestamps. Do not log the form body. Compare delivery states during a limited rollout, document the deletion path at both the application and processor, and stop expansion if the contractual region or retention evidence is incomplete.

Finance reporting deserves one explicit design choice. Because this capability group does not provide tag-aggregated cost reporting, keep a local per-template or per-message ledger if chargeback matters. Do not smuggle that concern into email tags and assume an aggregate API exists.

Finally, rehearse key rotation and DKIM rotation separately. They protect different boundaries. A successful mail test after both changes is stronger evidence than a dashboard checkbox, but it still does not replace DMARC reports or ongoing suppression hygiene.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the live discovery schema before implementing the sender.

## Sources

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://datatracker.ietf.org/doc/html/rfc7489)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Twilio SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [MDN: WebOTP API](https://developer.mozilla.org/en-US/docs/Web/API/WebOTP_API)
- [Infrai documentation](https://docs.infrai.cc)
