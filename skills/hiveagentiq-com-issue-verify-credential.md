---
name: hiveagentiq-com-issue-verify-credential
description: Issue a scoped HiveCredential or a W3C Verifiable Credential to an agent, verify one presented to you, and narrow or revoke it — with the fee and reversibility facts an agent must know first.
api: HiveTrust KYA Identity & Trust API (https://hivetrust.hiveagentiq.com)
operations:
- POST /v1/trust/did/generate
- POST /v1/trust/vc/issue
- POST /v1/credential/issue
- POST /v1/credential/verify
- POST /v1/credential/scope
mcp_tools:
- hivetrust_issue_credential
- hivetrust_verify_credential
- hivetrust_revoke_credential
method: generated
source: openapi/hiveagentiq-com-hivetrust-openapi.json, mcp/hiveagentiq-com-hivetrust-mcp-tools.json, a2a/hiveagentiq-com-agent-card.json (fee_schedule, capability_vcs), well-known/hiveagentiq-com-hivetrust-did-configuration.json
---

# Issue, verify and revoke credentials on HiveTrust

HiveTrust issues two credential shapes: **HiveCredential**, a scoped capability credential the spec
calls "issue scoped credential", and **W3C Verifiable Credentials** (VCDM 2.0, `did:key`,
`Ed25519Signature2020` per the card). Both need a registered Hive DID (see
hiveagentiq-com-onboard-agent) and both are paid per event.

## Steps

1. **Have a DID for the subject.** If the subject agent has none, `POST /v1/trust/did/generate`
   ($1.00 USDC, "One-time per new did:web:* registered") or the MCP tool `hivetrust_register_agent`
   (`name`, `owner_id` required; optional `public_key` in ed25519-base58/hex/jwk, `eu_ai_act_class`).
2. **Issue.**
   - Scoped HiveCredential: `POST /v1/credential/issue` — $0.10 (`x-mpp-charge` 100000).
   - W3C VC: `POST /v1/trust/vc/issue` — $0.50; or MCP `hivetrust_issue_credential` with
     `{"agent_id","credential_type","issuer_id","claims", "expires_at"?}`. The card's fee schedule
     prices the per-event route `POST /v1/agents/:id/credentials` at $0.10.
   The served OpenAPI declares no request schema for any of these; the MCP inputSchema is the only
   published input contract, so shape REST bodies from it.
3. **Verify a credential presented to you.** `POST /v1/credential/verify` — $0.01 (the cheapest
   metered call on the host), or MCP `hivetrust_verify_credential` `{"credential_id"}`: "Checks
   signature validity, revocation status, and expiration." Read tools are idempotent; verify as often
   as you like.
4. **Narrow, freeze or revoke.** `POST /v1/credential/scope` — $0.05 — "mutate scope
   (narrow/freeze/unfreeze/revoke)". Over MCP, `hivetrust_revoke_credential` `{"credential_id","reason",
   "agent_id"?, "evidence"?}` is annotated `destructiveHint: true`: "Revoked credentials immediately
   fail verification checks." The card prices a revocation event at $0.10 (`DELETE
   /v1/agents/:id/credentials/:cid`).
5. **Retrieve what an agent holds.** `GET https://hivetrust.hiveagentiq.com/v1/agents/:did/credentials`
   (card `capability_vcs.retrieve`; requires the DID — unauthenticated it is 401 `agent_not_registered`).

## Rules the contract states

- Fees are per event and non-refundable; revoking does not refund issuance. No reversal window is
  published for any of these operations — revocation is available, unfreeze exists for a frozen scope,
  but nothing states a deadline or a grace period (conventions/hiveagentiq-com-conventions.yml,
  reversibility).
- No idempotency key: a retried issue is a second credential and a second fee. Verify (step 3) before
  re-issuing after an ambiguous outcome.
- The domain-linkage credential at `/.well-known/did-configuration.json` binds `did:key:z6Mkrkfp…`
  to the host, but its `proofValue` is the literal placeholder `domain-linkage-proof-placeholder`; do
  not treat the host's own linkage as cryptographically verified.
- Every response is signed (`X-Hive-Prov-Sig`, key at `/v1/prov/pubkey`); a 200 whose body contains
  `"doctrine"` is the host's catch-all miss, not a result.
