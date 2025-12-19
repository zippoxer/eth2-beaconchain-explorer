# beaconcha.in Explorer - Codebase Overview

A comprehensive Ethereum beacon chain explorer built in Go, providing block/slot viewing, validator tracking, staking dashboards, and rich APIs for both consensus and execution layer data.

---

## Architecture Overview

```
                     +-------------------+
                     |  Lighthouse/CL    |
                     |  Beacon Node      |
                     +--------+----------+
                              |
              +---------------+---------------+
              |                               |
     +--------v--------+           +----------v---------+
     |  Slot Exporter  |           |   ETH1 Indexer     |
     |  (CL Data)      |           |   (EL Data)        |
     +--------+--------+           +----------+---------+
              |                               |
              |    +-----------------+        |
              +--->|   PostgreSQL    |<-------+
              |    |  (Relational)   |
              |    +-----------------+
              |
              |    +-----------------+
              +--->|    Bigtable     |
              |    | (Time-series)   |
              |    +-----------------+
              |
              |    +-----------------+
              +--->|     Redis       |
                   |   (Cache)       |
                   +-----------------+
                              |
                    +---------v---------+
                    |    Frontend /     |
                    |    API Server     |
                    +-------------------+
```

**Key insight**: The system uses a **dual-database architecture**:
- **PostgreSQL**: Relational data (validators, epochs, blocks metadata, users, subscriptions)
- **Google Bigtable**: Time-series data (validator balances, attestations, sync duties, EL transactions)

---

## Directory Structure

```
/
+-- cmd/                    # Entry points (binaries)
|   +-- explorer/           # Main web server + optional indexer
|   +-- eth1indexer/        # Execution layer indexer
|   +-- blobindexer/        # EIP-4844 blob data indexer
|   +-- notification-sender/
|   +-- notification-collector/
|   +-- statistics/         # Stats aggregation
|   +-- rewards-exporter/   # Validator rewards export
|   +-- frontend-data-updater/
|   +-- node-jobs-processor/
|   +-- user-service/
|   +-- validator-tagger/   # Entity/pool tagging
|   +-- migrations/         # DB migration runner
|
+-- db/                     # Database layer
|   +-- db.go               # PostgreSQL operations (116K lines)
|   +-- bigtable.go         # Bigtable operations (106K lines)
|   +-- bigtable_eth1.go    # EL-specific bigtable (181K lines)
|   +-- migrations/         # SQL migration files (goose)
|   +-- frontend.go         # Frontend-specific DB queries
|   +-- statistics.go       # Stats queries
|
+-- exporter/               # Data export logic
|   +-- exporter.go         # Main exporter orchestration
|   +-- slot_exporter.go    # CL slot/block export
|   +-- eth1.go             # EL deposit export
|   +-- rocketpool.go       # Rocketpool integration
|   +-- sync_committees.go
|   +-- queues.go           # Validator queue tracking
|
+-- handlers/               # HTTP handlers
|   +-- api.go              # REST API (155K lines - main API)
|   +-- api_eth1.go         # EL-specific API endpoints
|   +-- validator.go        # Validator page handlers
|   +-- slot.go             # Slot/block page handlers
|   +-- auth.go             # Authentication
|   +-- user.go             # User management (110K lines)
|
+-- rpc/                    # Chain node clients
|   +-- lighthouse.go       # Lighthouse CL client (65K lines)
|   +-- erigon.go           # Erigon EL client (38K lines)
|   +-- geth.go             # Geth EL client
|
+-- services/               # Background services
|   +-- services.go         # Service initialization & updaters
|   +-- notifications.go    # Notification system (109K lines)
|   +-- charts_updater.go   # Chart data aggregation
|   +-- validator_tagger_*.go  # Validator entity tagging
|
+-- types/                  # Data types & config
|   +-- config.go           # Configuration structs
|   +-- exporter.go         # Block/validator types
|   +-- eth1.go             # EL types
|   +-- frontend.go         # Frontend types
|
+-- utils/                  # Utilities
+-- templates/              # Go HTML templates
+-- static/                 # Static assets (JS/CSS)
+-- config/                 # Chain configs (mainnet, testnets)
```

---

## Database Schema

### PostgreSQL Tables (Key Tables)

**Consensus Layer:**
- `validators` - Validator state (pubkey, balance, status, activation/exit epochs)
- `validator_performance` - CL/EL/MEV performance metrics
- `validator_stats` - Daily validator statistics
- `epochs` - Epoch metadata (blocks count, participation rate, finalized)
- `blocks` - CL blocks/slots (proposer, attestations, exec payload info)
- `blocks_attestations` - Attestation data per block
- `blocks_deposits` - Deposits included in blocks
- `blocks_withdrawals` - Withdrawals per block
- `sync_committees` - Sync committee assignments

**Execution Layer:**
- `eth1_deposits` - Deposit contract events
- `eth1_deposits_aggregated` - Aggregated deposit stats

**User/Frontend:**
- `users` - User accounts
- `users_subscriptions` - Notification subscriptions
- `users_validators_tags` - User's watched validators

**Integrations:**
- `rocketpool_minipools` - Rocketpool minipool data
- `rocketpool_nodes` - Rocketpool node data
- `relays_blocks` - MEV relay block data

### Bigtable Tables

See `bigtable_config.md` for full setup. Key tables:
- `beaconchain_validators_history` - Validator balance history, attestation duties, sync duties
- `beaconchain_validators` - Current validator attestation data
- `blocks` - EL block data (raw transactions, receipts, traces)
- `data` - Processed EL events (ERC20, ERC721, ERC1155 transfers)
- `metadata` - Address metadata (names, token info, ENS)
- `machine_metrics` - User-submitted machine metrics

---

## Indexer Flow

### Consensus Layer (Slot Exporter)

**Entry**: `exporter/exporter.go` -> `Start()`

```
1. Wait for beacon node availability
2. Start background exporters:
   - networkLivenessUpdater
   - eth1DepositsExporter
   - genesisDepositsExporter
   - syncCommitteesExporter
   - PendingQueueIndexer
   - rocketpoolExporter (if enabled)
   - mevBoostRelaysExporter (if enabled)

3. Main loop (RunSlotExporter):
   a. Get chain head from Lighthouse
   b. On first run: fill gaps in DB
   c. Export new slots (lastDbSlot -> head)
   d. Check non-finalized slots for reorgs
   e. Update finalization status
```

**ExportSlot** (`exporter/slot_exporter.go`):
```
1. Fetch block data from Lighthouse
2. If first slot of epoch:
   - Export epoch duties (attestation/sync assignments) -> Bigtable
   - Save validator balances -> Bigtable
   - Update validators table -> PostgreSQL
3. For each slot:
   - Save attestation duties -> Bigtable
   - Save sync duties -> Bigtable
   - Save block -> PostgreSQL
```

### Execution Layer (ETH1 Indexer)

**Entry**: `cmd/eth1indexer/main.go`

```
1. Connect to Erigon node
2. Connect to Bigtable
3. Main loop:
   a. Handle chain reorgs (check last N blocks)
   b. Index missing blocks (node -> blocks table)
   c. Transform events (blocks table -> data table)
   d. Update balances (if enabled)
```

**Transform Functions** (`db/bigtable_eth1.go`):
- `TransformBlock` - Block metadata
- `TransformTx` - Transactions
- `TransformItx` - Internal transactions (traces)
- `TransformERC20/721/1155` - Token transfers
- `TransformWithdrawals` - CL withdrawals
- `TransformEnsNameRegistered` - ENS registrations

---

## RPC Clients

### Lighthouse Client (`rpc/lighthouse.go`)

Implements standard Beacon API endpoints:
- `GetChainHead()` - Current head/finality
- `GetBlockBySlot(slot)` - Full block with attestations
- `GetValidatorState(epoch)` - Validator set at epoch
- `GetValidatorParticipation(epoch)` - Participation stats
- `GetValidatorQueue()` - Activation/exit queues

Uses SSE for real-time block events.

### Erigon Client (`rpc/erigon.go`)

For execution layer data:
- `GetBlock(number)` - Block with receipts and traces
- `GetBalances(addresses)` - Account balances
- `GetERC20TokenMetadata(address)` - Token info
- `TraceTransaction(hash)` - Internal calls

---

## API Structure

**Base path**: `/api/v1`

### Public Endpoints
- `GET /epoch/{epoch}` - Epoch data
- `GET /slot/{slotOrHash}` - Slot/block data
- `GET /validator/{indexOrPubkey}` - Validator info
- `GET /validator/{indexOrPubkey}/attestations` - Attestation history
- `GET /validator/{indexOrPubkey}/proposals` - Proposal history
- `GET /validator/{indexOrPubkey}/withdrawals` - Withdrawals
- `GET /validators/queue` - Activation/exit queue
- `GET /execution/block/{blockNumber}` - EL block
- `GET /execution/address/{address}` - Address info
- `GET /rocketpool/stats` - Rocketpool network stats

### Authenticated Endpoints (`/api/v1/user/`)
- `POST /mobile/notify/register` - Register push notifications
- `GET /validator/saved` - User's watched validators
- `POST /notifications/subscribe` - Subscribe to events
- `GET /stats` - User's machine stats

---

## Configuration

**Config file**: `config.yaml` or env vars

Key sections:
```yaml
readerDatabase:    # PostgreSQL read replica
writerDatabase:    # PostgreSQL primary
bigtable:          # GCP Bigtable settings
chain:             # Network config (mainnet/testnet)
  clConfigPath:    # Path to CL chain spec
indexer:
  enabled: true
  node:
    host: "localhost"
    port: "5052"
    type: "lighthouse"
frontend:
  enabled: true
  server:
    port: "3333"
eth1ErigonEndpoint: "http://localhost:8545"
redisCacheEndpoint: "localhost:6379"
```

---

## Build & Run

```bash
# Build all binaries
make all

# Build specific binary
make explorer
make eth1indexer

# Run explorer (frontend + optional indexer)
./bin/explorer --config config.yaml

# Run ETH1 indexer separately
./bin/eth1indexer --config config.yaml

# Run database migrations
./bin/migrations --config config.yaml
```

---

## Key Files to Read

| Understanding... | Read these files |
|-----------------|------------------|
| Overall architecture | `cmd/explorer/main.go` |
| CL indexing | `exporter/slot_exporter.go`, `exporter/exporter.go` |
| EL indexing | `cmd/eth1indexer/main.go`, `db/bigtable_eth1.go` |
| Database schema | `db/migrations/20230330131622_init_database.sql` |
| Bigtable schema | `bigtable_config.md` |
| Lighthouse RPC | `rpc/lighthouse.go` |
| API endpoints | `handlers/api.go` |
| Configuration | `types/config.go` |
| Validator data | `db/db.go` (search for `SaveValidators`, `SaveBlock`) |

---

## Important Patterns

### Reader/Writer DB Split
The system uses separate reader/writer DB connections for PostgreSQL:
```go
db.WriterDb  // For writes
db.ReaderDb  // For reads (can be replica)
```

### Bigtable Row Keys
Row keys are designed for efficient range scans:
- Validators: `<chainId>:v:<reversedValidatorIndex>:<reversedEpoch>`
- EL Blocks: `<chainId>:b:<reversedBlockNumber>`
- Addresses: `<chainId>:a:<address>:<reversedBlockNumber>`

The "reversed" numbers enable descending-order scans.

### Transaction Handling
Database writes use transactions with rollback:
```go
tx, _ := db.WriterDb.Beginx()
defer tx.Rollback()
// ... operations ...
tx.Commit()
```

### Service Status Reporting
Services report health to `service_status` table:
```go
services.ReportStatus("slotExporter", "Running", nil)
```

---

## UI (Brief)

- **Templates**: Go HTML templates in `templates/`
- **Frontend Framework**: Bootstrap 4
- **Charts**: Highcharts (note: requires license for commercial use)
- **Static assets**: Bundled via `cmd/bundle/main.go` into `static/` embed

The frontend is server-rendered with some AJAX for data tables.
