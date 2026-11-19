# Agent Card and CIS-8004 registration
Implements D2, D4 and TB4, TB6, TB12 of [technical-solution.md](../../technical-solution.md).

## 1. Purpose
The Agent Card publishes an agent's public keys and its `delegationPolicy`. Its hash is written to CIS-8004 by owner account X, which is how X authorises the keys. Anyone can check a card against the chain.

## 2. Card format
```json
{
  "type": "NumberOneAgentCardV1",
  "role": "n1",
  "name": "Vishwas's Number One",
  "owner": "ccd:testnet:acc:4VishwasXq",
  "keys": {
    "invocation": { "type": "Ed25519", "publicKeyMultibase": "z6MkInvocationKeyA1" },
    "delegation": { "type": "Ed25519", "publicKeyMultibase": "z6MkDelegationKeyD7" },
    "encryption": { "type": "X25519", "publicKeyMultibase": "z6LSEncryptionKeyE3" }
  },
  "crews": ["calendar/1"],
  "delegationPolicy": {
    "calendar": {
      "maxActions": ["calendar.freebusy.read", "calendar.meeting.propose"],
      "allowedTiers": ["friend"],
      "maxExpiryDays": 90,
      "maxDelegationDepth": 0
    }
  },
  "endpoints": { "relay": "wss://relay.number-one.dev/ws" },
  "created": "2026-11-20T09:00:00Z"
}
```
| Field | Rule |
|---|---|
| `type` | Exactly `NumberOneAgentCardV1` |
| `role` | `n1` in the POC (crew roles are reserved) |
| `owner` | Must equal the on-chain `owner_account` (else `CARD_INVALID`) |
| `keys.*` | All three required. The three public keys must be distinct. |
| `crews` | Installed crew modules as `<id>/<version>` ([crew-modules.md](crew-modules.md)). Peers show them in the directory and only use crews both sides have. |
| `delegationPolicy` | One entry per installed crew, with its limits on any slip whose `controller` is **another** agent (slip ②). It doesn't apply to the own-N1 slip ①. A crew in `crews` without a policy can't be granted to peers. See [capabilities.md](capabilities.md) §6. |
| `name` | Display only; the verified name comes from the attestation, never from the card |

The card contains **no agent ID**: the token ID is only known after registration, and the binding comes from the chain (agent record → `metadata_hash`).

## 3. Hash and hosting
- `cardHash = sha256(JCS(card))` → 32 bytes → `metadata_hash` on chain.
- The node uploads the card to the relay: `PUT /cards` ([relay-service.md](../relay/relay-service.md) §3.2). The relay stores it by hash.
- `agent_uri = <RELAY_PUBLIC_URL>/cards/<cardHash hex>` (content-addressed).
- A peer fetches by URI, recomputes the hash, and rejects on mismatch (`CARD_HASH_MISMATCH`). The relay can't alter cards.

## 4. CIS-8004 operations
| Operation | Entrypoint | Parameters | Signed by | When |
|---|---|---|---|---|
| Register N1 | `register` | `agent_uri`, `metadata_hash = cardHash`, no external reference, `initial_metadata = [{ key: "app", value: utf8("number-one/1") }]` | X (Browser Wallet) | Setup |
| Rotate keys / policy | `setAgentURI` | `token_id`, new `agent_uri`, new `metadata_hash` | X | Key rotation |
| Revoke | `revoke` | `token_id`, `reason` | X | Owner revokes the N1 |
| Read | `agentOf`, `getMetadata` | `token_id` | — (free) | Every verification |

- The new token ID is read from the `Registered` event of the finalised `register` transaction.
- `app = number-one/1` marks Number One agents for tooling. It is **not** a trust signal: membership comes from the attestation.

## 5. `verifyAgent`
Used by every node before trusting a peer, and by the relay at attestation time.
```
verifyAgent(agentId, expectedOwner?):
  v = agentOf(tokenId)                       → AGENT_NOT_FOUND
  require v.status == Active                  → AGENT_REVOKED
  if expectedOwner: require v.owner_account == expectedOwner   → OWNER_MISMATCH
  card = cache.get(v.metadata_hash) ?? fetch(v.agent_uri)      → CARD_UNAVAILABLE
  require sha256(JCS(card)) == v.metadata_hash                 → CARD_HASH_MISMATCH
  require schema(card) && card.owner == v.owner_account        → CARD_INVALID
  return { agentId, owner: v.owner_account, card }
```
- Card cache key: `metadata_hash`. A hash change on chain (rotation) invalidates it automatically.
- `agentOf` results are cached for **60 s**. They're re-read at session start and before every commitment.

## 6. Key rotation
1. `n1-agent` generates new keys (any or all of the three) and builds a new card. A new `delegationPolicy` or crew list can be set at the same time. Adding or removing a crew uses this same flow, even without new keys ([crew-modules.md](crew-modules.md) §2).
2. `n1-agent` uploads the card to the relay.
3. The owner signs `setAgentURI` in the Browser Wallet.
4. After finalisation, `n1-agent` switches to the new keys.

**Effect:** every slip signed by the old delegation key fails with `CAP_SIG_INVALID` at the next check. Rotating the delegation key revokes all of its slips at once. Peers pick up the new card through the hash change.

## 7. Tests
- The card hash is stable across key orderings (JCS).
- A tampered card from the relay fails with `CARD_HASH_MISMATCH`.
- After rotation, an old slip fails with `CAP_SIG_INVALID` and a newly issued one passes.
- A card whose `owner` differs from the chain fails with `CARD_INVALID`.
