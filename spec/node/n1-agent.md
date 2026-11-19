# `n1-agent`
The owner's Number One agent. It's the only service in the node that holds agent keys and talks to the outside world. `apps/n1-agent`, TypeScript, Fastify, SQLite (`better-sqlite3`).

## 1. Purpose
Acts for the owner: verifies peers, issues and checks slips, runs crew tasks and commitments, asks the owner for consent, and keeps the audit log. **The core knows no crew.** Calendar logic lives in the calendar crew module ([crews/calendar.md](../crews/calendar.md)). Every trust decision is plain code calling `trust-lib`. The LLM only interprets the owner's text and drafts wording.

## 2. Responsibilities (modules)
| Module | Does | Spec |
|---|---|---|
| `setup` | Wallet-link challenge; crew choice; key generation; card build (keys + `crews` + per-crew policy) + upload; register params for the wallet; read the token ID; attestation start/complete; WalletConnect pairing; crew setup | [flows.md](../flows.md) F1 |
| `crews` | Load crew modules (POC: `calendar`), register their actions, caveats and message types; run `setup()` / `teardown()`; route verified crew messages to the module; serve crew views | [crew-modules.md](../shared/crew-modules.md) |
| `trust` | `verifyAgent`, `verifyAttestation`, envelope checks, handshake, `verifyChain` / `verifyInvocation`, revocation list, nonce and session stores | shared/* |
| `directory` | Fetch the relay index, verify every entry, cache, merge presence | §5.1 |
| `relay-client` | WSS connect + auth, send / receive / ack, reconnect with backoff | [relay-service.md](../relay/relay-service.md) §3.1 |
| `inbox` | Store `contact.request` from agents without a slip; **Allow** → slip issuer | §5.2 |
| `slips` | Issue (owner click only), store issued and received, revoke, policy check | [capabilities.md](../shared/capabilities.md) §8 |
| `tasks` | Generic owner tasks (one crew intent with one peer): state, options from the crew, owner's choice | §5.3 |
| `commitments` | Generic commitment protocol: propose, hold, consent, execute via the module, confirm | [consent.md](../shared/consent.md) |
| `consent` | Build consent VPR via the verifier, push over WalletConnect, verify the VP, store the consent record | [consent.md](../shared/consent.md) |
| `gateway-client` | Invoke a crew's own-N1 slip at that crew's gateway (`POST /v1/invoke`) | [crew-modules.md](../shared/crew-modules.md) §4a |
| `llm` | Owner text → typed intent; draft display text. Never sees keys, never calls trust or gateway functions. | §5.4 |
| `audit` | Hash-chained signed log; anchor on demand | §5.5 |
| `events` | Trust-log and UI event stream (SSE) | §3.3 |

## 3. Interfaces
### 3.1 Local API (for `ui` only, `127.0.0.1:7100`, JSON)
| Method + path | Body → response | Notes |
|---|---|---|
| `GET /api/setup/status` | → `{ step, account?, agentId?, attested, idAppPaired, crews: [{ id, ready }] }` | Wizard resume |
| `POST /api/setup/link/challenge` | → `{ message }` | `"Number One node link\nnode: <nodeId>\nnonce: <hex>\nexpires: <ts>"` |
| `POST /api/setup/link` | `{ account, signature }` → `{ ok }` | `verifyAccountMessage` against X's chain keys |
| `GET /api/crews` | → `{ available: [{ id, title, icon, defaultPolicy }], installed: [{ id, status, peersAllowed }] }` | Catalog = bundled modules; installed = own card |
| `POST /api/setup/crews` | `{ crews: ["calendar"], policies? }` → `{ ok }` | One-time choice before registration; POC requires `calendar` |
| `POST /api/setup/agent/prepare` | `{ displayName }` → `{ card, cardHash, agentUri, registerParams }` | Generates keys, builds the card with the chosen crews and policies, uploads it to the relay, builds `register` params |
| `POST /api/setup/agent/registered` | `{ txHash }` → `{ agentId }` | Waits for finalisation; reads the `Registered` event |
| `POST /api/setup/attestation/start` | → `{ requestId, vpr }` | Proxies the relay `POST /attestations/requests` |
| `POST /api/setup/attestation/complete` | `{ requestId, vp }` → `{ attestation }` | Proxies the relay; verifies the result with `verifyAttestation` |
| `POST /api/setup/idapp/pair` | → `{ wcUri }`; completion via SSE `idapp.paired` | WalletConnect SignClient; stores the session topic |
| `POST /api/crews/:id/setup` | crew-specific body (calendar: `{ adapter: "mock" }`) → `{ slipId }` | Runs the module's `setup()`; issues its own-N1 slip ① |
| `POST /api/crews/:id/add/prepare` · `/remove/prepare` | → `{ card, setAgentUriParams, affectedPeers }` | Later changes; the wallet signs `setAgentURI`, then `/api/crews/:id/add/done` or `/remove/done { txHash }` runs `setup()` / `teardown()` |
| `GET /api/directory` | → `[{ agentId, name, ageOver18, online, status, reason? }]` | Verified entries; failed entries appear with their reason |
| `GET /api/inbox` | → `[{ requestId, from, fromName, crew, topic, hint, receivedAt }]` | |
| `GET /api/crews/:id/grant-template` | → the crew's `defaultGrant` (fields for the Allow dialog) | |
| `POST /api/inbox/:requestId/allow` | `{ crew, actions, caveats, expiresDays }` → `{ slipId }` | Builds slip ② for that crew's resource, checks the policy, signs, sends `capability.grant` |
| `POST /api/peers/:agentId/allow` | same as above | Shortcut from the directory |
| `GET /api/capabilities` | → `{ issued: [...], received: [...] }` | |
| `POST /api/capabilities/:id/revoke` | → `{ ok }` | Revocation list + `capability.revoke` |
| `POST /api/conversations` | `{ peer, text }` → `{ conversationId, taskId? }` | LLM → crew intent → task |
| `GET /api/conversations/:id` | → `{ messages, task? }` | Includes agent-to-agent messages for the conversation view |
| `GET /api/tasks/:id` | → `{ id, crew, peer, state, options: [...], renderer }` | `renderer` names the crew's option renderer (calendar: `slot-list`) |
| `POST /api/tasks/:id/choose` | `{ optionId }` → `{ commitmentId }` | The owner's pick = requester intent; the module's `fromOption()` builds the commitment, then `commitment.propose` |
| `GET /api/consents` | → pending and finished consents | |
| `POST /api/consents/:id/resend` | → `{ ok }` | Re-push to the ID App |
| `GET /api/crews/:id/views/:view` | query → view data | Crew views (calendar: `my-calendar`, via `calendar.event.list`) |
| `POST /api/crews/:id/views/:view/actions/:action` | action body → result | Owner-only view actions, from UI clicks only (calendar: `add-busy` → `calendar.event.create`) |
| `POST /api/audit/anchor` | → `{ txHash }` | |
| `POST /api/keys/rotate/prepare` | `{ rotate: ["delegation", …], policy? }` → `{ card, setAgentUriParams }` | Then the wallet signs `setAgentURI` |
| `POST /api/keys/rotate/done` | `{ txHash }` → `{ ok }` | Switches keys after finalisation |
| `POST /api/agent/revoke/prepare` | `{ reason }` → `{ revokeParams }` | Then the wallet signs `revoke` |
| `GET /api/events` | SSE stream | `trust` (reason codes), `message`, `inbox`, `task`, `commitment`, `consent`, `presence`, `idapp.paired` |

- **Auth:** the server binds to localhost only, and every request carries a per-install token from `data/ui-token`, which the `ui` container reads at start (TB1).
- **Errors:** [identifiers-and-encoding.md](../shared/identifiers-and-encoding.md) §5.

### 3.2 Internal API (Docker network only, separate listener `:7101`, for crew gateways)
| Method + path | Response |
|---|---|
| `GET /internal/identity` | `{ ownerAccount, agentId }` |
| `GET /internal/commitments/:id` | `{ id, crew, role, commitmentHash, state, ownerChoiceAt }` |

### 3.3 Trust-log event
```json
{
  "ts": "2026-11-21T18:02:11Z",
  "kind": "invocation",
  "peer": "agent:ccd:testnet:9876/11",
  "decision": "reject",
  "code": "CAP_CAVEAT_VIOLATED",
  "detail": "PermittedDays: window includes Mon 30 Nov",
  "ref": "urn:uuid:2d8b6f41-93a7-4c0e-b5d2-6e1f0a9c7b33"
}
```

## 4. Data (`data/n1.db`)
| Table | Key columns |
|---|---|
| `setup` | `step`, `account`, `agent_id`, `card_hash`, `attestation_json`, `wc_topic` |
| `crews` | `id` PK, `version`, `status` (`chosen` \| `active` \| `removed`), `own_slip_id`, `config_json` |
| `peers` | `agent_id` PK, `owner`, `name`, `age_over18`, `card_hash`, `attestation_id`, `verified_at`, `status`, `last_reason` |
| `sessions` | `session_id` PK, `peer`, `expires_at`, `send_seq`, `recv_seq` |
| `nonces` | `nonce` PK, `expires_at` |
| `inbox` | `request_id` PK, `from_agent`, `body_json`, `status` (`pending` \| `allowed` \| `ignored`), `received_at` |
| `capabilities` | `id` PK, `direction` (`issued` \| `received`), `peer`, `slip_json`, `expires_at`, `revoked_at` |
| `rate_counters` | `cap_id`, `day`, `count` |
| `conversations` | `id` PK, `peer`, `created_at` |
| `messages` | `id` PK, `conversation_id`, `direction`, `type`, `envelope_json`, `ts` |
| `tasks` | `id` PK, `crew`, `peer`, `intent`, `args_json`, `state`, `options_json`, `updated_at` |
| `commitments` | `id` PK, `task_id`, `crew`, `role` (`requester` \| `approver`), `peer`, `commitment_json`, `commitment_hash`, `state`, `owner_choice_at`, `expires_at` |
| `crew_<id>_*` | Module-scoped tables (e.g. `crew_calendar_cache`) |
| `consents` | `id` PK, `commitment_id`, `vpr_json`, `status`, `audit_record_id`, `audit_record_json`, `vaa_tx`, `expires_at` |
| `audit_log` | `seq` PK, `prev_hash`, `entry_json`, `hash`, `sig`, `anchored_tx` |

## 5. Behaviour
### 5.1 Directory
1. `GET <RELAY_URL>/directory` every 60 s, and on demand.
2. For each entry other than itself:
   - `verifyAgent(agentId, entry.owner)`
   - `verifyAttestation(entry.attestation)`
3. Store the result in `peers`, with `status` = `verified` or `rejected` plus the reason.
4. Only verified entries are offered for contact. Rejected ones are shown greyed out with their reason (NT11 demo).
5. Presence comes from the relay and is used for display only.

### 5.2 First contact and grants
- **Incoming `contact.request` from a peer without a valid received-by-them slip:**
  - store it in `inbox`
  - reply `contact.request.ack { status: "pending_permission" }`
  - emit the trust event `OK_FIRST_CONTACT`
- **Owner clicks Allow:**
  - build slip ② for the requested crew's resource, from the dialog (defaults: the crew's `defaultGrant`)
  - run `checkPolicy`, sign with the delegation key, store it as `issued`
  - send `capability.grant`, and mark the inbox entry `allowed`
- **Requester receives `capability.grant`:** `verifyChain`, store it as `received`, then **automatically retry** the pending task for that peer.

### 5.3 Tasks and commitments
A **task** is one crew intent with one peer, started by the owner (e.g. calendar `find_time` with Vishwas). Core states:
```
open ──contact.request.ack pending_permission──► waiting_permission ──capability.grant──► open (auto-retry)
open ──module.start(task)──► negotiating ──module reports options──► options_shown
options_shown ──owner choose (optionId)──► committing ──commitment.* (consent.md §3)──► confirmed | declined
any ──error / 48 h timeout──► closed
```
- **Negotiating:** the core hands the task to the module, which sends and handles its own crew messages (calendar: `calendar.availability.*`). The core still runs every envelope check, `verifyInvocation` and rate limit before the module sees a message.
- **Committing:** the generic commitment protocol in [consent.md](../shared/consent.md) §3, with module hooks for `fromOption`, `validate`, `conflicts`, `recheck`, `text` and `execute`.
- **Approver side:** no task. Incoming crew messages go to the module's handler. Incoming `commitment.propose` goes to the core `commitments` module.

### 5.4 LLM boundary
- **Input:** the owner's chat text, a list of verified peer names and their crews, and the `intents` schemas of installed crews.
- **Output:** JSON validated against a schema, otherwise rejected:
  `{ crew: <installed crew id> | null, intent: <an intent of that crew> | "smalltalk" | "unknown", peerAgentId?, args }`. `args` must match that crew's intent schema (e.g. calendar `find_time`, [crews/calendar.md](../crews/calendar.md) §5), and the peer must have the crew too.
- **Peer content** (`topic`, `title`, `note`) may be passed in only for summarising, as quoted data and never as instructions. The LLM output never becomes a protocol message without passing through the typed builders.
- **`LLM_MODE=openai`:** the OpenAI Chat Completions API (`openai` npm package) with `response_format: json_schema`, so the intent comes back as typed JSON. It uses the owner's own `OPENAI_API_KEY`.
- **`LLM_MODE=scripted`:** a regex parser gives deterministic demo runs (R4).
- The LLM has no tools: no function calling into `slips`, `gateway-client`, `commitments` or `consent`.

### 5.5 Audit log
- **What's logged:** every trust decision, every slip issued, revoked or received, every commitment and consent, and every gateway call.
- **Entry format:** `{ seq, ts, kind, ref, decision, code, prevHash }`.
  - `hash = sha256(JCS(entry))`
  - `sig` = Ed25519 with the invocation key
- **Anchoring:** `POST /api/audit/anchor` sends `RegisterData("N1-audit:" + agentId + ":" + seq + ":" + hash)` from the node service account (`/keys/service.export`) and stores `anchored_tx`. Because the log is a hash chain, one anchor covers every earlier entry.

## 6. Config
`.env` from [README.md](README.md) §2. Fixed timeouts: session 1 h, nonce TTL 10 min, consent 24 h, task 48 h, directory refresh 60 s, `agentOf` cache 60 s.

## 7. Security notes
- **TB1:** the API is localhost-only with a token. Grants are capped by `delegationPolicy`, and confirmations need the ID App.
- **TB6:** slips are issued only from API calls that come from UI clicks. No code path lets the LLM or a peer message trigger issuing.
- **TB8:** the full envelope check runs before any business logic.
- **Keys:** the keystore is decrypted into memory at start. Keys are never logged and never sent to the LLM.

## 8. Tests
- Unit: task and commitment state machines (all transitions + timeouts) against a **fake crew module**, so the core is tested without the calendar; the LLM schema guard. Calendar logic is tested in its module ([crews/calendar.md](../crews/calendar.md) §10).
- Integration (two nodes + relay, docker compose): F2 and F3 from [flows.md](../flows.md), plus NT1–NT11 via `tools/evil-peer` ([testing.md](../testing.md)).
