# Reconciling Seller Orders Through SMS Delivery Status Events (Bounded Fetching)

Use a bounded receipt reconciler to fetch SMS delivery status after notifying a gaming marketplace seller about a new order. Integration effort depends on the sender's documented receipt contract: a status snapshot and a delivery event history require different consumers. Short answer: persist the send identifier, fetch through one small adapter, and keep notification evidence separate from the order itself.

The implementation question often arrives as a request for a Node.js code example for fetching SMS delivery status events via API. Before writing that request, establish whether the API actually exposes events, which identifier it accepts, and what its delivery labels mean. The example here uses Python to make the consumer contract explicit. A Node.js implementation needs the same boundaries; translating the syntax does not establish the remote API contract.

## Decision: reconcile evidence without blocking the seller's order

For this design, the marketplace records an order notification intent alongside the order transaction. A separate sender records each SMS attempt and the identifier returned by its transport. The reconciler later fetches receipt evidence for that identifier. This is a proposed application design, not a claim that an SMS service implements a particular resource or response schema.

Keep orders authoritative.

A receipt is evidence about a notification attempt. It must not create an order, mark an item fulfilled, or establish that the seller read the message. Define those application invariants before connecting any transport. An example fixture could identify order `game-order-260`, seller `seller-17`, and attempt `attempt-2`; these are invented test identifiers, with no personal data or implied production measurements. Two attempts for one order must remain distinguishable, particularly when the first send request has an ambiguous outcome.

The failure boundary sits between the local order record and the external messaging system. A successful local commit does not prove external acceptance. Likewise, an unanswered send request leaves an uncertain result; retrying it blindly can produce an additional notification. Document whether the sending interface supports a deduplication key and what that key covers. Until its contract answers that question, record the uncertainty instead of claiming the attempt failed.

## How should code fetch SMS delivery status events via API?

A current-state resource answers a different question from an event collection.

A snapshot can tell the application what the sender reports now. It cannot reconstruct transitions that were never returned. An event feed can support a history only to the extent that its documented retention, pagination, ordering, and replay behavior permit it. Treat these as integration questions, not assumed features.

The comparison below concerns consumer designs. It does not imply that every messaging service offers all three interfaces.

| Approach | Initial integration work | Evidence boundary | Appropriate use |
| --- | --- | --- | --- |
| Poll a message snapshot | Map one lookup response and schedule bounded checks | Intermediate transitions may be unavailable | The application only needs the latest documented state |
| Fetch an event collection | Implement page traversal, durable progress, and deduplication | Completeness depends on the collection contract | A seller-support timeline needs individual receipts |
| Receive callbacks and reconcile | Authenticate inbound requests, persist receipts, and add recovery reads | Recovery still depends on what the read API retains | The application needs timely updates and can operate an inbound receiver |

For the smallest first integration, select snapshot polling only if the seller workflow can tolerate its update delay and does not require a transition history. If support needs to explain which receipt arrived before a second notification attempt, the event collection becomes a functional requirement. A convenient SDK cannot turn snapshots into missing history.

HTTP status is another boundary. RFC 9110 defines HTTP response semantics; an HTTP success response from a receipt lookup is not, by itself, a declaration that an SMS reached a handset. Read the application payload according to the sender's documented schema. Keep unknown delivery labels available for inspection rather than silently translating them into success.

## Critical path: fetch, commit, then advance

This example models an event collection. Its data classes are an internal adapter contract, deliberately separated from any external API. The adapter owns the documented URL, authentication, identifier encoding, response validation, and pagination translation. No universal SMS receipt endpoint is assumed. A snapshot adapter needs a different contract because it cannot promise individual events.

The scheduler supplies a page budget of 3 in this example. That is a local policy choice for a testable work unit, not a service limit. Each database operation commits its receipt inserts and progress marker together. A unique constraint on the attempt identifier plus event identifier makes replay harmless within this proposed design.

```python
from dataclasses import dataclass
from typing import Callable, Protocol


@dataclass(frozen=True)
class Event:
    event_id: str
    raw_status: str


@dataclass(frozen=True)
class Page:
    events: tuple[Event, ...]
    resume_token: str | None
    more: bool


class Store(Protocol):
    def load_progress(self, attempt_id: str) -> str | None: ...

    def commit_page(
        self,
        attempt_id: str,
        events: tuple[Event, ...],
        resume_token: str | None,
    ) -> None:
        # Atomically insert unseen events and persist progress.
        ...


def reconcile(
    attempt_id: str,
    remote_message_id: str,
    store: Store,
    fetch_page: Callable[[str, str | None], Page],
    page_budget: int = 3,
) -> str:
    if page_budget < 1:
        raise ValueError("page_budget must be positive")
    token = store.load_progress(attempt_id)

    for _ in range(page_budget):
        page = fetch_page(remote_message_id, token)
        if page.more and (
            page.resume_token is None or page.resume_token == token
        ):
            raise ValueError("adapter returned nonadvancing progress")
        store.commit_page(attempt_id, page.events, page.resume_token)
        token = page.resume_token
        if not page.more:
            return "caught_up"

    return "reschedule"
```

A caught-up result means this pass reached the end reported by the adapter. It does not mean delivery is final. Future receipts may still require another pass, subject to the sender's contract and the application's reconciliation deadline.

There is a deliberate constraint hidden in the adapter interface: the resume token must be safe to persist and reuse for later reads. If the remote API supplies only temporary page tokens, the adapter must instead implement a documented replay strategy, such as rereading an overlapping interval and deduplicating stable event identifiers. If neither replay nor durable progress is available, do not claim this consumer can maintain a complete event history. Consider a test fixture with two pages: the collector commits the first page, exits, and resumes after the original continuation token has expired. The stored token is accurate local state, yet it no longer provides usable remote progress. The acceptance test must establish how the adapter restarts that read, which receipts can be replayed, and how the database recognizes them. Otherwise, replacing a failed token with an empty value could quietly skip evidence or restart an expensive scan. Resolve that requirement before deployment.

Replay needs proof.

Fetch failures propagate to the worker, which retains existing progress and schedules another bounded attempt. The adapter must impose a request timeout. The scheduler must also cap attempts and elapsed reconciliation time; a page budget alone does not bound a slow request or endless rescheduling. On a documented retryable response, honor any applicable Retry-After semantics and add an application retry policy. Authentication errors and invalid schemas belong in an operator-visible failure path rather than an unlimited retry loop.

## Operational acceptance belongs in the integration estimate

Test the crash between fetching a page and committing it.

Then test a replay after a successful commit, an empty page that still advances, a repeated continuation token, an unfamiliar status, and a receipt belonging to an older send attempt. The expected behavior is specific: replay must not duplicate stored events, malformed progress must stop the pass, and an older attempt must not overwrite a newer attempt's evidence. These checks expose integration work that a successful request example leaves out.

Deploy the collector first with status-driven follow-up actions disabled. Compare its stored evidence with the documented response fixtures, then enable actions only for states whose meanings and precedence are established. Retain raw status alongside any normalized application state. An event identifier is a deduplication key only if the external contract guarantees the necessary uniqueness scope. A timestamp alone is not such a guarantee.

Measure the age of the oldest unresolved attempt, fetch failures by category, and work remaining after each bounded pass. Avoid putting phone numbers, message bodies, credentials, or order secrets into logs. For compliance review, document why receipt data is retained, who can access it, and when it is deleted; the appropriate policy depends on the marketplace's obligations. No retention duration is assumed here.

Email fallback needs its own delivery checks. RFC 7208 specifies SPF authorization for mail senders; it does not establish SMS receipt semantics or prove inbox placement. A seller's notification preferences and the application's escalation policy should govern channel changes. Receipt uncertainty alone should not trigger repeated messages across every available channel.

## Rejected: keep fetching inside the order request

The trade-off is explicit: asynchronous reconciliation adds a worker, durable progress, and operational ownership. Its limitation is that it cannot offer an immediate delivery verdict to the order handler. For a one-time operator lookup, choose a bounded synchronous read instead.

Synchronous fetching makes a demo shorter because one handler owns the send and subsequent checks. It also couples order-request latency to the receipt service and leaves a difficult recovery question when the process exits between external acceptance and local persistence. For this marketplace design, durable intent and a separate reconciler justify the additional worker and storage work.

The synchronous option still has a valid use case: an operator explicitly requests a one-time status refresh for an existing attempt. Bound the lookup, persist the evidence, and show uncertainty if no conclusive receipt is available. That operation does not need to wait for eventual delivery.

Choose the smallest receipt contract that satisfies the seller workflow. Include scheduler ownership, replay tests, observability, retention, and request volume in the integration estimate. The acceptance criterion is that a new order remains correct while its notification evidence can be fetched, explained, and recovered independently.

## References

- HTTP Semantics, RFC 9110: https://datatracker.ietf.org/doc/html/rfc9110
- Sender Policy Framework, RFC 7208: https://datatracker.ietf.org/doc/html/rfc7208
