# Human\-in\-the\-Loop Consent & Identity Assurance for Agentic Workflows Through Concordium

# **Vision:**

- Enable autonomous Agents to complete real-world workflows without taking custody of a user’s identity or being granted unrestricted authority to make sensitive decisions on their behalf.
- Agents should be able to work independently and, when a service requires **human consent and/or identity assurance**, pause and bring the human into the loop through Concordium ID to explicitly authorize the action and provide any required privacy-preserving identity proofs.
- Concordium becomes the **human trust layer for Agentic workflows** — providing a trusted boundary where **consent and identity assurance remain between the human and the service requesting them**, while the Agent can safely resume its task once the requirement has been satisfied.
- The underlying **consent + privacy-preserving proof capability** should be reusable beyond Agentic workflows, providing a foundation for other Concordium services such as **KYC-as-a-Service and enhanced identity verification**, wherever a service needs verifiable human authorization together with identity assurance.

 

# **Problem Statement :**

- Autonomous Agents can perform increasingly complex tasks on behalf of users, but many real-world workflows eventually encounter **human-gated steps**, such as explicit user consent or identity requirements including age verification, residency, eligibility, or KYC. An Agent cannot safely satisfy these requirements itself without being given access to the user's identity credentials or authority to make sensitive decisions on the user's behalf, which would significantly expand the security and privacy boundary.
- The challenge is therefore to allow the **Agent to pause when human authorization or identity assurance is required**, bring the correct human into the existing IdApp verification flow, and resume automatically once the Service Provider confirms that the requirement has been satisfied — **without exposing the user's identity credentials or sensitive identity proofs to the Agent**.

 

# **Business Case:**

- This capability allows businesses to extend **autonomous Agent experiences into identity-sensitive and regulated workflows**. Agents can perform the majority of a task independently, while Concordium brings the human into the loop when **explicit consent and/or identity assurance** is required. This enables Agent-driven services to support KYC, age verification, residency, eligibility and other human-gated requirements without giving Agents custody of sensitive identity credentials.
- For Concordium, this extends the existing **Verify & Access infrastructure into the Agentic ecosystem**, positioning Concordium as a **trust, consent and identity assurance layer for autonomous Agent workflows**. A single verification interaction can potentially combine **user consent **with** privacy-preserving identity proofs**, rather than requiring businesses to integrate separate consent and identity-verification mechanisms. This combined consent + proof concept was specifically highlighted in the meeting.
- Concordium does **not need to replace Agent platforms, marketplaces or their existing communication channels**. Those platforms continue to manage the Agent experience and communication with their users; Concordium provides the trusted human-verification step when the autonomous workflow reaches an action requiring consent or verified human attributes.


|  |  |
| --- | --- |
| **Requirement from Service Provider**  | **Verify & Access request**   |
| Consent + identity   | Consent string + identity attributes   |
| Consent only   | Consent string, **no identity attributes**   |

# Flow 1 — Human-in-the-Loop: Consent + Identity Verification

The Agent operates autonomously until the Merchant encounters an action requiring **both explicit human consent and identity assurance**. The Agent enters the `AGENT_WAITING_FOR_HUMAN_AUTHORIZATION` state while the Merchant brings the user into the Verify & Access flow. 

The Verify & Access request contains the **consent context together with the required identity attributes/proofs**. The user reviews and approves the request in IdApp, which generates the required proof. Once the Merchant successfully verifies the response, the Agent is released from the waiting state and automatically resumes its task.

 

**Example:** Authorize the purchase of a flight **and** prove that the user is over 18. 

# Flow 2 — Human-in-the-Loop: Consent-Only Authorization

The Agent operates autonomously until it encounters an action that requires **explicit human authorization but no additional identity attributes**. The Agent enters the same `AGENT_WAITING_FOR_HUMAN_AUTHORIZATION` state while the Merchant initiates the Verify & Access flow.

In this case, the request contains the **consent context only**, with no identity-proof requirement. The user reviews the exact action being requested and explicitly approves it in IdApp. IdApp generates evidence of that authorization, which the Merchant verifies before allowing the Agent to resume its task.

 

**Example:** Authorize the Agent to purchase a specific flight for a specified amount, where no age, residency, KYC, or other identity attributes are required.

To support this flow, the solution needs to define two key aspects:

1. **Consent Context Structure and IdApp Rendering** — how the Merchant communicates what the user is being asked to authorize, and how IdApp presents that context to the user.
2. **Consent Proof Generation in IdApp** — how IdApp generates cryptographic evidence that the user approved the specific consent context using a valid Concordium identity, without revealing unnecessary identity attributes.

## Consent Context Structure and IdApp Rendering

For a consent-only authorization request, the Merchant must clearly describe what the user is being asked to authorize.

The solution reuses the existing Verify & Access `ContextString` through `GivenContextContextString`. The string contains a prefix identifying the context as consent, a supported display format, and the message to show:

```elixir
consent:<format>:<message>
```

- `consent:` identifies a consent context.
- `<format>` specifies how IDApp renders the message: `markdown` or `text`.
- `<message>` contains the complete merchant-authored consent message.

For example:

```swift
{
  "label": "ContextString",
  "context": "consent:markdown:\n## Authorize flight purchase\n\n- **Agent:** Travel Agent\n- **Flight:** SK123\n- **Route:** Copenhagen to London\n- **Maximum price:** EUR 450\n\nI authorize Travel Agent to purchase flight SK123 from Copenhagen to London for up to EUR 450."
}
```

The equivalent plain-text context is:

```css
{
  "label": "ContextString",
  "context": "consent:text:\nAuthorize Travel Agent to purchase flight SK123 from Copenhagen to London for up to EUR 450."
}
```

 

### IDApp Context Processing

IDApp checks the prefix at the beginning of the context string, allowing leading whitespace or line breaks.

1. If the prefix is `consent:markdown:`, IDApp treats the context as consent and renders the remaining message using the supported Markdown features.
2. If the prefix is `consent:text:`, IDApp treats the context as consent and displays the remaining message literally, preserving spacing and line breaks.
3. If the string does not begin with `consent:`, IDApp handles it as an ordinary Verify & Access context.
4. If the string begins with `consent:` but specifies an unsupported format or an empty message, IDApp rejects it as an invalid consent context.

The prefix is not displayed to the user. Merchant-provided content is rendered within IDApp’s consent screen, with approval controls managed by IDApp. 

Sample code snippet to parse the prefix, then render the remaining content according to its format. Preserve the original string for consent evidence.

#### Parsing the Consent Context

IDApp identifies consent requests using the prefix `consent:<format>:`.

```sql
FUNCTION parseContext(context):

    Preserve context as originalContext
    Ignore leading whitespace for prefix detection

    IF context does not start with "consent:":
        RETURN:
            type = "ordinary"
            originalContext = originalContext

    Read the format between "consent:" and the next colon

    IF the format is not "markdown" or "text",
       OR the terminating colon is missing:
        RETURN error "UNSUPPORTED_CONSENT_FORMAT"

    Extract the content after "consent:<format>:"
    Remove one optional initial line break
    Preserve all remaining content and spacing

    IF the content is empty or contains only whitespace:
        RETURN error "EMPTY_CONSENT_MESSAGE"

    RETURN:
        type = "consent"
        format = detected format
        content = extracted content
        originalContext = originalContext
```

Only contexts with a supported format and a non-empty message are treated as valid consent contexts. The original string remains unchanged for consent evidence.

#### Rendering the Consent Context

After parsing the context string, IDApp renders its content according to the specified format:

```sql
parsed = parseContext(request.context)

IF parsing fails:
    Show an invalid consent context error
    Do not allow consent approval

ELSE IF parsed.type is "ordinary":
    Render using the existing Verify & Access context display

ELSE IF parsed.format is "markdown":
    Convert the content into HTML using the supported Markdown subset
    Disable raw HTML, external images and clickable links
    Sanitize the generated HTML
    Render it within the consent message area

ELSE IF parsed.format is "text":
    Render the content as literal text
    Preserve line breaks and spacing
    Do not interpret Markdown or HTML
```

IDApp controls the styling and approval buttons. The Merchant supplies the message content.

> **Note:** The original `contextString` must remain unchanged for consent evidence; parsing and rendering affect presentation only.

#### Proposed Consent Screens

### Examples

#### Example 1 — Energy Contract

The Merchant sends:

```markdown
consent:markdown:
## You are being asked to accept an energy contract

| Detail | Value |
| --- | --- |
| Provider | Example Energy |
| Plan | Fixed 12 Month |
| Estimated monthly cost | 82.00 EUR |
| Contract period | 12 months |

I authorize acceptance of the **Example Energy Fixed 12 Month** contract.
```


#### Example 2 — Cancel Hotel Reservation

An AI agent could send hotel booking cancellation context request to user/idApp

```markdown
consent:markdown:
## Authorize cancellation of a hotel reservation

| Detail | Value |
| --- | --- |
| Hotel | Copenhagen Marriott |
| Check-in | 15 Oct 2026 |
| Cancellation fee | EUR 75 |
| Reservation ID | HTL-78231 |

I authorize cancellation of reservation **HTL-78231** and accept the **EUR 75 cancellation fee**.
```

#### Example 3 — Authorize Data Access 

```markdown
consent:markdown:
## Authorize data access

- **Requester:** Example Research
- **Data:** Activity summaries
- **Purpose:** Health research
- **Access period:** 3 months

I authorize Example Research to access my activity summaries for the stated research purpose for **3 months**.
```

 

---

## Consent Proof Generation & Verification

For **Flow 2 — Human-in-the-Loop: Consent-Only Authorization**, IDApp generates an identity-based presentation after the user explicitly approves the displayed consent message.

The flow reuses the existing Verify & Access protocol. It does not require additional attribute statements solely to obtain a consent presentation.

The Merchant must establish:

1. **Identity ownership** — the presenter controls the Concordium identity credential used to generate the proof.
2. **Identity validity** — the identity credential has not expired when the presentation is verified.
3. **Request binding** — the verified presentation corresponds to the original request, including its nonce and exact consent `ContextString`.

### Merchant generates Consent Presentation Request Without Attribute Statements

The request includes an identity claim with `source: ["identityCredential"]` and an empty `statements` array:

```json
{
  "subjectClaims": [
    {
      "type": "identity",
      "source": ["identityCredential"],
      "statements": [],
      "issuers": [
        "did:ccd:testnet:idp:0",
        "did:ccd:testnet:idp:1",
        "did:ccd:testnet:idp:3"
      ]
    }
  ]
}
```

An empty `statements` array means no additional identity-attribute statements are requested; the identity claim itself remains part of the request.

The issuer list must correspond to the request’s network and the identity providers accepted by the Merchant

Sample presentation request generated by the Merchant:

```plain
{
    type: 'ConcordiumVerificationRequestV1',
    context: {
      type: 'ConcordiumUnfilledContextInformationV1',
      given: [
        {
          label: 'Nonce',
          context: '2398ff5f354946185992e2b1c85b75d94a851a62f4f8783006054c5e38ea5ad5'
        },
        {
          label: 'ConnectionID',
          context: 'a6814940-91b8-45c8-b1b6-f4cafa478254'
        },
        {
          label: 'ContextString',
          context: '\n            consent:markdown:\n            ## You are being asked to accept energy contract\n            Provider                Example Energy\n            Plan                    Fixed 12 Month\n            Estimated monthly cost  82.00 EUR\n            Contract period         12 months\n            I authorize acceptance of the Example Energy Fixed 12 Month contract.\n          '
        },
        {
          label: 'ResourceID',
          context: '/api/v1/verify/id-proof'
        }
      ],
      requested: [
        'BlockHash'
      ]
    },
    subjectClaims: [
      {
        type: 'identity',
        source: [
          'identityCredential'
        ],
        statements: [],
        issuers: [
          'did:ccd:testnet:idp:0',
          'did:ccd:testnet:idp:1',
          'did:ccd:testnet:idp:3'
        ]
      }
    ],
    transactionRef: '8bba3cf4f686e6d83b1c5df2daf12a127ca4e6445476137c6f7a5cda1120a15e'
  }
}
```

### Merchant sends the presentation request to IdApp

The merchant uses the same WalletConnect flow as Verify & Access. It establishes a secure session with **Concordium IDApp**, requesting support for the `request_verifiable_presentation_v1` method on the relevant Concordium network.

Once the session is approved, the merchant sends the consent presentation request using this method. IDApp displays the consent message and, after explicit user approval, generates and returns the presentation through the same session.

### IDApp Generates the Presentation

After explicit user approval, IDApp generates the presentation using the same SDK method as Verify & Access:

```cs
const presentation = await VerifiablePresentationV1.createFromAnchor(
  client, // ConcordiumGRPCWebClient
  vpr,  // Consent presentation request of type VerificationRequestV1.Type
  statements, // Consent empty statement of type SubjectClaims. i.e The claims to prove for the corresponding subject
  inputs,  // IdentityCommitmentInput
  contextValues // The credential inputs for generating proofs corresponding to the `claims`
);
```

IDApp returns the generated presentation to the merchant through the WalletConnect session for verification. Sample presentation:

```plain
{
  "type": [
    "VerifiablePresentation",
    "ConcordiumVerifiablePresentationV1"
  ],
  "presentationContext": {
    "type": "ConcordiumContextInformationV1",
    "given": [
      {
        "label": "Nonce",
        "context": "<REQUEST_NONCE>"
      },
      {
        "label": "ConnectionID",
        "context": "<CONNECTION_ID>"
      },
      {
        "label": "ContextString",
        "context": "\n            consent:markdown:\n            ## You are being asked to accept energy contract\n            Provider                Example Energy\n            Plan                    Fixed 12 Month\n            Estimated monthly cost  82.00 EUR\n            Contract period         12 months\n            I authorize acceptance of the Example Energy Fixed 12 Month contract.\n          "
      },
      {
        "label": "ResourceID",
        "context": "/api/v1/verify/id-proof"
      }
    ],
    "requested": [
      {
        "label": "BlockHash",
        "context": "<ANCHOR_BLOCK_HASH>"
      }
    ]
  },
  "verifiableCredential": [
    {
      "type": [
        "VerifiableCredential",
        "ConcordiumVerifiableCredentialV1",
        "ConcordiumIdBasedCredential"
      ],
      "credentialSubject": {
        "id": "did:ccd:testnet:encidcred:<ENCRYPTED_IDENTITY_CREDENTIAL>",
        "statement": []
      },
      "validFrom": "2026-09-01T00:00:00Z",
      "validUntil": "2027-09-30T23:59:59Z",
      "issuer": "did:ccd:testnet:idp:0",
      "proof": {
        "created": "2026-10-05T05:57:20.299Z",
        "proofValue": "<CREDENTIAL_PROOF_VALUE>",
        "type": "ConcordiumZKProofV4"
      }
    }
  ],
  "proof": {
    "created": "2026-10-05T05:57:20.299Z",
    "proofValue": "",
    "type": "ConcordiumWeakLinkingProofV1"
  }
}
```

The presentation contains:

- **Request context:** The nonce, connection identifier, original consent `ContextString`, resource identifier and requested block hash.
- **Identity-based credential:** The issuer, encrypted identity credential identifier and an empty `statement` array.
- **Validity information:** `validFrom` and `validUntil`, included directly in the credential.
- **Cryptographic proof:** A `ConcordiumZKProofV4` credential proof, despite no additional attribute statements being requested.

The request uses `statements: []`, while the generated credential uses `credentialSubject.statement: []`.

### Identity Validity Verification By Merchant

The expiry field in the generated presentation is `validUntil`. The Merchat must cryptographically verify the presentation and ensure that the verification time falls within the credential’s validity period, using the applicable validity-boundary rules:

```plain
validFrom ≤ verificationTime ≤ validUntil
```

> Note: A separate attribute range statement is unnecessary for this flow because the presentation already contains the validity dates. These dates are visible to the Merchant; the design therefore does not claim to conceal the expiry value.

Successful generation confirms that IDApp can produce a presentation with an empty statement array. Acceptance still requires successful proof verification, validity checks and matching the verified context to the original consent request.
