# Capabilities (ZCAP profile)
Our own minimal implementation of ZCAP v0.4, with no library. It uses JCS + `eddsa-jcs-2022`. Implements D4, D10 and TB6, TB10, TB12, plus §5 of [technical-solution.md](../../technical-solution.md).

## 1. Model
```
root  urn:zcap:root:<crewId>:…:<X>             controller: X, resolved to X's N1 delegation key (§3)
 ├─ slip ①  → own N1       all of the crew's actions (e.g. calendar.freebusy.read, calendar.event.insert)   signed by the delegation key
 └─ slip ②  → peer's N1    delegable actions only (e.g. calendar.freebusy.read, calendar.meeting.propose)  within delegationPolicy[crew]
```
- **The root is never stored or signed.** It's derived from the target (§3).
- In the POC, every slip is a direct child of the root, so chains have depth 1. The algorithm in §7 handles deeper chains for future crew slips (③).
- **Who checks what:**
  - the approver's N1 checks slip ② invocations from peers
  - each crew's gateway checks slip ① invocations from its own N1
  - a requester's N1 checks a received slip ② once, on receipt

## 2. Slip format
Examples use the calendar crew. The format is the same for every crew; only the target's crew ID, the actions and the crew caveats differ.

**Slip ②** (Vishwas → Kash's N1):
```json
{
  "@context": ["https://w3id.org/zcap/v1"],
  "id": "urn:uuid:7f3e2a10-5c4b-4d8e-9a71-0b6c2d4e8f17",
  "parentCapability": "urn:zcap:root:calendar%3Accd%3Atestnet%3A4VishwasXq",
  "invocationTarget": "calendar:ccd:testnet:4VishwasXq",
  "controller": "agent:ccd:testnet:9876/11",
  "allowedAction": ["calendar.freebusy.read", "calendar.meeting.propose"],
  "caveats": [
    { "type": "SharingTier", "tier": "friend" },
    { "type": "PermittedDays", "days": ["sat", "sun"], "tz": "Europe/Copenhagen" },
    { "type": "SlotGranularity", "minutes": 30 },
    { "type": "RateLimit", "perDay": 20 },
    { "type": "MaxDelegationDepth", "value": 0 }
  ],
  "created": "2026-11-20T10:00:00Z",
  "expires": "2027-02-18T10:00:00Z",
  "proof": {
    "type": "DataIntegrityProof",
    "cryptosuite": "eddsa-jcs-2022",
    "verificationMethod": "agent:ccd:testnet:9876/2a#delegation",
    "proofPurpose": "capabilityDelegation",
    "capabilityChain": ["urn:zcap:root:calendar%3Accd%3Atestnet%3A4VishwasXq"],
    "created": "2026-11-20T10:00:00Z",
    "proofValue": "z4SlipSignature"
  }
}
```
**Slip ①** (Vishwas → own N1 #42) differs in these fields:
- `controller`: `agent:ccd:testnet:9876/2a`
- `allowedAction`: `["calendar.freebusy.read", "calendar.event.insert", "calendar.event.list", "calendar.event.create"]`
- `caveats`: `[{ "type": "RequiresOwnerConfirmation", "actions": ["calendar.event.insert"] }, { "type": "MaxDelegationDepth", "value": 0 }]`
- `expires`: one year after `created`

| Field | Rule |
|---|---|
| `allowedAction` | Values from §5 only |
| `caveats` | Array of typed objects. Every `type` must be in the registry (§6), otherwise `CAP_CAVEAT_UNKNOWN`. |
| `created`, `expires` | Required. `expires` must be after `created`. |
| `proof.capabilityChain` | IDs from the root to the parent. For a non-root parent, the last element is the full parent object. |

## 3. Root resolution
`resolveRootSigner(target, verificationMethod)`:
1. Parse the crew ID and X from `target` (`<crewId>:ccd:testnet:<X>`). The root ID is `urn:zcap:root:` + `encodeURIComponent(target)`.
2. `verificationMethod` must be `<agentId>#delegation`.
3. `verifyAgent(agentId, expectedOwner = X)` ([agent-card.md](agent-card.md) §5) must pass, with `card.role == "n1"`.
4. The crew must be listed in `card.crews`, otherwise `CAP_POLICY_EXCEEDED`. This is how removing a crew revokes all of its slips.
5. The signer is `card.keys.delegation`, and the limits are `card.delegationPolicy[crewId]`.

A node or gateway that serves a resource also checks `X == its own owner`. This stops anyone from presenting a slip for someone else's resource (`CAP_ROOT_MISMATCH`).

## 4. Invocation format
```json
{
  "type": "CapabilityInvocation",
  "id": "urn:uuid:2d8b6f41-93a7-4c0e-b5d2-6e1f0a9c7b33",
  "capability": { "id": "urn:uuid:7f3e2a10-5c4b-4d8e-9a71-0b6c2d4e8f17" },
  "capabilityAction": "calendar.freebusy.read",
  "invocationTarget": "calendar:ccd:testnet:4VishwasXq",
  "params": { "from": "2026-11-28T00:00:00Z", "to": "2026-11-30T00:00:00Z" },
  "proof": {
    "type": "DataIntegrityProof",
    "cryptosuite": "eddsa-jcs-2022",
    "verificationMethod": "agent:ccd:testnet:9876/11#invocation",
    "proofPurpose": "capabilityInvocation",
    "created": "2026-11-21T18:02:11Z",
    "nonce": "5e0c9b2a7d14f863",
    "proofValue": "z2InvocationSignature"
  }
}
```
`capability` carries the **full slip** in practice. It's abbreviated above. The proof covers everything, including `params`.

## 5. Actions and params
Actions are defined by **crew modules**, never by the core ([crew-modules.md](crew-modules.md) §3). At start, the loader registers each crew's actions in `trust-lib`'s `actionRegistry`, with a params schema and the `delegable` and `requiresConfirmation` flags. An action that isn't registered → `CAP_ACTION_NOT_ALLOWED`. Calendar actions: [crews/calendar.md](../crews/calendar.md) §2.

## 6. Caveat registry
**Core caveats**, available to every crew:
| Type | Value | Check at invocation | A child may only… | In slip ② |
|---|---|---|---|---|
| `RateLimit` | `{ perDay }` | At most N invocations per UTC day per slip | lower it | Optional (default 20) |
| `MaxDelegationDepth` | `{ value }` | — | decrease it (child = parent − 1) | Required; ≤ policy |
| `RequiresOwnerConfirmation` | `{ actions: [...] }` | Listed actions need a confirmation object ([consent.md](consent.md) §6) | add actions | Own slip ① only |

**Crew caveats** are registered by modules with the same shape: schema, `check(params, value, ctx)` and `narrower(child, parent)`. A crew caveat applies only to actions of its own crew. Calendar's (`SharingTier`, `PermittedDays`, `SlotGranularity`) are in [crews/calendar.md](../crews/calendar.md) §3.

**`delegationPolicy` check**, for any slip whose `controller` isn't the signing N1 itself. `P = card.delegationPolicy[<crewId of target>]`; no entry → reject.
- **Core fields:**
  - every action is `delegable` in its crew, and `allowedAction ⊆ P.maxActions`
  - `expires − created ≤ P.maxExpiryDays`
  - `MaxDelegationDepth` is present and `≤ P.maxDelegationDepth`
- **Crew fields:** any other keys in `P` (calendar: `allowedTiers`) are checked by the module's `checkPolicy(slip, P)`.

Otherwise the slip fails with `CAP_POLICY_EXCEEDED`. The issuer runs the check before signing, and every verifier runs it again.

## 7. Validation algorithms
**`verifyChain(slip, ctx)`**, from leaf to root, failing closed:
| # | Step | Reject with |
|---|---|---|
| 1 | Schema valid; every caveat type is in the registry | `SCHEMA_INVALID` / `CAP_CAVEAT_UNKNOWN` |
| 2 | Parent is the root: `parentCapability == rootId(target)`; signer from §3; `ctx.expectedOwner(target)` matches | `CAP_ROOT_MISMATCH` |
| 2′ | Parent is a slip: signer = the parent controller's delegation key; attenuation holds (actions ⊆ parent's, same target, `expires ≤` parent's, each caveat narrower per §6) | `CAP_ATTENUATION_VIOLATED` |
| 3 | `proof` verifies (`eddsa-jcs-2022`) with the signer key | `CAP_SIG_INVALID` |
| 4 | `delegationPolicy` check when the controller isn't the signer | `CAP_POLICY_EXCEEDED` |
| 5 | `now < expires` for every slip in the chain | `CAP_EXPIRED` |
| 6 | No slip in the chain is revoked (`ctx.isRevoked`) | `CAP_REVOKED` |

**`verifyInvocation(inv, ctx)`**:
| # | Step | Reject with |
|---|---|---|
| 1 | Schema valid; `capability` present | `SCHEMA_INVALID` / `CAP_MISSING` |
| 2 | `verifyChain(inv.capability, ctx)` | as above |
| 3 | Invoker = agent of `proof.verificationMethod`, and it equals `capability.controller` | `CAP_NOT_HOLDER` |
| 4 | `verifyAgent(invoker)`; `proof` verifies with the invoker's **invocation** key | agent codes / `CAP_SIG_INVALID` |
| 5 | `proof.nonce` unseen; `proof.created` within ±5 min | `NONCE_REPLAY` / `TS_SKEW` |
| 6 | `invocationTarget` = slip's target; `capabilityAction ∈ allowedAction` | `CAP_ROOT_MISMATCH` / `CAP_ACTION_NOT_ALLOWED` |
| 7 | `params` match the action schema (§5) and **every caveat in the chain** | `SCHEMA_INVALID` / `CAP_CAVEAT_VIOLATED` |
| 8 | Rate counter under `RateLimit` | `CAP_RATE_LIMITED` |
| 9 | `RequiresOwnerConfirmation`: the confirmation is present and valid ([consent.md](consent.md)) | `CONSENT_REQUIRED` / `CONSENT_*` |

It returns `{ ok: true, slip, action, caveats }`, the effective caveats of the whole chain, which the crew module uses to shape its response (calendar: tier and granularity). On success the rate counter and nonce cache are updated.

## 8. Issuing and revoking
**Issuing** happens in `n1-agent`, only from an owner click in the UI:
1. Build the slip from the chosen scope.
2. Run the §6 policy check.
3. Sign with the delegation key.
4. Store it as `issued`.
5. Send `capability.grant`.

The LLM never calls this path.

**On receipt** (requester side): `verifyChain`, with `expectedOwner` taken from the target. If it passes, store the slip as `received`. If not, log the reason and drop it.

**Revoking:**
- **One slip:** add its ID to the issuer's revocation list (effective immediately at the issuer's own checks), then send `capability.revoke` so the holder stops using it.
- **All slips from the delegation key:** rotate the key ([agent-card.md](agent-card.md) §6).
- **The whole agent:** CIS-8004 `revoke`. Every check then fails with `AGENT_REVOKED`.

## 9. Tests (unit vectors in `trust-lib`)
- valid ② invocation → `OK_INVOCATION`
- no slip → `CAP_MISSING` (NT4)
- weekday window → `CAP_CAVEAT_VIOLATED` (NT5)
- `calendar.event.insert` via ② → `CAP_ACTION_NOT_ALLOWED` (NT5)
- slip ② with `calendar.event.insert` in `allowedAction` → `CAP_POLICY_EXCEEDED` (NT6)
- unknown caveat type → `CAP_CAVEAT_UNKNOWN` (NT6)
- child wider than its parent → `CAP_ATTENUATION_VIOLATED`
- slip signed by another agent's delegation key → `CAP_ROOT_MISMATCH`
- invoked by a non-holder → `CAP_NOT_HOLDER`
- revoked slip → `CAP_REVOKED` (NT10)
- after delegation-key rotation → `CAP_SIG_INVALID`
- replayed invocation nonce → `NONCE_REPLAY`
