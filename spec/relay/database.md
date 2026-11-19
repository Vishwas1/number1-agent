# Relay database (SQLite)
`/data/relay.db`, WAL mode, owned only by `relay-service`. Migrations are numbered SQL files in `apps/relay-service/migrations/`.

## Schema
```sql
CREATE TABLE cards (
  hash         TEXT PRIMARY KEY,          -- hex sha256(JCS(card))
  json         TEXT NOT NULL,
  created_at   TEXT NOT NULL
);

CREATE TABLE directory (
  agent_id       TEXT PRIMARY KEY,        -- agent:ccd:testnet:<idx>/<token>
  owner          TEXT NOT NULL,           -- ccd:testnet:acc:<addr>
  card_hash      TEXT NOT NULL REFERENCES cards(hash),
  attestation_id TEXT NOT NULL REFERENCES attestations(id),
  status         TEXT NOT NULL CHECK (status IN ('active','removed')),
  updated_at     TEXT NOT NULL
);

CREATE TABLE attestation_requests (
  id           TEXT PRIMARY KEY,          -- uuid
  agent_id     TEXT NOT NULL,
  owner        TEXT NOT NULL,
  vpr_json     TEXT NOT NULL,
  status       TEXT NOT NULL CHECK (status IN ('pending','done','failed','expired')),
  fail_code    TEXT,
  created_at   TEXT NOT NULL,
  expires_at   TEXT NOT NULL
);

CREATE TABLE attestations (
  id           TEXT PRIMARY KEY,          -- urn:uuid:…
  agent_id     TEXT NOT NULL,
  owner        TEXT NOT NULL,
  json         TEXT NOT NULL,             -- the signed attestation
  vaa_tx       TEXT NOT NULL,
  issued_at    TEXT NOT NULL,
  expires_at   TEXT NOT NULL
);
CREATE INDEX attestations_agent ON attestations(agent_id, issued_at DESC);

CREATE TABLE envelopes (
  id           TEXT PRIMARY KEY,          -- envelope id (urn:uuid)
  to_agent     TEXT NOT NULL,
  from_agent   TEXT NOT NULL,
  json         TEXT NOT NULL,
  received_at  TEXT NOT NULL,
  expires_at   TEXT NOT NULL
);
CREATE INDEX envelopes_to ON envelopes(to_agent, received_at);

CREATE TABLE presence (
  agent_id     TEXT PRIMARY KEY,
  online       INTEGER NOT NULL,          -- 0/1
  last_seen    TEXT NOT NULL
);

CREATE TABLE auth_nonces (
  nonce        TEXT PRIMARY KEY,
  expires_at   TEXT NOT NULL
);
```

## Retention
| Table | Rule |
|---|---|
| `envelopes` | Deleted on `ack`. Unacknowledged ones are purged after `QUEUE_TTL_DAYS`. |
| `attestation_requests` | Expired or failed rows purged after 7 days |
| `attestations` | Kept, as an audit trail. Only the latest per agent is indexed. |
| `auth_nonces` | Purged after expiry (5 min) |
| `cards` | Kept (content-addressed, small) |

## Notes
- **Single writer:** `relay-service` is the only process that opens the database.
- **Integrity:** nothing in this database is trusted by nodes without verification. Cards are hash-checked, attestations signature- and VAA-checked, and envelopes signature-checked.
