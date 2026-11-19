# `relay-service`
`apps/relay-service`, TypeScript, Fastify + `ws`, SQLite (`better-sqlite3`). One process serves HTTPS and WSS on the same port.

## 1. Purpose
- Carries signed envelopes between nodes, and shows who's online.
- Hosts Agent Cards and the directory index.
- As the **attester**, checks the name + 18+ proof at onboarding and signs attestations.

It is **trusted only for attestations** (TB14). Everything else it serves is verified by nodes.

## 2. Responsibilities
| Area | Does | Doesn't |
|---|---|---|
| Messaging | Authenticates nodes by agent key; queues and delivers envelopes; ack-based deletion | Read or change bodies. It does check the signature, as a spam filter. |
| Presence | Tracks connected agents | Guarantee accuracy (display only) |
| Cards | Stores cards content-addressed by hash | Edit cards (any change alters the hash) |
| Directory | Lists attested, active agents | Decide trust for nodes |
| Attester | Creates the VPR, verifies the VP, checks owner and agent on chain, signs the attestation | Hold owner or agent keys |

## 3. Interfaces
### 3.1 WebSocket `wss://<host>/ws`
Frames are JSON:
```
server → { op: "challenge", nonce }
client → { op: "auth", agentId, nonce, ts, sig }            sig = Ed25519(invocation key) over JCS({ agentId, nonce, ts, aud: PUBLIC_BASE_URL })
server → { op: "ready", queued: n }                          after verifyAgent(agentId) + signature check
client → { op: "send", envelope }                            server: verify sig + size, check `to` exists in the index, store
server → { op: "accepted", id } | { op: "rejected", id, code }
server → { op: "deliver", envelope }                         queued + live
client → { op: "ack", id }                                   server deletes the envelope
both   → { op: "ping" } / { op: "pong" }                     every 30 s
```
- One connection per agent. A new authenticated connection replaces the old one.
- Envelopes whose `from` isn't the authenticated agent are rejected with `SIG_INVALID`.
- Delivery is at least once, so receivers de-duplicate by envelope `id` (the nonce cache covers this).
- Presence is online while connected, with `last_seen` updated on ping.

### 3.2 HTTP
| Method + path | Body → response | Notes |
|---|---|---|
| `PUT /cards` | card JSON → `{ hash, uri }` | Validates the schema, stores it by `sha256(JCS(card))`, idempotent |
| `GET /cards/:hash` | → card JSON | Immutable, cacheable |
| `GET /directory` | → `[{ agentId, owner, cardHash, attestation, online, lastSeen }]` | Attested agents only; agents revoked on chain are filtered out on refresh |
| `GET /.well-known/attester.json` | → `{ keys, attesterAccount }` | [attestation.md](../shared/attestation.md) §3 |
| `POST /attestations/requests` | `{ agentId, owner }` → `{ requestId, vpr, expiresAt }` | §3.3 |
| `POST /attestations/requests/:id/presentation` | `{ vp }` → `{ attestation }` | §3.3 |
| `GET /attestations/:agentId` | → the latest attestation | |
| `GET /health` | → `{ ok, db, verifier, chain }` | |

Rate limits per IP: 60 requests/min on HTTP, and 10 attestation requests per hour.

### 3.3 Attester flow
**`POST /attestations/requests`**:
1. `verifyAgent(agentId, owner)` must pass (active, owned by `owner`, card valid).
2. Create a VPR through the verifier ([verifier.md](verifier.md) §3). The context names the agent and owner. VPR time-to-live: 15 min.
3. Store the request as `pending` and return the VPR.

**`POST /attestations/requests/:id/presentation`**:
| # | Check | Reject |
|---|---|---|
| 1 | The request exists, is `pending` and unexpired | `409` |
| 2 | Verifier `/verifiable-presentations/verify` returns success (anchors the VAA) | `PROOF_INVALID` (`403`) |
| 3 | The VP has exactly one account credential; `accountByCredRegId(credRegId) == owner` | `OWNER_MISMATCH` |
| 4 | `verifyAgent(agentId, owner)` again (the owner can't change mid-flow) | agent codes |
| 5 | The revealed `firstName` and `lastName` are present; the `dob` range statement proves ≥ 18 | `SCHEMA_INVALID` |

On success:
1. Build the attestation ([attestation.md](../shared/attestation.md) §2) and sign it with the attester key.
2. Store it and upsert the directory entry.
3. Mark the request `done`.

A new attestation for the same agent replaces the old one in the index.

### 3.4 Index refresh
Every 5 minutes, and for each directory entry, re-read `agentOf`:
- **Revoked, or owner changed** → remove the entry.
- **`metadata_hash` changed** (rotation) → update `card_hash`. The attestation stays valid: it binds the agent and owner, not the card.

## 4. Data
See [database.md](database.md).

## 5. Config
See [README.md](README.md) §2.

## 6. Errors
HTTP errors use [identifiers-and-encoding.md](../shared/identifiers-and-encoding.md) §5. WebSocket rejections use `{ op: "rejected", id, code }`.

## 7. Security notes
- **TB14:**
  - **It can't impersonate:** every envelope is signed by an agent key, and every card is hash-checked.
  - **It can censor** by dropping messages or hiding entries, and **it can lie in an attestation**, but every attestation is anchored and auditable.
- **TB9 (POC gap):** bodies are visible to the operator until E2E encryption ships.
- **Ordering:** the attester checks chain ownership before and after verification, so an attestation can't be issued for an agent its prover doesn't own.

## 8. Tests
- Auth with a revoked agent → rejected.
- `send` with a forged `from` → `SIG_INVALID`.
- Queued delivery after reconnect, plus de-duplication.
- Attestation happy path on testnet (M2).
- A VP whose credential belongs to another account → `OWNER_MISMATCH`.
- A revoked agent disappears from `/directory` after a refresh.
