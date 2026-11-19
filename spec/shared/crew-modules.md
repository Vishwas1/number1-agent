# Crew modules
How an N1 gets capabilities such as Calendar, without the core knowing about any of them. Implements D12 of [technical-solution.md](../../technical-solution.md).

## 1. Concept
A **crew** is a pluggable skill of the N1. Each crew is:
| Part | Where | Role |
|---|---|---|
| **Module** | A package loaded by `n1-agent` (`apps/n1-agent/crews/<id>/`) | Actions, caveats, message types, intents, negotiation logic, consent text, UI hints |
| **Gateway** | Its own container in the node bundle (e.g. `calendar-gateway`) | Holds the resource's credentials and data; acts only on valid invocations (`POST /v1/invoke`) |
| **Card entry** | `crews` + `delegationPolicy.<id>` in the Agent Card | Tells peers the crew exists, and caps what the N1 may grant for it. X signs it. |

The core (`trust-lib` plus the `n1-agent` core) only knows the interface in §3. **In the POC there is one crew, `calendar`, and every user enables it at setup.**

## 2. Lifecycle
| Event | What happens | Wallet signature? |
|---|---|---|
| **Choose crews** (one-time setup, before registration) | The owner picks crews from the catalog (POC: Calendar, pre-selected and required). Their policies go into the card. | No: they're included in the `register` transaction |
| **Crew setup** | The module's `setup()` runs: connect the adapter (mock calendar), then issue the own-N1 slip for `<crewId>:…:<X>` | No |
| **Add a crew later** | New card with the crew and its policy → `setAgentURI` → `setup()` | Yes, once |
| **Remove a crew** | Confirmation listing affected peers → new card without it → `setAgentURI` → `teardown()` (revoke own and issued slips, stop the adapter). Peers' slips for it now fail with `CAP_POLICY_EXCEEDED`. | Yes, once |

Adding a crew widens what the N1 may grant, so the owner signs it. Removing one revokes every peer's access to that resource automatically.

## 3. Module interface (`n1-agent`)
```ts
interface CrewModule {
  id: string                         // 'calendar' — also the resource type and the action/message namespace
  version: string                    // '1' — card lists 'calendar/1'
  title: string; icon: string        // UI card
  actions: Record<string, {          // full names, e.g. 'calendar.freebusy.read'
    params: JSONSchema
    delegable: boolean               // may appear in slip ② (still capped by the card policy)
    requiresConfirmation?: boolean   // RequiresOwnerConfirmation in the own-N1 slip
  }>
  caveats: CaveatDefinition[]        // registered into trust-lib's caveat registry at start
  defaultPolicy: CrewPolicy          // proposed card policy at setup (owner can narrow it)
  defaultGrant: GrantTemplate        // scope offered in the "Allow" dialog
  messages: Record<string, {         // full names, e.g. 'calendar.meeting.propose'
    schema: JSONSchema
    requires?: string                // action the sender must invoke, e.g. 'calendar.meeting.propose'
    handle(msg: VerifiedEnvelope, ctx: CrewContext): Promise<void>
  }>
  intents: JSONSchema                // what the LLM may route to this crew (e.g. find_time)
  checkPolicy?(slip, policy): Result // crew-specific policy fields (calendar: allowedTiers)
  start(task, ctx): Promise<void>    // requester: run the negotiation phase with crew messages, then ctx.reportOptions(options)
  optionRenderer: string             // UI renderer for options (calendar: 'slot-list')
  commitment: {                      // generic commitment protocol (consent.md §3)
    schema: JSONSchema               // crew part of the commitment, e.g. { start, end, title }
    proposeAction: string            // action invoked by commitment.propose, e.g. 'calendar.meeting.propose'
    fromOption(option, task): unknown          // owner's pick → commitment
    validate(c, invocation): Result            // fits the slip's caveats
    conflicts?(a, b): boolean                  // two holds can't coexist
    recheck(c, ctx): Promise<boolean>          // still possible right before execution
    text(c, names): string                     // consent markdown (the core adds Ref)
    execute(c, confirmation, ctx): Promise<void>  // call the gateway's confirmation-gated action (both sides)
  }
  gateway: { url: string }           // e.g. http://calendar-gateway:7200
  setup(ctx: CrewContext): Promise<void>
  teardown(ctx: CrewContext): Promise<void>
  views?: {                          // e.g. 'my-calendar'
    id: string; title: string
    load(query, ctx): Promise<unknown>
    actions?: Record<string, { body: JSONSchema; run(body, ctx): Promise<unknown> }>  // owner-only, from UI clicks (e.g. 'add-busy')
  }[]
}
```
`CrewContext` gives the module only typed helpers:
- `invokeGateway(action, params, confirmation?)`
- `send(to, type, body)` (crew message types only)
- `reportOptions(taskId, options)`
- `issueOwnSlip()`
- `revokeSlips()`
- `emit(trustEvent)`
- `store` (a module-scoped table prefix)

**Modules never get keys.** Signing, verification and slip issuing stay in the core.

## 4. Rules the core enforces
| Rule | Where |
|---|---|
| Resource ID = `<crewId>:ccd:testnet:<X>`; the crew must be installed in X's card | [capabilities.md](capabilities.md) §3 |
| Action and message names must start with `<crewId>.` | Module loader (rejected at start otherwise) |
| Allowed message types = core types (`session.*`, `contact.request(.ack)`, `capability.*`, `commitment.*`, `error`) + types of crews installed in **both** cards | [messages.md](messages.md) §4 |
| Slip ② actions ⊆ the issuer card's `delegationPolicy.<crewId>.maxActions`, and only `delegable` actions | [capabilities.md](capabilities.md) §6 |
| Commitments use the core protocol; the hash is `sha256(JCS({ crew, commitmentId, requester, approver, commitment }))` | [consent.md](consent.md) §2 |
| The LLM routes only to intents of installed crews; output is validated against the crew's `intents` schema | [n1-agent.md](../node/n1-agent.md) §5.4 |

## 4a. Gateway contract (every crew)
| Endpoint | Behaviour |
|---|---|
| `POST /v1/invoke { invocation, confirmation? }` | `trust-lib` `verifyInvocation` with `expectedOwner` = own X and `controller` = own N1; for actions with `requiresConfirmation`, check the confirmation object ([consent.md](consent.md) §6) via `GET <n1-agent>:7101/internal/commitments/:id`; then call the adapter |
| `GET /v1/log?since=<seq>` | Decision log, pulled by `n1-agent` into the audit log |
| `GET /health` | `{ ok, adapter }` |

Rules: no host port (Docker network only); holds the resource's credentials; no LLM; returns only what the action allows.

## 5. Catalog
- **POC:** the modules are bundled in the `n1-agent` image, and their gateways are listed in `docker-compose.yml`. `GET /api/crews` returns `{ available, installed }`.
- **Later:** a signed catalog from the relay (untrusted for authority, like the directory). Module code still runs locally.

## 6. Adding a crew (definition of done)
A new crew needs **only**:
1. `spec/crews/<id>.md`, following [crews/calendar.md](../crews/calendar.md): actions, caveats, policy, intents, negotiation messages, commitment hooks, gateway, UI
2. a module in `apps/n1-agent/crews/<id>/` implementing §3
3. a gateway container implementing §4a, added to `docker-compose.yml`

**No changes** to `trust-lib`, the `n1-agent` core, the relay, the message core, consent or the UI shell. If a crew needs a core change, that's a gap in this interface and must be fixed here first.

**Crews:** [calendar](../crews/calendar.md) (POC).

## 7. Stretch: a second crew
A minimal `notes` crew (`notes.read`, `notes.append`), added and removed live in the demo, proves the abstraction. It's optional and only built if week 4 allows.

## 8. Tests
- The loader rejects a module whose actions aren't namespaced.
- A message type of a crew the peer doesn't have → `TYPE_NOT_ALLOWED`.
- After removing a crew, a peer's slip for it fails with `CAP_POLICY_EXCEEDED`.
- A slip ② containing a non-delegable action → `CAP_POLICY_EXCEEDED`.
