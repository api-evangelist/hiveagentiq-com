---
name: hiveagentiq-com-check-agent-trust
description: Look up another agent's Hive trust standing before transacting with it — free unauthenticated lookup first, paid attested score only when a decision needs it.
api: HiveTrust KYA Identity & Trust API (https://hivetrust.hiveagentiq.com)
operations:
- GET /v1/trust/lookup/{did}
- GET /v1/trust/score/{did}
- GET /v1/trust/reputation/proof
mcp_tools:
- hivetrust_get_trust_score
- hivetrust_verify_agent_risk
- hivetrust_verify_bond
method: generated
source: openapi/hiveagentiq-com-hivetrust-openapi.json, mcp/hiveagentiq-com-hivetrust-mcp-tools.json, live GET https://hivetrust.hiveagentiq.com/v1/trust/lookup/did:key:z6Mktest (2026-09-19)
---

# Check an agent's trust standing on HiveTrust

HiveTrust scores registered agents on a 0–1000 composite (the served OpenAPI says 0–100 for the
paid read; the MCP tool and the cards say 0–1000 — expect the latter) and exposes the result three
ways. Use the cheapest one that answers the question.

## Steps

1. **Free lookup, no credential.** `GET https://hivetrust.hiveagentiq.com/v1/trust/lookup/{did}`.
   This route is undeclared in the OpenAPI but the agent card's `extensions.asqav` block names it
   as the public derivation right ("No auth required — GET /v1/trust/lookup/:did") and it answered
   200 live. For an unknown DID the body is
   `{"did":…,"found":false,"trust_score":null,"trust_tier":null,"status":"unknown",
   "recommendation":"unverified — no Hive identity. Proceed with caution or require onboarding.", …}`.
   If `found` is false, stop: the counterparty has no Hive identity and nothing below will change that.
2. **Paid attested score.** `GET /v1/trust/score/{did}` — $0.10 USDC per call (`x-mpp-charge`
   100000). Without payment the response is 402 "Payment required — x402 or MPP"; settle via x402
   (`X-Payment` header, USDC on Base) or the MPP rail and repeat the call. Over MCP the same read is
   `hivetrust_get_trust_score` with `{"agent_id": "<uuid or did>"}` (readOnlyHint true,
   idempotentHint true); `hivetrust_verify_agent_risk` returns an ALLOW/REVIEW/BLOCK verdict with a
   max transaction limit, which is the shape a payment gate wants.
3. **Portable proof.** When you need to hand the score to a third party, `GET /v1/trust/reputation/proof`
   ($0.10) returns a signed reputation proof. Verify the response signature with the issuer key at
   `GET /v1/prov/pubkey` (Ed25519, `X-Hive-Prov-*` headers on every response).
4. **Collateral.** `hivetrust_verify_bond` `{"did": …}` reports whether the agent has staked a USDC
   bond and at which tier (bronze/silver/gold/platinum; tier minimums 100/500/2,000/10,000 USDC per
   hivetrust.json). The REST twin `GET /v1/bond/verify/:did` is documented as free but was not observed.

## Rules the contract states

- Every response carries `X-RateLimit-Limit: 100` / `X-RateLimit-Remaining` / `X-RateLimit-Reset`
  (epoch milliseconds, hourly window). Back off when Remaining approaches 0; `Retry-After` is exposed.
- Read tools are idempotent (annotated); repeat them freely. Paid reads are billed per call.
- On this host an HTTP 200 whose body has `"requested"` and `"doctrine"` keys is a **miss**, not a
  result — the host answers 200 for unknown paths. Check for `trust_score` / `data` before trusting it.
- Registration and every write need a Hive DID (see hiveagentiq-com-onboard-agent).
