# Messages
Agent-to-agent protocol over the relay. Implements D10, D11 and TB8 (TB9 in production).

## 1. Envelope
```json
{
  "v": 1,
  "id": "urn:uuid:9a1d7c4e-2f63-4b0a-8e15-7c2d9f3b6a01",
  "type": "calendar.availability.request",
  "from": "agent:ccd:testnet:9876/11",
  "to": "agent:ccd:testnet:9876/2a",
  "session": "sess:9f04c2",
  "seq": 3,
  "ts": "2026-11-21T18:02:11Z",
  "nonce": "c1f04b7e9a2d6f38",
  "body": {},
  "sig": {
    "kid": "agent:ccd:testnet:9876/11#invocation",
    "alg": "Ed25519",
    "value": "z3k9EnvelopeSignature"
  }
}
```
- `sig.value` = Ed25519 over `JCS(envelope without sig)` with the sender's **invocation key**.
- `nonce`: 16 random bytes, hex.
- `session`: required for every type except `session.hello`.
- Production adds E2E encryption of `body` to the recipient's encryption key (HPKE, X25519). The POC sends `body` in clear over TLS (poc-scope §6).

## 2. Receive checks (in order)
| # | Check | Reject with |
|---|---|---|
| 1 | Envelope schema; `v == 1`; `type` in the allowed set (§4) | `SCHEMA_INVALID` / `TYPE_NOT_ALLOWED` |
| 2 | `to` is this node's agent | drop silently |
| 3 | `verifyAgent(from)` ([agent-card.md](agent-card.md) §5) and `verifyAttestation` | agent / attestation codes |
| 4 | `sig` verifies with `from`'s card invocation key; `kid` matches | `SIG_INVALID` |
| 5 | `|now − ts| ≤ 5 min` | `TS_SKEW` |
| 6 | `nonce` unseen (kept for 10 min) | `NONCE_REPLAY` |
| 7 | Session known and unexpired (except `session.hello`); `seq` strictly increasing | `SESSION_UNKNOWN` / `SEQ_OUT_OF_ORDER` |
| 8 | `body` matches the type's JSON schema; unknown fields → reject | `SCHEMA_INVALID` |
| 9 | Type-specific authorization (§4) | capability / consent codes |

Every rejection is logged to the trust log and audit log. The sender receives an `error` message, except for steps 2–4, where the node never replies to unverified senders.

## 3. Handshake
```
A → B  session.hello    { nA }                      (no session field)
B → A  session.welcome  { nA, nB }                  (session = sess:sha256(nA ‖ nB))
A → B  session.confirm  { nB }
```
- Each side runs `verifyAgent` + `verifyAttestation` on the other before replying.
- Nonces are 32 random bytes, hex. The session TTL is **1 hour**, and `seq` restarts at 1 for each session.

## 4. Message types
**Core types**, the same for every crew:
| Type | Direction | Body | Authorization |
|---|---|---|---|
| `session.hello` / `welcome` / `confirm` | both | see §3 | verified agent |
| `contact.request` | requester → approver | `{ requestId, crew, topic, note?, hint? }` (`hint` is crew-defined) | **any verified agent** (first contact). Stored in the inbox; reply `contact.request.ack { requestId, status: "pending_permission" \| "has_permission" }` |
| `capability.grant` | approver → requester | `{ capability }` (slip ②) | sender must be the slip's issuer (the root owner's N1) |
| `capability.revoke` | approver → requester | `{ capabilityId, reason }` | sender must be the issuer |
| `commitment.propose` / `tentative` / `confirmed` / `declined` | see [consent.md](consent.md) §3 | | invocation of the crew's `proposeAction` |
| `error` | both | `{ refId, code, message }` | — |

**Crew types** (`<crewId>.*`, e.g. `calendar.availability.request`) are defined by crew modules for their negotiation phase ([crew-modules.md](crew-modules.md), [crews/calendar.md](../crews/calendar.md) §6). They're accepted only for crews listed in **both** agents' cards, otherwise `TYPE_NOT_ALLOWED`. Each crew type declares the action its sender must invoke.

- **Peer text never reaches the LLM as instructions.** Free-text fields (`topic`, `note`, crew fields such as a title) are plain strings, at most 200 characters, shown to the owner as quoted data (TB8).
- Messages from an agent without a slip: only `session.*`, `contact.request` and `error` are accepted. Anything else fails with `CAP_MISSING` (NT4).

## 5. Tests
- Each check in §2 has a negative test (`tools/evil-peer`): bad signature, skewed `ts`, replayed nonce (NT7), unknown type, extra body field, no session.
- A message from an unregistered agent (NT1) or a revoked one (NT2) gets no reply, and the trust log shows the reason.
