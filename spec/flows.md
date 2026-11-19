# End-to-end flows
Sequences across services. Each step references the spec that defines it.

## F1. Onboarding (once per user)
```mermaid
sequenceDiagram
    actor O as Owner
    participant UI as ui
    participant BW as Browser Wallet (X)
    participant N1 as n1-agent
    participant RS as relay-service
    participant RV as relay verifier
    participant CH as Concordium

    O->>UI: open localhost:3000
    UI->>BW: connect → account X
    UI->>N1: link/challenge
    UI->>BW: signMessage(X, challenge)
    UI->>N1: link {X, sig} (verified against X's keys on chain)
    UI->>N1: setup/crews {calendar} (one-time choice)
    UI->>N1: agent/prepare
    N1->>N1: generate invocation, delegation, encryption keys · build card (crews + per-crew policy)
    N1->>RS: PUT /cards → uri, hash
    N1-->>UI: registerParams
    UI->>BW: sendTransaction register(uri, hash, app=number-one/1)
    BW->>CH: register → Registered(token)
    UI->>N1: agent/registered {txHash} → agentId
    UI->>N1: attestation/start
    N1->>RS: POST /attestations/requests {agentId, X}
    RS->>RV: create-verification-request (accountCredential: names + dob range)
    RV->>CH: anchor VRA
    RS-->>UI: VPR (via n1-agent)
    UI->>BW: VP request → VP
    UI->>N1: attestation/complete {vp}
    N1->>RS: POST …/presentation {vp}
    RS->>RV: verify → VAA anchored
    RS->>CH: credRegId → X? agentOf → owner X?
    RS-->>N1: signed attestation (directory updated)
    N1->>N1: verifyAttestation (pinned key) ✓
    UI->>N1: idapp/pair → wcUri (QR) · owner scans with ID App
    UI->>N1: crews/calendar/setup → mock calendar connected · slip ① signed by the delegation key
```
Specs: [ui.md](node/ui.md) §3 · [agent-card.md](shared/agent-card.md) · [attestation.md](shared/attestation.md) · [relay-service.md](relay/relay-service.md) §3.3

## F2. Connect + reactive grant (once per peer)
```mermaid
sequenceDiagram
    actor K as Kash
    participant KN as Kash n1-agent (#17)
    participant R as relay
    participant VN as Vishwas n1-agent (#42)
    actor V as Vishwas

    K->>KN: directory → Vishwas ✓ → "find a time"
    KN->>R: GET /directory · verify #42 (chain + attestation)
    KN->>VN: session.hello / welcome / confirm (both verify each other)
    KN->>VN: contact.request {topic, window}
    VN->>VN: no slip for #17 → inbox (OK_FIRST_CONTACT)
    VN-->>KN: contact.request.ack {pending_permission}
    Note over V: later, at his desk
    V->>VN: Inbox → Allow (weekend free/busy + propose, 90 days)
    VN->>VN: build slip ②, checkPolicy, sign with delegation key
    VN-->>KN: capability.grant {slip ②}
    KN->>KN: verifyChain(②) → store received → auto-retry (F3)
```
Specs: [messages.md](shared/messages.md) · [capabilities.md](shared/capabilities.md) §8 · [n1-agent.md](node/n1-agent.md) §5.2

## F3. Task + commitment while the approver is away (calendar crew)
```mermaid
sequenceDiagram
    actor K as Kash
    participant KN as Kash n1-agent
    participant KG as Kash gateway
    participant VN as Vishwas n1-agent
    participant VG as Vishwas gateway
    participant VV as Vishwas verifier
    participant IDA as Vishwas ID App (phone)

    KN->>VN: calendar.availability.request + invocation(②, calendar.freebusy.read)
    VN->>VN: verifyEnvelope + verifyInvocation (A0)
    VN->>VG: invoke(①, calendar.freebusy.read)
    VG-->>VN: busy blocks
    VN-->>KN: calendar.availability.response {options} (friend tier, Sat–Sun, 30 min)
    K->>KN: pick Sat 20:00 (owner choice)
    KN->>VN: commitment.propose {crew: calendar, commitment} + invocation(②, calendar.meeting.propose)
    VN->>VN: validate + conflicts + recheck (calendar module) · hold (tentative) · commitmentHash
    VN-->>KN: commitment.tentative {commitmentHash}
    VN->>VV: create-verification-request (consent:markdown + names)
    VN->>IDA: request_verifiable_presentation_v1 (paired session)
    IDA-->>VN: VP (Vishwas approves)
    VN->>VV: verify → VAA anchored
    VN->>VN: checks (session, name match, recheck)
    VN->>VG: invoke(①, calendar.event.insert) + {consent record}
    VG-->>VN: eventId
    VN-->>KN: commitment.confirmed {commitmentHash, vaaTx}
    KN->>KG: invoke(Kash ①, calendar.event.insert) + {approverConfirmed envelope}
    KG-->>KN: eventId
```
The negotiation messages (`calendar.availability.*`) and the hooks (validate, recheck, execute) come from the calendar crew; everything else is the core commitment protocol, the same for every crew.

Specs: [consent.md](shared/consent.md) · [crews/calendar.md](crews/calendar.md) · [calendar-gateway.md](node/calendar-gateway.md) §3 · [n1-agent.md](node/n1-agent.md) §5.3

## F4. Revoke a slip
1. Vishwas → Permissions → **Revoke** on Kash's slip ②.
2. `n1-agent` adds the slip ID to its revocation list. From now on, any invocation fails with `CAP_REVOKED` (NT10).
3. `n1-agent` sends `capability.revoke` to Kash's N1, which marks the slip revoked and stops using it.

## F5. Rotate the delegation key / revoke the N1
- **Rotate:**
  1. Settings → **Rotate keys** → `keys/rotate/prepare`: new card, uploaded to the relay.
  2. Wallet `setAgentURI(token, uri, hash)`, then `keys/rotate/done`.
  3. Every slip signed by the old key now fails with `CAP_SIG_INVALID`. Re-allow peers as needed.
- **Revoke the N1:** Settings → **Revoke my N1** → wallet `revoke(token)`. Peers get `AGENT_REVOKED` on their next check (NT2), and the relay removes the agent from the directory on its next refresh.

## F6. Audit anchoring
Settings → **Anchor audit now** → `n1-agent` sends `RegisterData("N1-audit:<agentId>:<seq>:<hash>")` from the node service account. The UI shows the transaction, and anyone can check the hash chain against it (S8).
