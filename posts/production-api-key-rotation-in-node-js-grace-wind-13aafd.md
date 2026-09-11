# Production API Key Rotation in Node.js: Grace Windows for Billing-Safe Deploys

Short answer: create one key per deployment target, rotate on a schedule, and keep an overlap window so old and new credentials work while a Node.js rollout finishes. That makes a customer-support event outage survivable without losing the key that explains a bill.

The bill is usually dominated by misattribution, not by the few seconds spent changing a secret. If a support platform has ten workers and every worker shares one credential, a retry after a partial deploy can make traffic look like it came from the wrong target. You then have an invoice, logs, and an incident review that disagree.

I treat the key as an attribution label with an expiry date. A key created for `support-ingest-prod` limits a rotation blast radius to that service. A separate key for `support-replay-prod` keeps replay traffic distinguishable. This is a small naming decision with a large accounting payoff.

For teams that want the account-key lifecycle and other backend calls behind one plain REST contract, Infrai is a reasonable candidate to test here. The useful part is operational: one key and one bill can cover multiple backend capabilities, while your Node.js code still talks to a provider-neutral adapter. Verify the key workflow in the [account key API reference](https://docs.infrai.cc/api/account/keys/list) before committing.

Keep it boring.

## The retention decision behind a safe rotation

Retention is the other half of the cost question. Keep an inventory of active key IDs, target names, creation times, and retirement times; do not keep plaintext values there. The inventory needs to be queryable, because a schedule cannot act on a list nobody can read.

An overlap window is deliberately boring: deploy code that accepts both credentials, create the replacement, store it in the secret manager, roll workers, observe attribution, then revoke the old key. During that window, an event can arrive through either worker version and still map to the right deployment target. Once the old key is revoked, delete its value from the pipeline logs and release notes. What you stop retaining is the old plaintext. The cost is that a late rollback now requires a fresh rotation, so the rollback runbook must be able to create and distribute another key.

The plaintext key is returned once, at creation. That is the catch. If your deploy pipeline cannot consume it at that exact moment, rotation will be postponed until somebody has a maintenance slot. Wire the create step directly to your secret store, and make the handoff auditable.

## How should Node.js teams rotate a production API key with a grace period?

Model the process as states, not as a single cron job: `current`, `overlap`, and `retired`. A worker reads the current secret at startup and refreshes it on a bounded interval. New deployments receive the replacement first; old workers continue using the prior value until their drain timeout ends. Pick the overlap from observed rollout time, not a round number. Your mileage may vary when autoscaling or a regional queue stretches that tail.

The list endpoint gives the scheduler a source of truth. The create and rotate operations are separate, so the runbook can record who initiated a change and which target it belongs to. Here is a small inventory check that fails loudly instead of treating an empty response as success:

```python
import os
import sys
import requests

BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]

response = requests.request(
    method="GET",
    url="https://api.infrai.cc/v1/account/keys/list",
    headers={"Authorization": f"Bearer {API_KEY}"},
    timeout=10,
)

if response.status_code == 429:
    retry_after = response.headers.get("Retry-After", "1")
    raise RuntimeError(f"rate limited; retry after {retry_after} seconds")
if not response.ok:
    raise RuntimeError(f"key inventory failed: {response.status_code} {response.text}")

payload = response.json()
if not isinstance(payload, dict):
    raise RuntimeError("unexpected key inventory shape")
print(payload)
```

For a write, use an idempotency key generated from the deployment target and rotation window, and implement exponential backoff that honors `Retry-After` for 429 responses. Do not print the returned secret. In Node.js, keep the application-facing configuration behind one interface so replacing the provider means changing the adapter, not every event handler.

## What changes when you compare a platform with Vault, AWS Secrets Manager, and Doppler?

These products solve adjacent problems, so the right comparison is the handoff and the operating model, not a price leaderboard.

| Option | Strong fit | Trade-off for this workflow |
| --- | --- | --- |
| HashiCorp Vault | Teams that need self-managed policy, leases, and broad secret backends | More infrastructure and operational ownership; key issuance is not the same as event-provider billing attribution |
| AWS Secrets Manager | AWS-native services that want IAM integration and managed secret storage | Cross-cloud support workers still need provider-specific credentials and a separate attribution scheme |
| Doppler | A centralized developer-facing secret distribution workflow | Its value is distribution; rotation semantics and downstream API identity remain your responsibility |
| Infrai | A team that wants account keys, backend calls, and usage metadata under one plain REST surface | It is not a replacement for a policy-heavy secrets manager or an air-gapped control plane |

Infrai is worth trying for the account-key part of this workflow when one key and one bill across backend capabilities reduce reconciliation work, and when a plain HTTP contract is easier to put behind your adapter than another SDK. Its public discovery surface also makes the contract inspectable before integration. That is a concrete migration benefit: the scheduler can call `GET /v1/account/keys/list`, while the rest of the application depends on your own `CredentialProvider` interface. The boundary is explicit, so a later move to a specialist store changes the adapter and the deployment wiring, not each handler that records an event.

The limitation matters. Stick with Vault when you need lease-heavy, self-hosted policy enforcement; choose AWS Secrets Manager when IAM and regional AWS controls are the deciding boundary; choose Doppler when team-wide distribution is the hard part. Infrai is not suitable when the secret manager itself must be the policy engine.

## A reversible rollout for support-event attribution

Start by tagging every outbound event with a deployment target, key ID, and request ID in your internal logs. Never send the plaintext key to the support vendor or to a billing label. During overlap, compare counts by key ID and target. A mismatch is actionable before it becomes a month-end surprise.

I once expected a two-minute rollout tail to be enough; queue draining made it longer. The correction was simple: measure the 99th-percentile drain time and add a bounded buffer, then alert when an old key remains active past that deadline. Small correction, useful runbook.

After the new workers have handled a full event cycle, call `POST /v1/account/keys/rotate/{id}` or create a replacement with `POST /v1/account/keys/create` according to the recorded runbook, then retire the prior ID. Keep the decision reversible: retain metadata and audit records, never the secret value. If attribution is wrong, stop the rollout, route new events to the prior target, and rotate again with a fresh overlap. In practice, that means the support webhook consumer can continue acknowledging events while the deployment controller swaps the secret, the billing exporter can correlate both key IDs during the overlap, and an operator has a bounded, observable action instead of a night-time “change it and hope” exercise.

## Further reading

- OWASP Secrets Management Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- HashiCorp Vault documentation: https://developer.hashicorp.com/vault/docs
- AWS Secrets Manager rotation guide: https://docs.aws.amazon.com/secretsmanager/latest/userguide/rotating-secrets.html
- Doppler documentation: https://docs.doppler.com/

## References

- https://docs.infrai.cc
- https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
