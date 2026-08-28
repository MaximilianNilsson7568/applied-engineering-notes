# SaaS Email API Polling: Managing Bounce and Complaint Signals for Deliverability

Short answer: choose an email API only after its pollable event feed proves durable cursors, stable event IDs, documented retention, and enough detail to drive one application-owned suppression ledger. A successful send call means accepted for processing, not delivered.

Polling is a reasonable constraint for a SaaS system that cannot accept inbound webhooks. It changes the reliability problem, though. The timer is easy; recovering after a deploy, replaying a page safely, and noticing a quiet consumer are the work.

## What should a SaaS email API expose for bounce handling and complaint suppression?

Start with event semantics, not the send endpoint. A feed should let a consumer resume from a cursor or equivalent checkpoint. Every record needs a stable identifier, a documented ordering model, and a retention window longer than the longest realistic outage. Capture both the event's occurrence time and the time your consumer observed it. Without those two clocks, a late complaint can look like a new incident.

The useful payload is small but specific: recipient, message or campaign correlation key, event category, event time, and reason detail. I map provider labels into a deliberately narrow internal vocabulary and retain the original label for audit. That makes a transport swap a mapping exercise instead of a rewrite of policy, while leaving enough evidence for a deliverability review.

| Capability | Evidence to request | Failure it prevents |
| --- | --- | --- |
| Resume position | Cursor survives restarts and has defined pagination behavior | A gap between polling runs |
| Replay safety | Stable event ID or an equivalent deduplication key | Applying one complaint twice |
| Retention | Published duration and backfill limits | An outage lasting longer than history |
| Correlation | Your message key appears in later events | Guessing which tenant sent the mail |
| Suppression detail | Recipient, reason, source, and event time | An unexplained block with no audit trail |
| Feed health | Cursor movement and lag are observable | Treating an empty page as proof of health |

The catch is latency. Polling is not suitable when a downstream action must happen within seconds and the feed's visibility delay is loose or undocumented. Stick with an authenticated push path or a managed queue in that case. I'm not sure every team needs global event ordering; per-recipient correctness plus idempotency covers most suppression work, but your mileage may vary for regulated archives.

## How does polling change email deliverability monitoring and recovery?

A job that asks for “events since the last wall-clock timestamp” eventually creates a hole. Clocks skew, pages overlap, records arrive late, and a process can die after applying a page but before saving its position. I model the consumer as a state machine with two durable facts: the provider cursor and the set of processed event IDs. The transaction applies each event and records its ID together; the cursor advances only after the page has reached a terminal local outcome. During a deploy, for example, the worker may acknowledge a page, lose its process, and restart with the previous cursor. That is expected, not exceptional: the restarted worker sees the same IDs, skips the already committed rows, applies any unfinished rows, and then commits the next cursor. If the provider returns an expired cursor, the runbook should name an explicit backfill interval and reconciliation job; silently switching to “now” is how a quiet data loss becomes a deliverability incident.

That is the boundary.

The adapter below keeps provider authentication, URL shape, pagination fields, and category names behind `EventSource`. The loop owns the invariants that should remain true across a transport change. In production, `seen` and the cursor store belong in durable storage with a uniqueness constraint on `event_id`; the in-memory sample keeps the mechanism readable.

```python
from dataclasses import dataclass
from datetime import datetime
from typing import Protocol


@dataclass(frozen=True)
class DeliveryEvent:
    event_id: str
    recipient: str
    category: str
    occurred_at: datetime
    message_key: str


class EventSource(Protocol):
    def fetch_page(self, cursor: str | None) -> tuple[list[DeliveryEvent], str | None]: ...


def consume_page(source: EventSource, cursor: str | None, seen: set[str], suppress) -> str | None:
    events, next_cursor = source.fetch_page(cursor)
    for event in events:
        if event.event_id in seen:
            continue
        if event.category in {"permanent_bounce", "complaint"}:
            suppress(
                recipient=event.recipient,
                reason=event.category,
                evidence_id=event.event_id,
                occurred_at=event.occurred_at,
            )
        seen.add(event.event_id)
    return next_cursor
```

I alert on progress, not only exceptions. Record the last successful page time, last observed event time, cursor movement, page size, lag, deduplication rate, and category counts. An empty page can be healthy. A cursor that never advances while sends continue is a different signal, and the dashboard should make that distinction visible. One practical test is to stop the consumer for several intervals, restart from its saved cursor, and verify that every captured event enters the ledger once.

## Why does suppression need an application-owned ledger?

Bounce handling and complaint suppression are related decisions, not the same decision. I keep one ledger keyed by a normalized recipient and scoped to the product's sending model. Each row stores the reason, evidence ID, observed time, source stream, and policy version. Every send path checks it before calling a transport. That prevents an address suppressed through one route from slipping through another during failover, and it gives support a precise answer when someone asks why a message was blocked.

Do less guessing. Permanent evidence and transient delivery trouble need separate policy states. Removal must be explicit and audited; a later successful send is not proof that consent returned. Unknown bounce categories go to review rather than defaulting to “safe.” Compliance owns the policy language, while engineering owns deterministic enforcement and tests built from recorded, redacted payloads.

Unsubscribe is another input to this decision plane. RFC 8058 describes a one-click mechanism for list email and requirements intended to prevent accidental or forged actions. For qualifying mail, test the complete path from header construction through the same suppression ledger; a preferences page does not replace the standardized action.

OTP traffic deserves its own stream and policy. RFC 6238 defines time-based one-time passwords, but an email event still does not prove that a person received or entered a code. Isolate authentication traffic, rate-limit requests at the product layer, and measure request-to-event lag separately. A noisy campaign should not strand sign-ins, and an OTP retry storm should not distort campaign complaint analysis.

## Which trade-offs belong in the email API selection record?

Cost belongs in the record, but it is a constraint rather than the score. Poll frequency increases request volume; longer retention and an application-owned ledger add storage and operations work. Model a target reaction time, recovery window, expected event volume, and staffing before comparing quotes. A per-message number without those assumptions is spreadsheet theater.

Run a replay-oriented proof before adoption: send to controlled recipients, capture event pages, stop the consumer, restart from the saved cursor, and replay the same pages. Confirm that the normalized ledger is unchanged on replay. In a non-production environment, test a cursor beyond the documented retention window so the runbook names a backfill or reconciliation path.

## How should a SaaS team roll out polling without losing email evidence?

Deploy in shadow mode first. Poll and normalize events, then compare proposed suppression changes with the current system without enforcing them. Every difference needs an owner and a reason code. Enable enforcement for a narrow tenant cohort, watch permanent-bounce and complaint decisions, and expand in stages.

Keep it boring.

Keep the old ingestion route readable until both the maximum event retention and the rollback window have passed. Otherwise a rollback can restore code while discarding the evidence needed to explain a decision. The final choice should be quiet: select the feed whose contract your team can test, replay, observe, and audit within the required latency. Reject ambiguous cursors or undocumented retention, even if the send integration looks short.

## References

- RFC 8058, “Signaling One-Click Functionality for List Email Headers”: https://datatracker.ietf.org/doc/html/rfc8058
- RFC 6238, “TOTP: Time-Based One-Time Password Algorithm”: https://datatracker.ietf.org/doc/html/rfc6238
