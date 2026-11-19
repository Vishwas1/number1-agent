# Reason codes
Every accept or reject decision carries one code. The trust log shows it, HTTP errors return it ([identifiers-and-encoding.md](identifiers-and-encoding.md) §5), and negative tests assert it ([testing.md](../testing.md)).

## Agent identity (TB4, TB5)
| Code | Meaning |
|---|---|
| `AGENT_NOT_FOUND` | No CIS-8004 record for this agent ID |
| `AGENT_REVOKED` | CIS-8004 status is `Revoked` |
| `OWNER_MISMATCH` | On-chain owner differs from the expected owner |
| `CARD_UNAVAILABLE` | Agent Card can't be fetched from the relay cache |
| `CARD_HASH_MISMATCH` | `sha256(JCS(card))` ≠ on-chain `metadata_hash` |
| `CARD_INVALID` | Card fails its schema, or `card.owner` ≠ on-chain owner |
| `ATTESTATION_MISSING` | No attestation for the agent |
| `ATTESTATION_BAD_SIGNATURE` | Not signed by the pinned attester key |
| `ATTESTATION_EXPIRED` | Past `expires` |
| `ATTESTATION_SUBJECT_MISMATCH` | Attestation names a different agent or owner |
| `ATTESTATION_VAA_NOT_FOUND` | Evidence VAA missing on chain, or not from the attester account |
| `PROOF_INVALID` | The verifier rejected the onboarding VP (attester only) |

## Messages (TB8)
| Code | Meaning |
|---|---|
| `SIG_INVALID` | Envelope signature fails |
| `TS_SKEW` | `ts` outside ±5 min |
| `NONCE_REPLAY` | Nonce already seen |
| `SEQ_OUT_OF_ORDER` | `seq` not strictly increasing within the session |
| `SESSION_UNKNOWN` | No handshake / expired session |
| `TYPE_NOT_ALLOWED` | Message type not in the allowed set |
| `SCHEMA_INVALID` | Body fails its JSON schema |

## Capabilities (TB6, TB12)
| Code | Meaning |
|---|---|
| `CAP_MISSING` | Action needs a slip and none was presented |
| `CAP_SIG_INVALID` | Slip or invocation signature fails |
| `CAP_ROOT_MISMATCH` | Chain doesn't end at the expected root / controller |
| `CAP_NOT_HOLDER` | Invoker isn't the slip's `controller` |
| `CAP_ACTION_NOT_ALLOWED` | Action not in `allowedAction` |
| `CAP_EXPIRED` | Past `expires` |
| `CAP_REVOKED` | Slip (or an ancestor) is on the revocation list |
| `CAP_CAVEAT_VIOLATED` | Params break a caveat |
| `CAP_CAVEAT_UNKNOWN` | Caveat type not in the registry |
| `CAP_ATTENUATION_VIOLATED` | Child slip is wider than its parent |
| `CAP_POLICY_EXCEEDED` | Slip is wider than the issuer's `delegationPolicy` |
| `CAP_RATE_LIMITED` | `RateLimit` caveat exceeded |

## Commitments (TB10)
| Code | Meaning |
|---|---|
| `CONSENT_REQUIRED` | `calendar.event.insert` without an owner confirmation |
| `CONSENT_INVALID` | Consent record fails verification |
| `CONSENT_COMMITMENT_MISMATCH` | Consent is for a different commitment (e.g. another slot) |
| `CONSENT_EXPIRED` | Consent request timed out |
| `COMMITMENT_UNAVAILABLE` | Commitment no longer possible: conflicts with a hold, or the crew's re-check fails (calendar: slot taken) |

## Accept codes
| Code | Meaning |
|---|---|
| `OK_VERIFIED_AGENT` | Agent passed `verifyAgent` |
| `OK_FIRST_CONTACT` | `contact.request` accepted into the inbox, nothing shared |
| `OK_INVOCATION` | Invocation passed all checks |
| `OK_CONSENT` | Consent verified |
