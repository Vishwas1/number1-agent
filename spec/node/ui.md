# `ui`
Local web app on `http://localhost:3000`. React + Vite, served by a small static server. It talks only to `n1-agent` (local API) and to the Browser Wallet extension in the owner's browser. **It holds nothing sensitive.** UX polish is out of scope (poc-scope §8).

## 1. Purpose
Lets the owner set up their node, see the directory, allow peers, chat with their N1, watch agent-to-agent messages, follow pending approvals and their calendar, and see every trust decision.

## 2. Screens
| Screen | Shows | Actions → API |
|---|---|---|
| **Setup wizard** (first run, resumable) | Steps 1–6 with status | see §3 |
| **My crew** | Crew cards from `GET /api/crews`: icon, title, status (Active / Available), peers allowed | **Add** / **Remove** (confirmation lists affected peers; wallet signs `setAgentURI`) → `/api/crews/:id/add|remove/*` |
| **Directory** | Verified Number Ones: name, 18+ badge, crew icons (from their card), online dot. Rejected entries greyed out with their reason. | **Connect** → `POST /api/conversations` · **Allow…** → `POST /api/peers/:id/allow` |
| **Inbox** | First-contact requests: who, topic, window | **Allow** (scope dialog: actions, days, expiry, defaults from the policy) → `POST /api/inbox/:id/allow` · **Ignore** |
| **Chat** | Owner ↔ own N1 messages; a task's options, drawn by the crew's option renderer (calendar: `slot-list` buttons) | Send text → `POST /api/conversations` · pick an option → `POST /api/tasks/:id/choose { optionId }` |
| **Conversation view** | Agent-to-agent envelopes for this task: type, direction, decision | read-only |
| **Pending approvals** | Commitments waiting for the owner (crew icon, the consent text, short commitment hash, time left on the hold) | **Approve** → `approve/start` → Browser Wallet Verify & Access request → `approve/complete` · **Decline** |
| **Crew views** | One tab per view of each installed crew, from `GET /api/crews/:id/views/:view` (calendar: **My calendar** week view, events tagged seed / busy / meeting) | View actions declared by the crew → `POST /api/crews/:id/views/:view/actions/:action` (calendar: **Add busy time**) |
| **Permissions** | Issued and received slips: peer, actions, days, expiry | **Revoke** → `POST /api/capabilities/:id/revoke` |
| **Trust log** | Live stream of trust events: time, peer, decision, code, detail | filter by peer / decision |
| **Settings** | Agent ID, card hash, attestation expiry, audit anchors | **Anchor audit now** · **Rotate keys** · **Revoke my N1** |

The live updates come from SSE `GET /api/events`. The UI shell knows no crew: crew cards, the Allow dialog fields (`GET /api/crews/:id/grant-template`), option renderers and views all come from the crew list. A small renderer registry in `ui` maps renderer names (`slot-list`) to components, and adding a crew adds one renderer.

## 3. Setup wizard and wallet calls
The Browser Wallet is reached through `@concordium/browser-wallet-api-helpers` (`detectConcordiumProvider()`).
| Step | UI does | Wallet call |
|---|---|---|
| 1. Connect wallet | `provider.connect()` → account X; `POST /api/setup/link/challenge`; sign; `POST /api/setup/link` | `signMessage(X, challenge)` |
| 2. Choose my crew | Crew cards from `GET /api/crews`; Calendar is pre-selected and required in the POC; shows each crew's sharing limits (`defaultPolicy`) → `POST /api/setup/crews` | — |
| 3. Create my Number One | `POST /api/setup/agent/prepare` → `registerParams`; send; `POST /api/setup/agent/registered { txHash }` | `sendTransaction(X, Update, { address: CIS8004, receiveName: "<contract>.register", maxContractExecutionEnergy, message: registerParams }, schema)` |
| 4. Prove name + 18+ | `POST /api/setup/attestation/start` → VPR; ask the wallet for the VP; `POST /api/setup/attestation/complete { vp }` | Verifiable-presentation request with the VPR, `source: accountCredential` (exact Browser Wallet method confirmed in the **G1 spike**) |
| 5. Set up my crew | For each chosen crew: its setup card (Calendar: connect the mock calendar) → `POST /api/crews/:id/setup` | — (slip ① is signed by the node) |
| 6. Done | Show agent ID, verified name, attestation expiry | — |

The wallet is also used later for:
- **Rotate keys:** `POST /api/keys/rotate/prepare`, then `sendTransaction(… setAgentURI …)`, then `POST /api/keys/rotate/done`.
- **Revoke my N1:** `POST /api/agent/revoke/prepare`, then `sendTransaction(… revoke …)`.

## 4. Config
`N1_API_URL`. The API token is read from `/data/ui-token` at container start and injected as a header.

## 5. Security notes
- **TB1:** served on localhost only. The UI can grant only within `delegationPolicy`. Confirmations and key changes both need the Browser Wallet (account X).
- Peer-supplied strings (`topic`, `title`, `note`) are rendered as plain text, never as HTML or markdown.

## 6. Tests
- A Playwright smoke test of the wizard with a mocked wallet provider.
- Directory and trust-log rendering from fixture SSE events.
- Real wallet steps are covered by the M0 spikes and the demo rehearsal (M8).
