# Identifiers and encoding
Conventions every other spec uses. Implemented in `trust-lib` ([trust-lib.md](trust-lib.md)).

## 1. Identifiers
| Thing | Format | Example |
|---|---|---|
| Account | `ccd:testnet:acc:<base58 address>` | `ccd:testnet:acc:4VishwasXq…` |
| Agent | `agent:ccd:testnet:<contractIndex>/<tokenIdHex>` | `agent:ccd:testnet:9876/2a` (#42) |
| Agent key | `<agentId>#invocation` · `#delegation` · `#encryption` | `agent:ccd:testnet:9876/2a#delegation` |
| Resource (per crew) | `<crewId>:ccd:testnet:<base58 address of X>` | `calendar:ccd:testnet:4VishwasXq…` |
| Action / crew message type | `<crewId>.<name>` | `calendar.freebusy.read`, `calendar.meeting.propose` |
| Root capability | `urn:zcap:root:` + `encodeURIComponent(<resource id>)` | `urn:zcap:root:calendar%3Accd%3Atestnet%3A4Vis…` |
| Capability, message, meeting, consent | `urn:uuid:<uuid v4>` | `urn:uuid:5b1c…` |
| Session | `sess:` + hex sha256(nA ‖ nB) | `sess:9f04…` |
| Attester key | `<RELAY_PUBLIC_URL>/.well-known/attester.json#<keyId>` | `https://relay.number-one.dev/.well-known/attester.json#k1` |

- The CIS-8004 contract address is `<index, subindex>`; subindex is always 0 on testnet, so only the index appears in agent IDs.
- Token IDs are the CIS-8004 `AgentTokenId`, written as lowercase hex without leading zeros.
- In docs and examples, `#42` / `#17` are shorthand for the full agent ID.

## 2. Canonical JSON and hashing
| Rule | Detail |
|---|---|
| Canonicalisation | **JCS** (RFC 8785) everywhere a hash or signature covers JSON |
| Hash | SHA-256. Written as lowercase hex unless a field says otherwise. |
| What is signed | `JCS(object without its signature field)` (details per format) |
| Numbers | Integers only in signed objects (no floats) |
| Strings | UTF-8, NFC-normalised before canonicalisation |

## 3. Keys and signatures
| Key | Algorithm | Encoding of public key |
|---|---|---|
| Invocation, delegation | Ed25519 | multibase base58btc of multicodec `ed25519-pub` (0xed01) → `z6Mk…` |
| Encryption | X25519 | multibase base58btc of multicodec `x25519-pub` (0xec01) → `z6LS…` |
| Attester | Ed25519 | as above |
| Owner account X | Concordium account keys (Ed25519) | read from chain with `getAccountInfo` |

- Signatures in proofs: `proofValue` = multibase base58btc (`z…`) of the 64-byte Ed25519 signature.
- Signatures in envelopes: see [messages.md](messages.md).
- Data Integrity proofs use the W3C `eddsa-jcs-2022` cryptosuite: `hashData = sha256(JCS(proofConfig)) ‖ sha256(JCS(document without proof))`, then Ed25519-sign `hashData`.

## 4. Time
- All timestamps are RFC 3339 in UTC with `Z` (`2026-11-28T19:00:00Z`).
- Meeting slots are stored in UTC; the UI renders them in the owner's local time zone.
- Allowed clock skew on received messages: **±5 minutes**.

## 5. Error format (HTTP APIs)
```json
{
  "error": {
    "code": "CAP_CAVEAT_VIOLATED",
    "message": "Requested window includes a weekday; slip allows Sat, Sun only",
    "details": { "capability": "urn:uuid:cap-017", "caveat": "PermittedDays" }
  }
}
```
`code` is always one of [reason-codes.md](reason-codes.md). HTTP status: `400` schema, `401` signature/auth, `403` trust or capability rejection, `404` not found, `409` state conflict, `502` upstream (chain, verifier, relay) failure.
