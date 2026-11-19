# Testing and acceptance
How the POC proves the success criteria (S1–S8) and negative tests (NT1–NT11) in [poc-scope.md](../poc-scope.md).

## 1. Test layers
| Layer | Where | What |
|---|---|---|
| Unit | `packages/trust-lib`, each app | Golden and negative vectors per format and reason code; core task and commitment state machines against a **fake crew**; calendar slot computation in its module |
| Integration | `deploy/test/docker-compose.yml`: relay bundle + 3 node bundles (Kash, Vishwas, Pete) on testnet | F1–F6 from [flows.md](flows.md) |
| Abstraction | A **fake crew** module + stub gateway in the test suite (no calendar code) | Proves a crew can be added without core changes: setup, grant, task, commitment, consent, remove |
| Adversarial | `tools/evil-peer`: a CLI that authenticates to the relay as a test agent and sends crafted envelopes and invocations | NT1–NT11 |
| Demo rehearsal | Real Browser Wallet + ID App on phones | The full demo script (M8) |

`tools/evil-peer` scenarios read fixtures from `tools/evil-peer/scenarios/*.json` and assert the **reason code** in the target node's trust log (`GET /api/events`).

## 2. Negative tests
| NT | Scenario (evil-peer) | Rejected by | Expected code |
|---|---|---|---|
| NT1 | Unregistered agent ID sends `session.hello` | relay auth / Vishwas n1-agent | `AGENT_NOT_FOUND` |
| NT2 | Revoked test agent sends `contact.request` | Vishwas n1-agent | `AGENT_REVOKED` |
| NT3 | Agent #99 (owner Y) claims to be Vishwas's, presenting #42's attestation | Kash n1-agent | `ATTESTATION_SUBJECT_MISMATCH` |
| NT4 | Pete's N1 sends `calendar.availability.request` without a slip | Vishwas n1-agent | `CAP_MISSING` |
| NT5a | Kash's N1 requests free/busy for a Monday | Vishwas n1-agent | `CAP_CAVEAT_VIOLATED` |
| NT5b | Kash's N1 invokes ② with `calendar.event.insert` | Vishwas n1-agent | `CAP_ACTION_NOT_ALLOWED` |
| NT6a | A slip ② edited to add `calendar.event.insert` (widened) | Vishwas n1-agent | `CAP_SIG_INVALID` |
| NT6a′ | A slip for Vishwas's calendar signed by another agent's delegation key | Vishwas n1-agent | `CAP_ROOT_MISMATCH` |
| NT6b | A slip with an unknown caveat type | Vishwas n1-agent | `CAP_CAVEAT_UNKNOWN` |
| NT7 | Replay of a captured `calendar.availability.request` | Vishwas n1-agent | `NONCE_REPLAY` |
| NT8 | Direct `POST /v1/invoke` to the Gateway without a valid slip ① (from inside the Docker network) | calendar-gateway | `CAP_MISSING` / `CAP_NOT_HOLDER` |
| NT9 | `calendar.event.insert` with slip ① but no consent | calendar-gateway | `CONSENT_REQUIRED` |
| NT10 | Kash's N1 invokes ② after Vishwas revokes it | Vishwas n1-agent | `CAP_REVOKED` |
| NT11 | Directory entry with a forged / expired attestation (fixture injected into a test relay) | Kash n1-agent (directory) | `ATTESTATION_BAD_SIGNATURE` / `ATTESTATION_EXPIRED` |

## 3. Success criteria checks
| S | Check | Evidence |
|---|---|---|
| S1 | F1 completes for Vishwas | Trust log "attested: …, owner = X"; VRA + VAA transactions; `Registered` event |
| S2 | Kash's directory shows Vishwas ✓ 18+ and Pete ✓ 18+ | `GET /api/directory` both `verified`; NT11 entry `rejected` |
| S3 | F2 with Pete: inbox → Allow → retry → slots | Trust log `OK_FIRST_CONTACT`, then `OK_INVOCATION` |
| S4 | Vishwas's slip ② to Kash verifies | `GET /api/capabilities` issued/received; `verifyChain` ok |
| S5 | F3 up to `commitment.tentative`, Vishwas away | No calendar write on either side; free/busy without titles |
| S6 | Consent on Vishwas's phone → event in both "My calendar" views | Consent record + VAA; two `eventId`s |
| S7 | NT1–NT11 all rejected with the codes above | evil-peer report: 11/11 pass |
| S8 | Audit anchor + VAAs found on chain | Transaction hashes resolve in a testnet explorer; hash chain verifies |

## 4. Traceability: roadmap → spec
| Milestone | Spec files |
|---|---|
| M1 | [trust-lib.md](shared/trust-lib.md), [identifiers-and-encoding.md](shared/identifiers-and-encoding.md), [node/README.md](node/README.md), [relay/README.md](relay/README.md) |
| M2 | [agent-card.md](shared/agent-card.md), [attestation.md](shared/attestation.md), [relay/verifier.md](relay/verifier.md), [relay-service.md](relay/relay-service.md) §3.3, [ui.md](node/ui.md) §3 |
| M3 | [relay-service.md](relay/relay-service.md) §3.1–3.2, [database.md](relay/database.md), [n1-agent.md](node/n1-agent.md) §5.1 |
| M4 | [messages.md](shared/messages.md), [n1-agent.md](node/n1-agent.md) §5.2, §5.5 |
| M5 | [capabilities.md](shared/capabilities.md), [crew-modules.md](shared/crew-modules.md), [crews/calendar.md](crews/calendar.md), [calendar-gateway.md](node/calendar-gateway.md) |
| M6 | [consent.md](shared/consent.md), [node/verifier.md](node/verifier.md), [n1-agent.md](node/n1-agent.md) §5.3–5.4 |
| M7 | [flows.md](flows.md) F4–F6, [ui.md](node/ui.md) §2 |
| M8 | this file |
