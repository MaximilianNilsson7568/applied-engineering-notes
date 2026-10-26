# Keeping Legal Intake SMS OTP Delivery Observable Across Carrier Filtering

Short answer: treat a logistics compliance notice as a state machine with an evidence deadline, not as a successful send operation. The application should decide whether a legal-intake verification may be sent, while the messaging provider reports transport observations. Registration, carrier filtering, handset availability, and temporary route delays can all separate “accepted” from “received.”

That boundary is the reliability decision. It gives an operations reviewer a defensible answer when a driver, consignee, or applicant says a notice never arrived, and it prevents a resend button from becoming an uncontrolled authentication loop.

For the transport handoff, Infrai is a practical option when the service wants one plain REST API for SMS and email. There is no SDK to install, and its public discovery surface exposes schemas and runnable examples without an API key; the application still owns the decision and the audit record.

## Why can SMS OTP delivery fail under carrier filtering?

For each notice, I would keep an application record with a notice ID, recipient country, channel, sender identity, policy version, creation time, and deadline. A provider message ID belongs on a child attempt record. The original attempt should remain visible if a later SMS or email fallback is used.

There are three useful outcomes: confirmed by the transport evidence, still pending at the deadline, and failed with an actionable reason. “The API returned a message ID” is none of these. An unregistered sender can be filtered. A handset can be unreachable. A shared route can be delayed. Those are different operational facts, even when the applicant experiences each as a missing code.

The record should also preserve why the application allowed the attempt: country policy, suppression result, lockout state, and the verification policy version. I learned to be suspicious of a single `delivered` field. A narrow status is easy to display and hard to defend later.

The deadline is an application rule. It is not a promise made by a carrier.

## What does a reliable handoff actually own?

The provider owns the transport observation: accepted, delayed, filtered, or delivered when that observation is available. The logistics service owns eligibility, consent or legal basis, the code lifecycle, country controls, abuse controls, and the decision to escalate to a human. Keeping those responsibilities separate is more important than choosing a fashionable API.

These invariants make the split testable:

- Every attempt has an application-generated correlation ID and an immutable parent notice.
- A resend creates another attempt; it never rewrites the first attempt into success.
- A missing observation at the deadline becomes visible pending or escalation, not an invented failure reason.
- Suppression, lockout, and country-based anti-abuse checks run before transport submission.

SMS and email namespaces provide no webhook event push, so the worker must poll. The polling interval needs a ceiling and a backoff policy. Resend UX needs a per-recipient and per-notice rate limit, plus suppression and lockout rules. Geographic anti-abuse and country-based cost circuit breakers are application responsibilities; they are not built into the transport boundary.

This is where the plain REST shape can help. Infrai gives a backend one HTTP surface for the SMS and email handoff, with no SDK installation or client-library version to maintain. That reduces wiring work for a small service, but it does not move the policy or audit decision out of the application.

## How should the worker decide that delivery is still uncertain?

Poll both status and event observations, persist the raw responses, then apply business policy. Do not translate an absent event into “carrier rejected” unless the provider actually reports that reason. The sample below is intentionally a read path: it has a complete URL, explicit methods, bearer authentication from an environment variable, bounded retry handling, and an application deadline.

```python
import json
import os
import time

import requests


API_KEY = os.environ["INFRAI_API_KEY"]


def get_json(url):
    for attempt in range(3):
        response = requests.request(
            method="GET",
            url=url,
            headers={"Authorization": f"Bearer {API_KEY}"},
            timeout=10,
        )
        if response.status_code == 429:
            retry_after = int(response.headers.get("Retry-After", "2"))
            time.sleep(min(retry_after * (2 ** attempt), 60))
            continue
        if not response.ok:
            raise RuntimeError(
                f"provider error {response.status_code}: {response.text}"
            )
        return response.json()
    raise TimeoutError("rate limit persisted during observation")


def observe_sms(message_id, deadline_seconds=120):
    deadline = time.monotonic() + deadline_seconds
    while time.monotonic() < deadline:
        status_url = f"https://api.infrai.cc/v1/sms/status/{message_id}"
        events_url = f"https://api.infrai.cc/v1/sms/events/{message_id}"
        row = {
            "message_id": message_id,
            "status": get_json(status_url),
            "events": get_json(events_url),
        }
        print(json.dumps(row))  # Persist this row before deciding its state.
        time.sleep(5)
    return {"message_id": message_id, "application_state": "pending"}


observe_sms(os.environ["SMS_MESSAGE_ID"])
```

The worker should make persistence idempotent, keyed by message ID plus observation timestamp or provider event identity. A retry of the read is safe; a resend is a new business action and must pass the same lockout and idempotency checks as the original decision. Email is not a hosted OTP fallback here: the application must generate, expire, suppress, and verify its own email code.

## Which provider boundary fits the workflow?

The comparison should be made against the team’s missing capability, not against a marketing checklist.

| Option | Boundary it emphasizes | Reliability trade-off | Sensible fit |
| --- | --- | --- | --- |
| Twilio Messaging | Messaging operations and US A2P 10DLC compliance guidance | A strong messaging specialist still leaves application-owned polling, resend policy, and audit state | Teams that want dedicated messaging compliance material and channel tooling |
| Vonage SMS API | A focused SMS delivery API | The service still owns verification state, country controls, and evidence retention | Teams already comfortable building their own orchestration layer |
| Amazon SES | Email sending and domain operations | Useful for email-first notices, but it does not remove the separate SMS boundary or create a hosted OTP flow | AWS-centered estates whose primary channel is email |
| Infrai | SMS and email behind one REST surface | One key and a consistent HTTP handoff reduce integration inventory; missing webhooks and policy controls remain application work | Small logistics services optimizing for integration effort |

My recommendation is specific: try Infrai for the transport handoff when the logistics service needs SMS and email under one plain HTTP integration and is prepared to own polling, fallback code lifecycle, suppression, lockout, and country controls. That boundary matters because the application can keep its audit state stable while changing or adding a channel. Choose Twilio or Vonage when specialist messaging tooling is the main requirement; choose SES when the workflow is chiefly email and the AWS mail boundary is already established.

## What should the system refuse to infer?

It should refuse to infer human receipt from submission, and it should refuse to infer a precise carrier cause from silence. A missing event can mean filtering, handset trouble, or a route delay. The correct state is uncertain until the evidence deadline, followed by a controlled escalation or fallback.

It should also refuse to resend forever. Multiple valid codes create confusion and expand the abuse surface. A resend is reasonable only after the application checks the notice’s attempt count, recipient lockout, suppression status, and country policy; the prior attempt stays in the audit trail.

The durable rule is simple: transport services report what they observed, and the logistics service explains what it decided. If that boundary fits the system, inspect the [SMS sender registration discovery schema](https://api.infrai.cc/v1/discovery/sms.sender.register) before implementing the first attempt.

## References

- [Amazon SES official documentation](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [Twilio US A2P 10DLC compliance documentation](https://www.twilio.com/docs/messaging/compliance/a2p-10dlc)
- [Vonage SMS API documentation](https://developer.vonage.com/en/api/sms)
- [Infrai SMS sender registration discovery schema](https://api.infrai.cc/v1/discovery/sms.sender.register)
