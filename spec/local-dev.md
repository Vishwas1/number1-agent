# Local test setup (relay + 2 nodes on one laptop)
For development and testing: the relay and two node bundles (Kash, Vishwas) on one machine, still against Concordium **testnet**. Communication works the same as in production, because every message goes through the relay.

## 1. Port map
| Bundle | Compose project | Service | Host port | URL |
|---|---|---|---|---|
| Relay | `relay` | `relay-service` | **8080** | `http://localhost:8080` (nodes use `http://host.docker.internal:8080`) |
| Node: Kash | `kash` | `ui` | **3000** | `http://localhost:3000` |
| | | `n1-agent` | **7100** | `http://localhost:7100` (UI only) |
| Node: Vishwas | `vishwas` | `ui` | **3010** | `http://localhost:3010` |
| | | `n1-agent` | **7110** | `http://localhost:7110` (UI only) |
| *(optional)* Node: Pete | `pete` | `ui` / `n1-agent` | **3020 / 7120** | `http://localhost:3020` |

- Everything else has **no host port**: both verifiers, the calendar gateways, and `n1-agent`'s internal listener (7101). Each Compose project gets its **own Docker network**, so service names like `n1-agent` and `calendar-gateway` never clash between nodes.
- Nodes reach the relay through `host.docker.internal`. On macOS and Windows this works out of the box. On Linux it comes from `extra_hosts: host-gateway`, which is already in the node compose file.

## 2. Folder layout
```
deploy/local/
  relay/    .env  data/  secrets/attester-signing.key  secrets/attester-account.export
  kash/     .env  data/  secrets/node-service-account.export
  vishwas/  .env  data/  secrets/node-service-account.export
```
Each node has its **own** node service account, keystore, mock calendar and database. Nothing is shared between node folders.

## 3. `.env` differences
| Variable | relay | kash | vishwas |
|---|---|---|---|
| `RELAY_PORT` / `RELAY_TLS` | `8080` / `false` | — | — |
| `RELAY_DIR` | `../local/relay` | — | — |
| `PUBLIC_BASE_URL` | `http://host.docker.internal:8080` | — | — |
| `UI_PORT` / `N1_PORT` | — | `3000` / `7100` | `3010` / `7110` |
| `NODE_DIR` | — | `../local/kash` | `../local/vishwas` |
| `RELAY_URL` / `RELAY_WS_URL` | — | `http://host.docker.internal:8080` / `ws://host.docker.internal:8080/ws` | same |
| `ATTESTER_PUBLIC_KEY` / `ATTESTER_ACCOUNT` | — | from the local relay's `/.well-known/attester.json` | same |
| `NODE_SERVICE_ACCOUNT` | — | Kash's service account | Vishwas's service account |
| `OWNER_TZ` | — | e.g. `Europe/Copenhagen` | e.g. `Asia/Kolkata` |

`RELAY_DIR` and `NODE_DIR` are relative to the compose file's folder (`deploy/relay/`, `deploy/node/`), which is how Docker Compose resolves volume paths.

All other values (`GRPC_ENDPOINT`, `CIS8004_CONTRACT`, `OPENAI_API_KEY`) can be the same.

- **Card URIs:** these are written on chain as `http://host.docker.internal:8080/cards/<hash>`. That's fine for local testing, but these test agents only work with this local relay. Use separate test accounts and agents for the hosted demo.
- **`ws://` and `http://`** are accepted by `n1-agent` only when `RELAY_URL` points at `host.docker.internal` or `localhost`, so a production node can never be downgraded to plain text.

## 4. Commands
```bash
# relay
docker compose -f deploy/relay/docker-compose.yml --env-file deploy/local/relay/.env -p relay up -d

# nodes (same compose file, different project + env)
docker compose -f deploy/node/docker-compose.yml --env-file deploy/local/kash/.env    -p kash    up -d
docker compose -f deploy/node/docker-compose.yml --env-file deploy/local/vishwas/.env -p vishwas up -d

# logs / stop
docker compose -p vishwas logs -f n1-agent
docker compose -p kash down
```
A `Makefile` wraps these: `make relay`, `make node NAME=kash`, `make down-all`.

## 5. Wallets
| Need | Local setup |
|---|---|
| Browser Wallet (account X per user): setup, attestation **and approvals** | **One browser profile per user** (Chrome profiles "Kash" and "Vishwas"), each with its own Browser Wallet and testnet account. Open `localhost:3000` in Kash's profile and `localhost:3010` in Vishwas's. |
| Name + 18+ attestation | Each Browser Wallet account must be ID-backed with its own test identity. Otherwise the attestation names will be the same. |

## 6. Checks before testing
- `curl localhost:8080/health` → relay, database, verifier and chain are all ok.
- Open both UIs and finish setup in each. Kash's directory should then show Vishwas ✓, and the reverse.
- The clock is synced (NTP), because messages more than ±5 min off are rejected.
- Port conflicts: `lsof -i :3000 -i :3010 -i :7100 -i :7110 -i :8080` should show only Docker.
