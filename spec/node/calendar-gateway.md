# `calendar-gateway`
The **calendar crew's gateway** ([crew-modules.md](../shared/crew-modules.md) §6), and the only component with calendar access. It acts only on valid invocations of slip ①. Every crew gateway follows the same contract: `POST /v1/invoke { invocation, confirmation? }`, `GET /v1/log`, `GET /health`, with checks done by `trust-lib`. `apps/calendar-gateway`, TypeScript, Fastify, SQLite. Implements D8, TB3 and §5.4 of technical-solution.md.

## 1. Purpose
Makes slips enforceable. Calendars don't understand capabilities, so the credentials sit behind this non-LLM service, which checks every call against the chain back to X.

## 2. Responsibilities
- Verify every request: `verifyInvocation` from `trust-lib`, with `expectedOwner = own X`, so only slips rooted at this owner are accepted.
- Accept only invocations from **this node's own N1** (`controller` = own agent ID).
- Enforce `RequiresOwnerConfirmation` for `calendar.event.insert` ([consent.md](../shared/consent.md) §5).
- Return the minimum: busy blocks for `calendar.freebusy.read`, full events only for `calendar.event.list`, the owner's own view.
- Log every decision. `n1-agent` pulls the log via `GET /v1/log` into the audit log.

## 3. Interfaces (Docker network only, port 7200)
| Method + path | Body | Response |
|---|---|---|
| `POST /v1/invoke` | `{ invocation, confirmation? }` | depends on the action (below) |
| `GET /v1/log?since=<seq>` | — | `[{ seq, ts, invocationId, action, decision, code }]` |
| `GET /health` | — | `{ ok, adapter }` |

| Action | Response |
|---|---|
| `calendar.freebusy.read` | `{ busy: [{ start, end }] }`, merged and sorted, UTC |
| `calendar.event.list` | `{ events: [{ id, start, end, title, source }] }` (`source`: `seed` \| `owner` \| `meeting`) |
| `calendar.event.create` | `{ eventId }`. Stored with `source = owner`. |
| `calendar.event.insert` | `{ eventId }` |

**Checks** for `POST /v1/invoke`:
| # | Check | Reject |
|---|---|---|
| 1 | `verifyInvocation(invocation, ctx)` with `expectedOwner = OWNER_ACCOUNT` (from setup) | its reason code |
| 2 | `capability.controller` = own agent ID | `CAP_NOT_HOLDER` |
| 3 | `calendar.event.insert` (`requiresConfirmation`): a confirmation object is present and valid, per the generic rules in [consent.md](../shared/consent.md) §6 (uses `GET <N1_AGENT_URL>/internal/commitments/:id`) | `CONSENT_REQUIRED` / `CONSENT_INVALID` / `CONSENT_COMMITMENT_MISMATCH` |
| 4 | `calendar.event.insert`: the slot is still free | `COMMITMENT_UNAVAILABLE` |

The gateway reaches `n1-agent` over the Docker network on its internal listener (`:7101`, not published to the host; [n1-agent.md](n1-agent.md) §3.2).

## 4. Adapter interface
```ts
interface CalendarAdapter {
  busy(from: Date, to: Date): Promise<{ start: Date; end: Date }[]>
  list(from: Date, to: Date): Promise<CalendarEvent[]>
  insert(e: { start: Date; end: Date; title: string; attendees: string[]; commitmentId: string }): Promise<string>
  create(e: { start: Date; end: Date; title: string }): Promise<string>     // owner-added busy time
}
```
| Adapter | POC | Notes |
|---|---|---|
| `mock` | ✅ | SQLite `calendar.db`. Table `events(id, start, end, title, source, meeting_id)`. Seeded from `seed.json` at first start. |
| `google` | stretch | Desktop OAuth. Uses `freeBusy.query` and `events.insert`. The token is held by the gateway only. |

**Demo seed for Vishwas:**
```json
[
  { "title": "Football", "start": "2026-11-28T14:00:00Z", "end": "2026-11-28T16:00:00Z" },
  { "title": "Family lunch", "start": "2026-11-29T11:00:00Z", "end": "2026-11-29T13:00:00Z" }
]
```

## 5. Config
`GRPC_ENDPOINT`, `CIS8004_CONTRACT`, `NODE_SERVICE_ACCOUNT`, `N1_AGENT_URL`, `OWNER_TZ`, `CALENDAR_ADAPTER=mock`, `SEED_FILE`. `OWNER_ACCOUNT` and `OWN_AGENT_ID` are read from `n1-agent` (`GET /internal/identity`) at start.

## 6. Security notes
- **TB3:** the only holder of calendar data or credentials. The LLM and peers can't reach it, because there's no host port.
- Even a compromised `n1-agent` can't insert without a consent record that was anchored by this node's service account and matches the slot.

## 7. Tests
- No invocation → `CAP_MISSING` (NT8).
- Slip ② presented at the gateway → `CAP_NOT_HOLDER`.
- Slip for another owner's calendar → `CAP_ROOT_MISMATCH`.
- `calendar.event.insert` without a confirmation → `CONSENT_REQUIRED` (NT9).
- A consent for another slot → `CONSENT_COMMITMENT_MISMATCH`.
- `calendar.freebusy.read` returns no titles.
