# Register an Agent on Concordium Testnet

This guide shows the simplest testnet flow for registering an agent in the CIS-8004 Agent Registry.

The hosted Agent Registry MCP builds sign-ready transaction payloads. It does not hold private keys and it does not sign or submit transactions for you.

## What You Need

- A funded Concordium testnet account.
- An Agent Card JSON hosted at a public HTTPS URL.
- A way to sign and submit Concordium transactions, for example Concordium Wallet tooling, `concordium-client`, or the `ccd-agent-kit` signer and chain-client tools.

Testnet registry:

```text
CIS-8004 Agent Registry: <12802,0>
CIS-8 External Key Registry: <12801,0>
MCP endpoint: https://agent-registry-mcp.concordium.com/mcp
```

## 1. Create an Agent Card

If you already have an Agent Card JSON, host it at a stable URL such as:

```text
https://example.com/.well-known/agent-card.json
```

If you want the MCP to compose a basic card for you, call `build_agent_card`:

```bash
curl -s -X POST https://agent-registry-mcp.concordium.com/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/call",
    "params": {
      "name": "build_agent_card",
      "arguments": {
        "network": "testnet",
        "name": "Demo Agent",
        "description": "A simple testnet agent.",
        "agent_uri": "https://example.com/.well-known/agent-card.json",
        "owner_account": "YOUR_TESTNET_ACCOUNT",
        "skills": [
          {
            "id": "hello",
            "name": "Hello",
            "description": "Returns a friendly test response.",
            "tags": ["demo"]
          }
        ]
      }
    }
  }'
```

The response includes canonical JSON and its SHA-256 hash. Host the JSON bytes exactly as returned, then use the returned hash as `metadata_hash_hex`.

## 2. Build the Register Transaction

Call `build_register` with your testnet account, the card URL, and the card hash.

```bash
curl -s -X POST https://agent-registry-mcp.concordium.com/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/call",
    "params": {
      "name": "build_register",
      "arguments": {
        "network": "testnet",
        "sender": "YOUR_TESTNET_ACCOUNT",
        "agent_uri": "https://example.com/.well-known/agent-card.json",
        "metadata_hash_hex": "SHA256_OF_AGENT_CARD_JSON",
        "initial_metadata": [
          {
            "key": "name",
            "value_hex": "44656d6f204167656e74"
          }
        ]
      }
    }
  }'
```

`value_hex` is bytes encoded as hex. The example above is `Demo Agent`.

The MCP response includes:

- `contract_address`: should be `{ "index": 12802, "subindex": 0 }`.
- `receive_name`: `CIS-8004.register`.
- `parameter_json`: schema-shaped parameter data.
- `parameter_hex`: serialized parameter bytes.
- `suggested_energy`: a suggested energy amount.
- `sender`: the account that should sign.

## 3. Sign and Submit

Use your Concordium signing tool to submit an update transaction:

```text
contract: <12802,0>
entrypoint: CIS-8004.register
parameter: parameter_hex from build_register
energy: suggested_energy from build_register
sender: YOUR_TESTNET_ACCOUNT
```

With `ccd-agent-kit`, the rough split is:

- `ccd-signer`: signs using locally controlled keys.
- `ccd-chain-client`: queries the chain and submits completed transaction artifacts.

Follow the package README for your installed `ccd-agent-kit` release, because command names and release packaging can change.

## 4. Verify the Registration

After the transaction finalizes, look up the token id returned by the transaction or by recent registry events.

Call `agent_of`:

```bash
curl -s -X POST https://agent-registry-mcp.concordium.com/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/call",
    "params": {
      "name": "agent_of",
      "arguments": {
        "network": "testnet",
        "token_id": "TOKEN_ID"
      }
    }
  }'
```

To verify that the hosted card still matches the on-chain hash, call `verify_agent_card`:

```bash
curl -s -X POST https://agent-registry-mcp.concordium.com/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/call",
    "params": {
      "name": "verify_agent_card",
      "arguments": {
        "network": "testnet",
        "token_id": "TOKEN_ID",
        "fetch": true
      }
    }
  }'
```

## Common Notes

- `agent_uri` can be omitted, but using it with `metadata_hash_hex` makes the registration easier to verify.
- `metadata_hash_hex` must be a 64-character SHA-256 hex string.
- Do not write the reserved metadata key `agentWallet` with `build_set_metadata`; use `build_set_agent_wallet` for payment routing.
- If the Agent Card JSON changes later, host the new JSON and call `build_set_agent_uri` with the new SHA-256 hash.
