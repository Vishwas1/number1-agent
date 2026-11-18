# Number One: Trusted AI Agents on Concordium
**Business overview · Phase 1 proof of concept · Nov 2026**

---

## Vision
Everyone gets a personal AI assistant, their **Number One**, with a small team of helper agents.

Number Ones work things out **with each other** for their owners: finding a time to meet, planning a dinner, sorting the details.

**Concordium makes it trustworthy.** You always know whose agent you're talking to, and your agent only does what you've allowed.

## Problem Statement
Arranging even simple things takes a lot of back-and-forth: *"Can't do Friday… what about Sunday… any allergies?"*

AI agents could do this for us. But the moment one person's agent talks to another's, there are four questions nobody can answer today:

1. **Is this really my friend's agent, or an impostor?**
2. **Is there a real, verified person behind it?**
3. **What is it allowed to ask, and what may mine share?**
4. **Did the person actually agree to this?**

Without answers, nobody will let an agent act on their behalf.

## Business Value
| For | Value |
|---|---|
| **Concordium** | A live showcase of Concordium as the **trust layer for AI agents**: identity, agent registry, consent and audit working together in one simple product |
| **Users** | Agents do the legwork, and people stay in control: they choose who their agent talks to, about what, and for how long |
| **Partners & businesses** | One trust pattern that carries over to bigger use cases: dinner planning and allergies, shopping and payments, sharing certificates. Each one builds on the same foundation. |
| **Risk** | Every agent is tied to a verified person, every permission is limited and can be withdrawn, and every important action leaves a tamper-proof record |

## Solution

### User journey: "Let's grab a drink"
Kash wants to meet Vishwas next week, and neither of them wants five rounds of WhatsApp messages.

| Step | What happens | Who's involved |
|---|---|---|
| **1. Get set up** (once) | Each person signs in with Concordium, proves their name once, and gets a Number One with a Scheduler helper, registered under their verified identity | each person |
| **2. Give permission** (once per friend) | Vishwas allows Kash's agent to **see when he's free on weekends** and **suggest meetings**, until February. He approves it in the Concordium ID App. | Vishwas |
| **3. Find people** | Kash opens the app and sees a **directory of Number Ones**: Vishwas ✓, Pete ✓, each showing a verified name | Kash |
| **4. Ask** | Kash picks Vishwas: *"Find a time for a drink next weekend."* **Vishwas is offline**, and that's fine. | Kash |
| **5. Agents agree** | The two agents talk on their own. Vishwas's agent checks who's asking and what's allowed, then shares **only free/busy times**. Kash picks Sat 7pm, and the slot is **held as tentative**. Nothing is booked yet. | the agents |
| **6. Confirm** | Kash confirms in his ID App. When **Vishwas comes online**, he sees *"Drink with Kash, Sat 7pm?"* and confirms in his ID App. Only then do both calendars update. | Kash, then Vishwas |

Someone who hasn't given permission, like Pete, still receives Kash's request in their inbox, but their agent shares nothing until they say so.

```mermaid
sequenceDiagram
    actor K as Kash
    participant KN as Kash's Number One
    participant C as Concordium
    participant VN as Vishwas's Number One
    participant VS as Vishwas's Scheduler
    actor V as Vishwas

    Note over K,V: One-time setup
    K->>C: Verify, prove my name, register my Number One
    V->>C: Verify, prove my name, register Number One + Scheduler
    V->>VN: Allow Kash's agent: weekend free/busy + suggest meetings
    V->>C: Approve permission in the Concordium ID App

    Note over K,V: Vishwas is offline
    K->>KN: Directory → Vishwas ✓ → "Find a time for a drink"
    KN->>VN: When is Vishwas free? (with Vishwas's permission slip)
    VN->>C: Is this really Kash's agent? Still active?
    C-->>VN: ✓ Yes
    VN->>VS: Check free slots (this one request only)
    VS-->>VN: Free slots
    VN-->>KN: Fri 7pm · Sat 3pm · Sat 7pm (free/busy only)
    K->>KN: Saturday 7pm
    KN->>VN: Propose Sat 7pm
    VN-->>KN: Held as tentative, waiting for Vishwas
    K->>C: Confirm in my ID App

    Note over K,V: Vishwas comes online
    VN->>V: Drink with Kash, Sat 7pm?
    V->>C: Confirm in my ID App
    VN-->>KN: Confirmed
    Note over KN,VN: Both calendars updated · receipt recorded on Concordium
```

**If an impostor tries:** an agent that isn't registered, has been switched off, or isn't Vishwas's real agent is turned away automatically. Kash is told why.

### How it works
Six building blocks, all built on Concordium:

| # | Building block | In one line |
|---|---|---|
| 1 | **Verified owner** | Every user is identity-verified once through Concordium ID |
| 2 | **Agent badge** | Every agent (the Number One and each helper) is registered on the **Concordium Agent Registry** under its owner, like a car registered to a driver |
| 3 | **Prove who you are** | Each person proves their name once with Concordium ID, revealing **only their name**. Anyone can then check whose agent they're talking to, even while that person is offline. |
| 4 | **Permission slips** | The owner signs exactly what another agent may do: which actions, which days, until when. Permissions can only shrink as they're passed on, and they can be withdrawn at any time. |
| 5 | **Human in the loop** | Agents can negotiate while you're away, but anything that commits you, like a meeting or a new permission, waits for your confirmation in the Concordium ID App |
| 6 | **Tamper-proof record** | Verifications and agent actions leave receipts on Concordium, so anyone can check later what was agreed |

```mermaid
flowchart LR
    P[Person] -->|verified once| ID[Concordium ID]
    ID --> A[Concordium account]
    A -->|registers| N1[Number One]
    A -->|registers| S[Scheduler helper]
    A -->|signs| PS[Permission slips]
    N1 -->|manages| S
    PS -->|limit what| OA[Other people's agents]
```

**What stays private:** calendars, preferences and personal details never go on the blockchain. Only badges, receipts and fingerprints of agreements do.

**Proof of concept scope:** two users and one task (calendar scheduling), end to end on Concordium testnet. It includes the "impostor rejected" demo.
