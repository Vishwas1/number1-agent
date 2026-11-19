# Number One: POC Scope
**Phase 1 · Calendar scheduling between two trusted agents** · v0.9 · Nov 2026

> Context: [business-overview.md](business-overview.md) · Architecture: [technical-solution.md](technical-solution.md) · Implementation details: [spec/](spec/README.md).

---

## 1. Goal
Show, end to end on **Concordium testnet**, a chat app where:
- Kash runs his own **local N1 node** (UI + server), sees a **directory of other people's N1 agents** (Vishwas, Pete…) with **verified names and age**, and starts a conversation with one of them
- **the agents negotiate a meeting while Vishwas is away from his desk** (his node keeps running)
- the meeting is **confirmed only when the approver (Vishwas) consents in the ID App on his phone**. Kash's own request counts as his intent.

Throughout, each side **verifies whose agent it's talking to**, and each agent **does only what its owner allowed**.

The POC proves the **trust layer**. The UI and AI are kept minimal on purpose.

## 2. POC simplifications of the architecture
| Area | Technical solution | POC |
|---|---|---|
| **Crew** | Crew modules (D12) + crew agents (e.g. Scheduler) | **One crew module: Calendar**, enabled by every user at setup. The N1 calls the Calendar Gateway directly with slip ①. Crew *agents* (TB2, slip ③) are not built. Calendar is built behind the generic crew interface, so other crews can be added later without core changes. |
| **Deployment** | Decentralised nodes + one relay | **Each user runs their own node** as a **Docker Compose bundle** (`ui`, `n1-agent`, `calendar-gateway`, `verifier`) with their own agent keys, LLM API key, calendar and verifier. The **only centralised piece is the company-hosted relay** (messaging, attester, directory index). |
| **Login** | Wallet-linked node | **No platform login.** The local app belongs to its owner. The wallet is used only during setup (and to rotate keys). |
| **Onboarding** | Several apps | **Browser Wallet** for everything (connect wallet → deploy N1 → name + age proof), then a **one-time ID App pairing** (QR scan) for later consents |

## 3. Success criteria
| # | The demo shows… | Trust boundaries |
|---|---|---|
| S1 | A user starts their **local node**, then in setup connects the **Browser Wallet**, deploys their N1 on CIS-8004, answers the relay attester's request with a **ZK proof of name and age (18+)** from the same wallet, receives a signed attestation, and pairs their ID App for later consent | TB1, TB4, TB5 |
| S2 | Kash's node fetches the directory **from the relay's index**, verifies each entry against CIS-8004 and the attestation (signature + VAA on chain), and shows the verified owner name and an 18+ badge | TB4, TB5, TB14 |
| S3 | **First contact + reactive grant:** Kash's N1 messages Pete's N1 (no slip). The request lands in Pete's inbox, and nothing is shared. Pete clicks **Allow** in his inbox, and his node signs slip ② with its delegation key. Kash's N1 retries and receives free/busy. | TB6, A2 |
| S4 | Vishwas has granted Kash's agent "free/busy on weekends" (the same inbox flow, done earlier) with a slip signed by his node's delegation key | TB6, A2 |
| S5 | **Owner away:** while Vishwas is away from his desk (his node running), his N1 shares free/busy only and holds a proposed slot as **tentative** | TB3, TB7, D11 |
| S6 | **Confirmation:** the consent request reaches Vishwas's **ID App on his phone**. The meeting is written to the calendars only after he approves. | TB10 |
| S7 | All negative tests (section 7) are rejected, with the reason shown in the trust log | TB4, TB6, TB8, TB10, TB12 |
| S8 | Proofs and the audit log are anchored on chain and can be looked up | TB11, TB13 |

## 4. In scope
| Area | POC implementation | TB |
|---|---|---|
| **Node bundle + relay** | The components in section 5, deployed as described there | TB1, TB3, TB13, TB14 |
| **Wallet setup** | During setup, the user connects the Browser Wallet and signs a link challenge, binding account X to their node. | TB1 |
| **ID App pairing** | During setup, the node pairs with the owner's ID App (WalletConnect QR), so later consent requests reach the phone | TB10 |
| **Agent deployment** | `n1-agent` generates the **invocation** and **delegation** Ed25519 keys. Register **one N1** per user on CIS-8004 (testnet), signed by X in the Browser Wallet, with the Agent Card (both public keys + `delegationPolicy`: free/busy + propose, at most 90 days, no re-delegation) and its hash | TB4, TB6 |
| **Name + age proof** | In setup, the Browser Wallet produces an **account-credential ZK proof**: reveal `firstName`, `lastName`, and prove `dob` = 18+ without revealing the date. The **relay's attester** creates a fresh request (context: N1 #42, account X, nonce), verifies the proof, confirms it resolves to X (the owner of the N1), anchors the VAA, and returns a **signed attestation** (agent, X, name, 18+, VAA reference, 90-day expiry). The agent is added to the directory index. | TB5, TB13 |
| **Directory** | **Indexed by the relay, verified by each node:** the relay lists attested Number One agents (attestation, Agent Card, presence). Each node verifies every entry: active in CIS-8004 and owned by X, card hash matches, attestation signed by the published attester key and not expired, VAA on chain. It then shows the verified name + 18+ badge. | D9, TB14 |
| **Agent verification** | Registry lookup + card hash check + attestation check + signed handshake | TB4 |
| **Crew choice + setup** | One-time, during setup: the owner picks crews from the catalog (POC: **Calendar**, pre-selected and required). The chosen crews and their per-crew `delegationPolicy` go into the Agent Card in the `register` transaction (no extra wallet step). After registration, the **Calendar** crew card connects the mock calendar and the node issues **slip ①** (own N1 → own calendar), signed by the delegation key. A **My crew** page lists installed and available crews. Crew modules aren't registered on CIS-8004 as agents. | TB3, TB6, D12 |
| **First contact + grant** | `contact.request` accepted from any verified N1 into the owner's **inbox**; nothing shared without a slip. **Reactive grant:** the owner clicks **Allow** on the request, picks the scope (actions, days, expiry; within `delegationPolicy`), and the node signs slip ② with the delegation key. It's delivered to the requester's N1 (`capability.grant`), which retries automatically. Shortcut: allow from the directory. | TB6 |
| **Capabilities** | Own minimal ZCAP implementation (no library; JCS + `eddsa-jcs-2022`). Slips ① (owner → own N1) and ② (owner → peer N1), signed by the delegation key; chain validation incl. root resolution (X → N1 → card → delegation key) and `delegationPolicy`; caveats: actions, days, expiry, tier, no re-delegation | TB6 |
| **Calendar Gateway** | Non-LLM, the only holder of calendar access; `calendar.freebusy.read`, `calendar.event.insert`; acts only on valid invocations from the N1. **Mock calendar adapter** (JSON/SQLite per node) plus a small **"My calendar" view** in the local UI, with **Add busy time** (owner-only `calendar.event.create` via slip ①, never delegable) for setting up demo scenarios live. | TB3 |
| **Owner-away negotiation** | Nodes keep running; N1s negotiate within existing slips; the agreed slot is held as tentative | D11 |
| **Confirmation** | **Approver only:** Vishwas's node (its own verifier) creates a consent request for the exact slot and pushes it to his ID App (phone), then verifies the reply. After he approves, his N1 sends Kash's N1 a signed confirmation with the VAA reference, and both Gateways insert the event. Pending consents also appear in the local UI. | TB10, TB13 |
| **Messaging** | Signed envelopes, nonces, fixed message types, via the shared relay | TB8 |
| **Sharing policy** | One tier only: **friend** (free/busy blocks) | TB7 |
| **Revocation** | Revoke an N1 (CIS-8004), revoke a slip, and rotate the delegation key (invalidates all its slips) | TB12 |
| **Audit** | Signed, hash-chained log; root anchored with `RegisterData` on demand | TB11 |
| **Chat UI** | Directory, conversation view (agent-to-agent messages), pending consents, trust log | — |

## 5. Components
### 5.1 Node bundle (one per user, `docker-compose.yml` on the user's machine)
| Service | Responsibilities | Holds | Talks to |
|---|---|---|---|
| **`ui`** (localhost) | Chat, directory, conversation view (agent-to-agent messages), pending consents, "My calendar" view, trust log. Setup wizard: Browser Wallet connect, choose crews, sign the N1 registration (card with keys, crews + per-crew policy), name + age proof, ID App pairing QR, crew setup (Calendar). My crew page. **Allow** buttons for grants. | Nothing sensitive | `n1-agent` (local API); Browser Wallet (browser extension, in the user's browser) |
| **`n1-agent`** | The **N1 agent**: conversation state machine and negotiation · **trust engine**: `verifyAgent`, signed handshake, envelope signing and checks (nonces, schemas), **slip issuing** (delegation key, on the owner's click only) and chain validation, revocation list, sharing policy (friend tier) · **directory client**: fetches the relay's index and verifies each entry against CIS-8004 and the attestation (cached) · **WalletConnect client**: pushes consent requests to the owner's ID App even when the UI is closed · **audit log**: hash-chained, signed, anchors on demand · **trust-log events** for the UI | Invocation key + **delegation key** (encrypted keystore; never exposed to the LLM), LLM API key, slips (①, ② issued and received), revocation list, audit log, WalletConnect session | `ui`, `calendar-gateway`, `verifier`, relay (outbound WSS), testnet node (gRPC), LLM provider, ID App (WalletConnect) |
| **`calendar-gateway`** | The only component with calendar access; fixed API `calendar.freebusy.read` / `calendar.event.insert`; acts only on a valid slip invocation (+ consent proof for inserts); returns free/busy blocks only | Calendar adapter: **mock store** (JSON/SQLite, pre-seeded); Google adapter as stretch | `n1-agent` only |
| **`verifier`** | Concordium Credential Verification Service: creates VPRs (anchors VRA) and verifies VPs (anchors VAA) for **its own owner's consent** only | **Node service account** key (not X), funded with test CCD | `n1-agent`, testnet node (gRPC) |

**Configuration (`.env`):** LLM API key · node service account key file · relay URL · testnet gRPC endpoint · CIS-8004 contract address · WalletConnect project ID.

**LLM boundary:** inside `n1-agent`, the LLM only interprets the owner's request and drafts replies. It **never holds keys, calls the Gateway, or makes trust decisions**. All of those are plain code in the trust engine. Peer messages reach it only as typed, validated fields (TB8).

### 5.2 Relay (company-hosted, the only centralised component)
| Responsibilities | Holds | Trust |
|---|---|---|
| **Messaging:** store-and-forward of signed envelopes over WSS; online presence. Nodes authenticate with agent-key signatures. **Attester:** a Credential Verification Service that requests and verifies the name + 18+ proof at onboarding, anchors the VAA, and signs attestations. **Directory index:** attested agents with their attestations and **Agent Cards** (cached, so peers can verify an agent while its node is offline). | Queued envelopes, attestations, cached cards; **attester signing key** (published) and **attester account** (anchors VRA/VAA, funded with test CCD) | **Trusted only for attestations** (POC); a false one is auditable via its anchored VAA. Untrusted for the rest: entries are checked against CIS-8004, cards against `metadata_hash`, envelopes by signature. It can hide agents or go down, but can't impersonate anyone in messages, slips or consent. |

### 5.3 External systems
| System | Used for |
|---|---|
| **Concordium testnet** (public gRPC node) | CIS-8004 registry (agents, metadata, events for the directory), account keys for signature checks, VRA/VAA/audit anchors |
| **Concordium Browser Wallet** | Owner account X: wallet link, N1 registration (publishes card keys + `delegationPolicy`), key rotation, account-credential name + age proof |
| **Concordium ID App** (phone) | Approver consent for meetings (via WalletConnect, paired once at setup) |
| **LLM provider** | Intent parsing and reply drafting (owner's own API key) |
| **WalletConnect relay** (public) | Transport between `n1-agent` and the ID App |

## 6. Simplifications (documented, not built)
| Area | In the POC | Production target |
|---|---|---|
| Crews | One crew module (Calendar), fixed at setup; add/remove later is specified but only demoed if the stretch crew is built | Crew catalog; add/remove from the My crew page |
| Crew agents | Not built; the N1 accesses the Gateway directly | N1 + crew agents with single-use slips (TB2) |
| Calendar | **Mock calendar only** (pre-seeded events) behind the Gateway. A Google Calendar adapter is a week-4 stretch goal, if time allows. | Google / Microsoft adapters |
| Agent key custody | Local encrypted keystore in the node | OS keychain / hardware-backed keys |
| Transport | TLS to the shared relay; envelopes signed but **not E2E-encrypted** (the relay operator could read them) | E2E encryption (TB9); a priority because the relay is shared |
| Agents | Simple LLM or scripted intent parsing; trust checks are plain code | Full orchestrator |
| Notifications | Consent request via the ID App's WalletConnect session; also listed in the local UI | Guaranteed push to the ID App |
| Identity attestation | One company attester (the relay), trusted for names + 18+; 90-day expiry | Several independent attesters, and/or a fresh peer-to-peer proof at first contact or grant |
| Directory | One relay index (can censor, can't forge) | Several indexers, or nodes cross-check by scanning the chain |
| Revocation state | In memory | Persisted; optional on-chain anchors |
| Audit anchoring | Manual "anchor now" | Hourly batches |
| Fees | Test CCD funded manually | Fee sponsorship model |

## 7. Demo script
**Onboarding (Vishwas, shown once)**
1. Start the local node (incl. its verifier) and open the UI on localhost. Configure the LLM API key and the node service account; the mock calendar is pre-seeded (e.g. "Football Sat 3pm"), and more busy time can be added live from My calendar.
2. Setup: connect the Browser Wallet → choose my crew (Calendar) → deploy my N1 (sign the transaction).
3. Setup: the relay's attester requests name + 18+. Vishwas answers from the Browser Wallet. The attester verifies it and returns a signed attestation, and the trust log shows "attested: Vishwas, 18+, owner = X" with the VAA transaction.
4. Setup: scan a QR code with the ID App to pair it for later consent requests.
5. Setup: on the **Calendar** crew card → connect the calendar (mock). The node issues slip ①, with no wallet prompt.

**Setup (shown briefly):** Kash and Pete are onboarded the same way. Vishwas allowed Kash's agent earlier, through the same inbox flow shown with Pete below (slip ②: weekend free/busy + propose, until Feb).

**Happy path**
1. Kash opens his local UI and the **directory**, fetched from the relay and verified by his node: Vishwas ✓ 18+, Pete ✓ 18+.
2. **Vishwas is away from his desk; his node is running.** Kash picks Vishwas: "Find a time for a drink next weekend."
3. The conversation view shows the agents talking: handshake → free/busy request → Vishwas's N1 reads through the Gateway → three slots that avoid "Football Sat 3pm" without revealing it.
4. Kash picks Sat 7pm. Vishwas's N1 replies *"tentative, waiting for Vishwas"*. Nothing is written to either calendar.
5. Vishwas's **phone** shows *"Drink with Kash, Sat 7pm"* in the ID App. He approves from wherever he is.
6. The event appears in both users' "My calendar" views. The audit is anchored, and the transaction is shown.

**First contact + reactive grant**
7. Kash messages **Pete's** N1, which has no slip. The request lands in Pete's inbox, and the trust log shows "no slip → nothing shared".
8. Pete opens his inbox and clicks **Allow** (weekend free/busy, until Feb). His node signs slip ② with its delegation key.
9. Kash's N1 receives the slip, retries automatically, and gets Pete's free/busy slots.

**Negative tests (each rejected, with the reason in the trust log)**
1. Unregistered agent sends a request
2. Revoked agent sends a request
3. Impostor N1 claiming to be Vishwas's (attestation names a different agent or owner)
4. Pete's N1 asks for free/busy without a slip
5. Request outside the slip (weekday, or `calendar.event.insert`)
6. Slip with a widened or unknown caveat
7. Replayed message
8. Calendar access without a valid slip (the Gateway rejects it)
9. Event insert without Vishwas's consent (a tentative slot can't be forced)
10. Request after Vishwas revokes slip ②
11. Directory entry with a forged or expired attestation (dropped by Kash's node)

## 8. Out of scope
- Crew agents (Scheduler etc.), crew introductions / handoff
- Voice, mobile app, onboarding UX, design polish
- Use cases beyond scheduling: dietary, grocery, payments, credentials
- More sharing tiers; standing pre-approvals
- CIS-8 / cross-chain agents, reputation, discovery beyond Number One agents in the registry
- Key and account recovery, multi-device support
- Production hosting, scaling, security audit

## 9. Environment & dependencies
| Item | Needed from | Status |
|---|---|---|
| CIS-8004 contract address (testnet) + metadata limits | Concordium | open |
| 3 test identities + 3 Browser Wallet accounts with test CCD (Kash, Vishwas, Pete) | team | open |
| **Per node:** Credential Verification Service (Docker) + a node service account with test CCD (3 accounts, separate from the owner accounts), for consent | team | open |
| Relay hosting (company), incl. the attester's Verification Service + attester account with test CCD + attester signing key | team | open |
| Docker on each user's machine | each user | open |
| LLM API key per node | each user | open |
| WalletConnect project ID (ID App pairing + consent) | team | open |
| **ID App surfaces incoming WalletConnect requests while closed** (notification) | Concordium | important (fallback agreed: approve when back at the desk) |
| **Browser Wallet account-credential proof** with name reveal + age range (G1) | Concordium / week 1 spike | **critical** |
| ID App consent (`consent:` context) on testnet | Concordium | **critical** (confirms every meeting) |

## 10. Plan (4 weeks)
| Week | Deliverable |
|---|---|
| **1** | **G1 spike** (name + age proof from the Browser Wallet) + ID App consent and notification check · local node skeleton incl. its own verifier · relay attester skeleton · wallet setup · N1 deployment on CIS-8004 |
| **2** | Name + age attestation in onboarding · relay (messaging + directory index) · Docker Compose bundle · signed envelopes · handshake + verification · directory verification · first contact |
| **3** | Slips + chain validation · Calendar Gateway + mock calendar + "My calendar" view · grant flow · owner-away negotiation + tentative hold |
| **4** | ID App pairing + phone confirmation · revocation · audit anchoring · chat UI + trust log · negative tests · demo video + `spec/` sync · *(stretch: Google Calendar adapter)* |

## 11. Risks
| Risk | Impact | Mitigation |
|---|---|---|
| G1 fails (the Browser Wallet can't produce account-credential proofs with name + age) | Names can't be bound to agents cryptographically | Fallback: ID App identity proof + node attestation; decided at the end of week 1 |
| Consent isn't tied to account X (G2): ID App proofs use the unlinkable identity credential | Someone who re-pairs a different ID App with the owner's node could approve meetings | Consent counts only through the session paired at setup (after linking X); plus a name match against the attestation if the ID App supports it (M0 spike). If not, the POC relies on the session binding alone. Recommend account-credential consent to Concordium. |
| ID App consent isn't available on testnet | Meetings can't be confirmed as designed | Fallback: confirmation signed by X in the Browser Wallet |
| The ID App doesn't surface WalletConnect requests while closed (no push) | Vishwas can't approve from his phone while away | **Accepted fallback:** Vishwas sees the pending consent in his local UI when he's back at his desk and approves from there (the ID App still gives the consent). Confirm with Concordium in week 1. |
| Local node must stay running | If Vishwas's node is off, negotiation pauses | The relay queues messages; the demo keeps both nodes running |
| Relay is the single centralised component | If it's down, nodes can't message, onboard or refresh the directory | Already-verified directory entries stay cached in each node; easy to restart or re-host |
| Relay is trusted for identity attestations | A compromised attester could vouch for a false name | Every attestation's VAA is anchored from a well-known account (auditable); production uses several attesters (section 6) |
| Shared relay sees message contents (no E2E in the POC) | The relay operator (the company) could read free/busy slots | Documented; E2E is the first production step (TB9) |
| Anyone with access to the owner's machine can use the local UI | They could chat, request meetings, or grant slips as the owner | Grants are capped by the `delegationPolicy` (never `calendar.event.insert`); confirming as approver needs the ID App on the phone |
| Delegation key sits on the owner's machine | If stolen, slips can be issued within the policy | Encrypted keystore; never reachable by the LLM; capped by `delegationPolicy`; rotate the key (X signs a card update) to invalidate all its slips |
| Node service account key sits on the user's machine | Could be stolen and its CCD spent | Low impact: it can only anchor data and holds no owner authority (slips need the delegation key, consent needs the ID App) |
| CIS-8004 draft changes, or testnet instability | Rework | Registry adapter; pin the contract version |
| Scope creep | Timeline | This document is the boundary; changes need sign-off |

## 12. Deliverables
1. **Docker Compose node bundle** (UI, N1 server, Calendar Gateway, verifier) and the company-hosted **relay** (messaging, attester, directory index), demoed with three users on testnet (source code)
2. Demo video covering onboarding, the happy path, first contact and all negative tests
3. [`spec/`](spec/README.md): implementation spec (APIs, data formats, validation rules)
4. Short findings note for Concordium: answers to G1–G10 and recommendations for CIS-8004, the Browser Wallet and the ID App
