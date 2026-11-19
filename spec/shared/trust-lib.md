# `trust-lib` (shared package)
`packages/trust-lib`. All cryptography and trust checks live here, and `n1-agent`, `calendar-gateway`, `relay-service` and `tools/evil-peer` share them. No service re-implements a check.

## 1. Dependencies
| Package | Used for |
|---|---|
| `canonicalize` | JCS (RFC 8785) |
| `@noble/ed25519`, `@noble/curves` (x25519), `@noble/hashes` | Keys, signatures, SHA-256 |
| `bs58` / `multiformats` | Multibase and multicodec key encoding |
| `@concordium/web-sdk` | gRPC client: `agentOf` / `getMetadata` (contract invoke), `getAccountInfo`, `getBlockItemStatus`, `verifyMessageSignature` |
| `ajv` | JSON schemas for cards, attestations, slips, invocations and message bodies |

## 2. Modules and API
```ts
// canonical.ts
canonicalBytes(obj: unknown): Uint8Array            // JCS → UTF-8
sha256Hex(bytes: Uint8Array): string

// keys.ts
generateEd25519(): KeyPair; generateX25519(): KeyPair
toMultibase(pub: Uint8Array, codec: 'ed25519-pub' | 'x25519-pub'): string
fromMultibase(mb: string): { codec: string; key: Uint8Array }
openKeystore(path: string, passphrase: string): Keystore      // scrypt + AES-256-GCM file
//   keystore.get('invocation' | 'delegation' | 'encryption'): KeyPair

// proofs.ts  (eddsa-jcs-2022)
signDataIntegrity<T>(doc: T, opts: ProofOptions, key: KeyPair): T & { proof: Proof }
verifyDataIntegrity(doc: Signed, publicKey: Uint8Array): boolean

// chain.ts  (Concordium)
createChainClient(cfg: { grpcEndpoint; cis8004: ContractAddress }): ChainClient
//   agentOf(tokenId), getMetadata(tokenId, key), accountByCredRegId(credRegId),
//   getRegisterData(txHash): { sender, data } | null, accountKeys(address)
verifyAccountMessage(account, message: string, signature, chain): Promise<boolean>   // wallet-link challenge only

// agent.ts
verifyAgent(agentId, opts: { expectedOwner?; chain; cardFetcher; cache }): Promise<AgentResult>   // agent-card.md §5
verifyAttestation(att, agent: AgentResult, opts: { attesterKey; attesterAccount; chain }): Promise<Result>  // attestation.md §5

// envelope.ts
signEnvelope(env: UnsignedEnvelope, invocationKey): Envelope
verifyEnvelope(env, ctx: EnvelopeCtx): Promise<Result>       // messages.md §2 steps 1–8
handshake: { hello(), welcome(), confirm(), sessionId(nA, nB) }

// zcap/
rootIdFor(target: string): string
draftSlip(opts: { target; controller; actions; caveats; expiresDays }): UnsignedSlip
checkPolicy(slip, policy: DelegationPolicy): Result          // capabilities.md §6
signSlip(slip, delegationKey, signerAgentId): Slip
invoke(slip, action, params, invocationKey, invokerAgentId): Invocation
verifyChain(slip, ctx: ZcapCtx): Promise<Result>              // capabilities.md §7
verifyInvocation(inv, ctx: ZcapCtx): Promise<InvocationResult>
caveatRegistry: Record<CaveatType, { schema; check(params, value, ctx); narrower(child, parent) }>
actionRegistry: Record<ActionName, { crew; params: JSONSchema; delegable: boolean; requiresConfirmation?: boolean }>
registerCrew(def: { id; actions; caveats }): void            // called by n1-agent / gateways at start (crew-modules.md)

// consent.ts
commitmentHash(crew: string, commitment: object): string     // consent.md §2
consentContextString(m, names): string                        // consent.md §3
verifyConsentRecord(rec, expected: { commitmentHash; serviceAccount }, chain): Promise<Result>   // consent.md §5

// reasons.ts
type Reason = 'AGENT_NOT_FOUND' | 'AGENT_REVOKED' | /* … reason-codes.md */ 'OK_INVOCATION'
type Result = { ok: true } | { ok: false; reason: Reason; detail?: string }
```

## 3. Context interfaces
```ts
interface ZcapCtx {
  now(): Date
  chain: ChainClient
  expectedOwner(target: string): string        // gateway / approver: own X; requester: X parsed from target
  isRevoked(capId: string): boolean
  seenNonce(nonce: string): boolean             // returns true if already seen, records otherwise
  rateCount(capId: string, day: string): number
  confirmation?: Confirmation                   // for RequiresOwnerConfirmation
}
interface EnvelopeCtx { now(): Date; selfAgentId: string; sessions: SessionStore; nonces: NonceStore; agentCache: AgentCache }
```
`Result` never throws for trust failures. It returns a reason, and callers log it. Exceptions are only for infrastructure errors (chain down), which surface as `502`.

## 4. Tests
- Golden vectors (JSON fixtures) for: card hash, attestation proof, envelope signature, slip ① / ②, invocation, commitment hash.
- One negative vector per reason code in [reason-codes.md](reason-codes.md) that `trust-lib` can produce.
- Run with `vitest`. The chain is mocked by a `FakeChain` fixture; integration tests use testnet ([testing.md](../testing.md)).
