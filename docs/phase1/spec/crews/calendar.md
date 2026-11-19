# Crew: Calendar
The first crew module, and the only one in the POC. Every user enables it at setup. This file holds **everything calendar-specific**; the core specs never mention calendars. Follows the interface in [crew-modules.md](../shared/crew-modules.md).

## 1. Identity
| Item | Value |
|---|---|
| `id` / card entry | `calendar` / `calendar/1` |
| Resource | `calendar:ccd:testnet:<X>` |
| Gateway | `calendar-gateway` ([node/calendar-gateway.md](../node/calendar-gateway.md)) |
| Module code | `apps/n1-agent/crews/calendar/` |
| UI | Crew card "Calendar" 📅; option renderer `slot-list`; view **My calendar** |

## 2. Actions
| Action | Params | Delegable | Requires confirmation | Used by |
|---|---|---|---|---|
| `calendar.freebusy.read` | `{ from, to }` (window ≤ 14 days) | ✅ | — | peer → approver N1 (slip ②); N1 → Gateway (slip ①) |
| `calendar.meeting.propose` | `{ commitmentId, start, end }` | ✅ | — | peer → approver N1 (slip ②): the **propose action** for `commitment.propose` |
| `calendar.event.insert` | `{ commitmentId, start, end, title, attendees: [agentId] }` | ❌ | ✅ | N1 → Gateway (slip ①): the **execute action** |
| `calendar.event.list` | `{ from, to }` | ❌ | — | N1 → Gateway (slip ①): the My calendar view |
| `calendar.event.create` | `{ start, end, title }` (title ≤ 200 chars, `end > start`, ≤ 24 h) | ❌ | — | N1 → Gateway (slip ①): **owner adds busy time** from My calendar. Owner-only: triggered by a UI click, never by a peer message or the LLM. |

## 3. Caveats (crew-provided)
| Type | Value | Check | A child may only… |
|---|---|---|---|
| `SharingTier` | `{ tier: "friend" }` | Shapes responses: free/busy blocks only, no titles | keep or lower it |
| `PermittedDays` | `{ days: ["mon".."sun"], tz }` | Every day touched by `[from, to)` or `[start, end)` in `tz` is listed | take a subset (same `tz`) |
| `SlotGranularity` | `{ minutes }` | Options rounded to blocks of this size | increase it |

Core caveats (`RateLimit`, `MaxDelegationDepth`, `RequiresOwnerConfirmation`) are in [capabilities.md](../shared/capabilities.md) §6.

## 4. Policy and grant
- **Default policy** (written into the card at setup): `{ maxActions: ["calendar.freebusy.read", "calendar.meeting.propose"], allowedTiers: ["friend"], maxExpiryDays: 90, maxDelegationDepth: 0 }`
- **Default grant** (the "Allow" dialog): free/busy + propose; Sat–Sun in `OWNER_TZ`; 90 days; `SharingTier friend`; `SlotGranularity 30`; `RateLimit 20/day`; `MaxDelegationDepth 0`
- **Own slip ①:** `calendar.freebusy.read`, `calendar.event.insert`, `calendar.event.list`, `calendar.event.create`; `RequiresOwnerConfirmation [calendar.event.insert]`; 1 year. `calendar.event.create` needs no consent because the owner is the one clicking (TB1).

## 5. Intent (LLM routing)
`find_time`: `{ peerAgentId, window: { from, to }, durationMin, title }`. Example: "Find a time for a drink with Vishwas next weekend" → `{ window: Sat–Sun next week, durationMin: 60, title: "Drink" }`.

## 6. Negotiation messages
| Type | Direction | Body | Authorization |
|---|---|---|---|
| `calendar.availability.request` | requester → approver | `{ taskId, window: { from, to }, durationMin, invocation }` | invocation of slip ② with `calendar.freebusy.read` |
| `calendar.availability.response` | approver → requester | `{ taskId, options: [ { optionId, start, end } ], granularityMin }` | reply to a valid request |

**Approver handler for `calendar.availability.request`:**
1. Run `verifyInvocation` through the core.
2. Call the gateway's `calendar.freebusy.read` with slip ①.
3. Compute free slots of `durationMin` inside the window. Subtract busy blocks and active holds (§7), keep only `PermittedDays`, round to `SlotGranularity`, and return at most 5 options.
4. Reply with the options only (tier `friend`, so no titles).

**Requester handler for `calendar.availability.response`:** store the options on the task. The UI renders them with `slot-list` buttons.

## 7. Commitment
| Hook | Calendar implementation |
|---|---|
| `schema` | `{ start, end, title }`. The core adds `commitmentId`, `requester` and `approver` ([consent.md](../shared/consent.md) §2). |
| `fromOption(option, task)` | `{ start: option.start, end: option.end, title: task.args.title }` |
| `proposeAction` | `calendar.meeting.propose` (params `{ commitmentId, start, end }`) |
| `validate(c, invocation)` | `[start, end)` passes the slip's caveats; `end − start` = the requested duration; title ≤ 200 characters |
| `conflicts(a, b)` | The time ranges overlap. A held slot blocks other proposals for it. |
| `recheck(c)` | `calendar.freebusy.read` for `[start, end)` is empty |
| `text(c, names)` | see below |
| `execute(c, confirmation)` | Gateway `calendar.event.insert` `{ commitmentId, start, end, title, attendees: [requester, approver] }` + confirmation. Same call on both sides, each with its own slip ①. |

**Approval text**, shown in the owner's **Pending approvals** screen before the Browser Wallet prompt (the UI adds the short commitment hash):
```
## Confirm meeting

- **With:** {requesterName} (Number One {requesterAgentId})
- **When:** {Sat 28 Nov 2026, 20:00–21:00} ({OWNER_TZ})
- **What:** {title}
```
`requesterName` comes from the requester's **attestation**, never from peer text.

## 8. Gateway
[node/calendar-gateway.md](../node/calendar-gateway.md): mock adapter (SQLite, seeded) in the POC; Google adapter as a stretch goal.

## 9. UI
| Element | Behaviour |
|---|---|
| Crew card | Setup: "Connect calendar" (adapter `mock`) → `POST /api/crews/calendar/setup` |
| Option renderer `slot-list` | One button per option: "Sat 28 Nov, 20:00–21:00". Click → `POST /api/tasks/:id/choose { optionId }` |
| View **My calendar** | Week view from `GET /api/crews/calendar/views/my-calendar?from&to` (`calendar.event.list`). Events from the seed, from the owner ("busy"), and confirmed meetings, each tagged by `source`. |
| View action **Add busy time** | Button on My calendar → dialog (date, start, end, title, default "Busy") → `POST /api/crews/calendar/views/my-calendar/actions/add-busy { start, end, title }` → module calls the gateway `calendar.event.create` with slip ①. The new block immediately affects free/busy answers to peers. |

## 10. Tests
- Slot computation (unit): busy blocks, holds, days, granularity, time-zone boundaries.
- A weekday window → `CAP_CAVEAT_VIOLATED` (NT5a).
- `calendar.event.insert` via slip ② → `CAP_ACTION_NOT_ALLOWED` (NT5b).
- A proposal for a slot that's held or busy → `COMMITMENT_UNAVAILABLE`.
- `freebusy.read` responses contain no titles.
- Owner adds busy time Sat 19:00–21:00 → the next `calendar.availability.request` no longer offers those slots.
- `calendar.event.create` via slip ② or from a peer message → `CAP_ACTION_NOT_ALLOWED`.
