---
name: hiveagentiq-com-onboard-agent
description: Register an agent with the Hive network through HiveGate — free first DID, then present it on every metered call and know exactly what each next step costs.
api: HiveGate Admission, Identity & Pricing Tier API (https://hivegate.hiveagentiq.com)
operations:
- POST /v1/gate/onboard
- POST /v1/gate/register-guest
- POST /v1/gate/admit
- GET /v1/gate/tier/verify
- POST /v1/gate/tier/upgrade
mcp_tools:
- hivegate_register_guest
- hivegate_bridge_trust
- hivegate_translate_intent
method: generated
source: openapi/hiveagentiq-com-hivegate-openapi.json, https://hivegate.hiveagentiq.com/llms.txt, https://hivegate.hiveagentiq.com/.well-known/hivegate.json, live 402 envelope (2026-09-19)
---

# Onboard an agent through HiveGate

HiveGate is the admission gateway for every Hive service on hiveagentiq.com. Nothing on HiveTrust,
HiveBank or HiveLaw beyond the free reads works without the DID it issues.

## Steps

1. **Read the free surfaces first.** `GET /health`, `GET /v1/gate/queue/stats` (admission queue
   depth and wait) and `GET /v1/gate/sample` (the 7-framework adapter manifest) need nothing and cost
   nothing; `GET /openapi.json` and `GET /llms.txt` are the contract and the guide.
2. **Mint the first DID — free.** `POST https://hivegate.hiveagentiq.com/v1/gate/onboard` with
   `{"agent_name":"your-agent","email":"you@domain.com"}` (the body the 402 envelope's quick_start
   gives). The llms.txt states "first DID is free, no payment required"; the OpenAPI nevertheless
   declares a $1.00 `x-mpp-charge` on this operation — expect free, be ready for a 402. The response
   returns a `did:hive:*` identifier and an API key. Optional: `referral_did` earns the referrer a credit.
3. **Present the DID on everything after that.** Send it as `X-Hive-DID: did:hive:…` (the "sovereign
   handshake"), or as `X-API-Key`, or `Authorization: Bearer did:hive:…`. Without it, metered routes
   answer `402 {"code":"HIVE_402","detail":{"x402":{"amount_usdc":9.99,"headers_required":["X-Hive-DID"]}}}`
   on HiveGate and `401 {"error":"agent_not_registered"}` on HiveTrust/HiveBank.
4. **Guest path for agents from another framework.** If the agent already lives in LangChain, CrewAI,
   AutoGen, OpenAI, Anthropic or an A2A network, `POST /v1/gate/register-guest` ($4.99, x402) or the
   MCP tool `hivegate_register_guest` `{"external_id","source_platform","agent_name"}` returns a
   `guest_did` (did:hive:guest:*) and an `hgate_*` access token; `hivegate_bridge_trust` then maps the
   external reputation onto a Hive score (0.5% bridge fee per the card skill).
5. **Admission and tiers.** `POST /v1/gate/admit` ($0.10) verifies admission status and tier;
   `GET /v1/gate/tier/verify` ($0.10) checks a tier credential; `POST /v1/gate/tier/upgrade` ($1.00)
   moves up. Priority onboarding that skips the queue is $100 USDC (hivegate.json).

## Rules the contract states

- Payment is x402: the 402 body carries `amount_usdc` (llms.txt: "amount_min_usd — the floor price.
  Submit any value >= that floor"); retry the same request with the payment attached. Settlement is
  USDC/USDT on Base to `0x15184Bf50B3d3F52b60434f8942b7D52F2eB436E`; the HiveGate hive-payments.json
  names `X-Payment` as the header. On-chain settlement is final.
- No idempotency key exists. A retried onboarding or guest registration is a second registration
  (and, for guests, a second $4.99). Keep the returned DID; do not re-run step 2 on a timeout without
  first checking whether the DID was issued.
- Unknown paths on HiveGate return the 402 envelope, not a 404; a real 404 (`"error":"not_found"`,
  with `recovery_actions`) only appears for `/.well-known/*` misses. `/.well-known/hivegate.json` is the
  full route catalogue.
