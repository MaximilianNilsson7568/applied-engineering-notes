# Who Owns the Email Template? Custom-Domain DKIM Sending for Node.js Order Alerts

Sellers on a media marketplace get paid after they deliver a licensed asset, not after they read your mail, so a new-order alert carries a deadline that the welcome message sharing its pipeline does not. That deadline is the operational constraint that decides the design. In short: own the template in the application repo, render it inside the Node.js worker at send time, authenticate the custom domain and DKIM before any production traffic reaches the sending API, and give every send a durable identity so a retry can never turn into a second email.

Delivery is not a purchase. Inbox placement follows from the domain you send from, the consent you hold, the content you ship, and the shape of your sending pattern — no configuration screen changes those inputs.

I'm not sure any particular transactional setup ranks ahead of another for a specific recipient population without a controlled test on your own list. What follows is the part that is decidable before you have any data.

## Invariants the order-alert path has to hold

Four rules hold no matter who moves the bytes.

Envelope identity comes first. DKIM signs a message with a selector key published in DNS (RFC 6376), but DMARC only credits that signature when the signing `d=` domain aligns with the visible RFC5322.From domain, and that alignment can be strict or relaxed depending on your policy record (RFC 7489). A perfectly valid signature from an unaligned domain passes DKIM and still fails DMARC, which is where a surprising number of "we already configured DKIM" tickets end up: **the signing domain has to align with the From domain the seller actually sees**.

Template determinism comes second. The exact bytes that reach a seller must be reproducible from a version identifier that ships with the deployment, or a copy edit made at 23:40 becomes an incident nobody can reconstruct.

Third, one order event maps to exactly one send intent. Generate that identity before the first API call, persist it, and reuse it across every retry, otherwise an ambiguous timeout turns into two order alerts and a support ticket about a duplicate payout.

Fourth, delivery state comes from delivery events. Opens are not evidence — Apple's Mail Privacy Protection fetches remote content through a proxy whether or not a human ever looked at the message, so an open cannot mark an order as acknowledged, cannot gate a follow-up, and cannot trigger a second attempt.

## How should a Node.js service send a transactional email from a custom domain with DKIM?

Four stages, in this order, with a gate between each: authenticate the domain, release the template, send from a durable job, reconcile the outcome.

Authenticate on a dedicated subdomain — `mail.marketplace.example` rather than the apex — so a promotional incident cannot drag the order path down with it. Publish the selector record with a 2048-bit key, an SPF record covering the sending source, and a DMARC record that starts at `p=none`. Read aggregate reports for two or three weeks, confirm that the streams you care about are aligned, then move to quarantine and later to reject. Rotating selectors is the reason the record is `s1._domainkey` and not `default`; you want a second selector live before you retire the first.

Verify the records from the deployment gate, not from memory:

```bash
dig +short TXT s1._domainkey.mail.marketplace.example
dig +short TXT mail.marketplace.example
dig +short TXT _dmarc.marketplace.example
```

DNS propagation is not a fixed number, because resolver caches expire on their own schedule, so a release job should poll for the expected record and fail after a bounded wait instead of sleeping for 60 seconds and hoping. Then send from a queue. A synchronous HTTP call inside the checkout handler makes the buyer wait on your provider's p99 latency, and it couples order creation to an unrelated availability domain; the worker owns the send, honours `Retry-After` on a 429, backs off with jitter otherwise, gives up after about five attempts, and parks the job where an operator can see it. Reconciliation runs slower and that's fine: push events or a polled cursor both work, as long as every state transition is idempotent and late events cannot move a message backwards from `bounced` to `delivered`.

## Three homes for a template, and the bill each one sends you

Template ownership is the axis that actually differentiates these systems, and it's the decision teams make by accident.

| Where the template lives | What it buys | The catch |
|---|---|---|
| Provider-hosted, edited in a dashboard | Non-engineers change copy without a deploy; previews are built in | Change history sits outside your VCS, review is informal, rollback is manual, and offline rendering tests are hard to write |
| Application repo, rendered at send time | Diffs get reviewed, rendering is deterministic and unit-testable, portable across senders | Every copy change needs a deploy, and the content team files pull requests |
| Repo-authored, published as an immutable versioned artifact | Review plus rollback plus copy edits without a full deploy | You now own a build step, a version registry, and the rule that pins a version to a release |

For a seller order alert, the repo owns the template. The message contains money, an order identifier and a deadline, and every one of those is a correctness concern rather than a copy concern. For a lifecycle series or a monthly editorial digest, dashboard ownership is the honest answer — a content team shipping daily changes should not be blocked on a deploy pipeline, and a marketing message that renders slightly wrong is not an incident.

The catch is that the third option is only worth its complexity above a certain change rate. If copy changes twice a quarter, stick with the repo and skip the registry.

## The preflight that keeps a broken template away from sellers

A template gate is worth more than another provider comparison, because it catches the failures that only appear in production data. Run it in CI against adversarial fixtures, not against a friendly `Sam` example: an ampersand in a shop name, an empty optional field, a 70-character asset title, a zero-decimal currency, a right-to-left display name, and a missing thumbnail.

```python
import hashlib
import re
from urllib.parse import urlparse

from jinja2 import Environment, FileSystemLoader, StrictUndefined

ALLOWED_LINK_HOSTS = {"marketplace.example", "mail.marketplace.example"}
APPROVED_DIGEST = "3f8b1c9d5e2a4706"  # bump deliberately, in the same commit as the copy change

FIXTURES = [
    {"seller": "Ría & Co.", "shop": "", "asset": "A" * 70,
     "order_id": "ord_9F2K", "amount_minor": 4900, "currency": "JPY", "thumbnail": None},
    # right-to-left display name, escaped so the fixture file stays ASCII
    {"seller": "\u05e6\u05dc\u05de\u05d9 \u05e8\u05d5\u05ea", "shop": "Cairo Stills", "asset": "Rooftop timelapse",
     "order_id": "ord_7QB1", "amount_minor": 1250, "currency": "EUR", "thumbnail": None},
]

env = Environment(loader=FileSystemLoader("templates"), undefined=StrictUndefined, autoescape=True)


def check(name):
    html_tpl = env.get_template(f"{name}.html.j2")
    text_tpl = env.get_template(f"{name}.txt.j2")
    digest = hashlib.sha256()

    for fixture in FIXTURES:
        html = html_tpl.render(**fixture)
        text = text_tpl.render(**fixture)

        assert "{{" not in html and "{{" not in text, f"unrendered placeholder in {name}"
        assert fixture["order_id"] in text, "plain-text part must carry the order id"
        assert len(text.strip()) > 120, "plain-text alternative is too thin to be useful"
        assert "&amp;" in html or "&" not in fixture["seller"], "seller name was not escaped"

        for url in re.findall(r'href="(https?://[^"]+)"', html):
            host = urlparse(url).hostname or ""
            assert host in ALLOWED_LINK_HOSTS, f"link host {host} is not on the allowlist"

        digest.update(html.encode("utf-8"))
        digest.update(text.encode("utf-8"))

    rendered = digest.hexdigest()[:16]
    assert rendered == APPROVED_DIGEST, f"{name} rendered {rendered}, approved {APPROVED_DIGEST}"


if __name__ == "__main__":
    check("seller_order_alert")
```

Three details matter more than the assertions themselves. `StrictUndefined` turns a typo in a variable name into a build failure instead of a blank line in a seller's inbox. The digest is a deliberate gate, not a lock: a reviewer bumps it in the same commit that changes the copy, which makes an unreviewed template edit impossible to ship. And the plain-text part is checked with the same rigour as the HTML, because a text alternative that says "view this email in your browser" is the kind of thing that quietly moves a message toward a spam folder.

Keep recipient addresses and rendered bodies out of ordinary logs, store the provider message identifier next to the send intent, and decide retention before launch rather than during a deletion request.

## What I rejected, and when the rejected option is right

Three designs got rejected here: sending inline in the request handler, letting the dashboard own the order-alert template, and treating opens as a delivery signal. They widen the failure boundary around a revenue path, and none of them is recoverable at the moment it matters.

Each one is correct somewhere else. Inline sending is fine for an internal tool with no worker tier, where an extra 400 ms and an occasional lost message cost nothing. Dashboard-owned templates are right when a content team ships copy daily and no deploy pipeline should stand between them and a promotional send. Open tracking is not suitable as a delivery signal, but it still has a place in aggregate engagement reporting, as long as nobody builds a state machine on top of it.

The acceptance test at the end is short: an aligned custom domain with a live DKIM selector, a template version pinned to the deployment, one durable send identity per order, bounded retries, suppression honoured, and delivery state that comes from events. **Miss any one of those and the system is guessing about whether a seller was notified.** Everything after that — which API you call, which language the worker speaks — is an implementation detail you can change on a Tuesday.

## Sources

- https://datatracker.ietf.org/doc/html/rfc6376
- https://datatracker.ietf.org/doc/html/rfc7208
- https://datatracker.ietf.org/doc/html/rfc7489
- https://datatracker.ietf.org/doc/html/rfc8058
- https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios
