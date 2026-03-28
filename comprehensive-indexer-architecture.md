# Umbra Comprehensive Indexer — Architecture Plan

## Executive Summary

A high-performance, fault-tolerant indexer that captures **every instruction and event** from the Umbra program, tracks MPC computation lifecycles (pending → confirmed/failed), and stores all data in a normalized PostgreSQL schema. Built on the existing `indexer-framework` crate with Helius Laserstream for sub-second ingestion latency.

---

## 1. High-Level Architecture

The key design constraint: **the Laserstream consumer must be as lightweight as possible**. It does zero parsing — just receives raw transaction bytes and pushes them to a durable queue. All heavy work (parsing, handler routing, DB writes) happens in a separate write service.

```
  ┌─────────────────────┐
  │   Helius Laserstream │
  │   (gRPC stream)      │
  └──────────┬──────────┘
             │
  ┌──────────▼──────────┐
  │   Consumer Service   │  ← ULTRA-LIGHT: receive + enqueue only
  │   (Rust binary)      │
  │                      │
  │  - Receive raw TX    │
  │  - Extract: sig,     │
  │    slot, raw_bytes   │
  │  - Push to SQS       │
  │  - Track checkpoint  │
  │  - NO parsing        │
  │  - NO DB writes      │
  └──────────┬──────────┘
             │
  ┌──────────▼──────────┐
  │  AWS SQS Standard    │  ← Durable queue (fully managed, same region)
  │                      │
  │  Queue:              │
  │    umbra-indexer-txs  │
  │  DLQ:                │
  │    umbra-indexer-dlq  │
  │  Retention: 14 days  │
  │  Latency: ~1-5ms     │
  └──────────┬──────────┘
             │
  ┌──────────▼──────────┐
  │   Write Service      │  ← HEAVY LIFTER: parse, route, write
  │   (Rust binary)      │
  │                      │
  │  ┌────────────────┐  │
  │  │ TX Parser       │  │
  │  │ (borsh decode)  │  │
  │  └───────┬────────┘  │
  │          │           │
  │  ┌───────▼────────┐  │
  │  │ Handler Router  │  │
  │  │ (discriminator  │  │
  │  │  → handler fn)  │  │
  │  └───────┬────────┘  │
  │          │           │
  │  ┌───────▼────────┐  │
  │  │ Batch Writer    │  │
  │  │ (DB inserts)    │  │
  │  └───────┬────────┘  │
  │          │           │
  │  ┌───────▼────────┐  │
  │  │ MPC Timeout     │  │
  │  │ Worker (bg)     │  │
  │  └────────────────┘  │
  └──────────┼──────────┘
             │
  ┌──────────▼──────────┐
  │   PostgreSQL 16      │
  │   (Primary)          │
  │                      │
  │   - transactions     │
  │   - instructions     │
  │   - mpc_computations │
  │   - events (per-type)│
  └──────────┬──────────┘
             │ WAL replication
  ┌──────────▼──────────┐
  │   PostgreSQL 16      │
  │   (Read Replica)     │
  └──────────┬──────────┘
             │
  ┌──────────┼──────────────────┐
  │          │                  │
  ▼          ▼                  ▼
 REST API   WebSocket       Backfill
 (Read Svc) (Live events)   Worker
```

**Why this split matters:**
- Consumer never blocks on DB latency, parse errors, or write retries
- If the write service crashes or the DB goes down, SQS buffers all messages (14-day retention, unlimited queue depth)
- Consumer can be restarted independently — it only needs Laserstream connection + SQS connection
- Write service can be scaled horizontally (multiple instances long-polling from the same queue)
- SQS is fully managed — zero ops, automatic scaling, no clusters to monitor
- Built-in dead-letter queue for poison messages
- At-least-once delivery + idempotent writes = exactly-once semantics

---

## 2. Core Design Principles

**P1: Consumer never blocks** — The Laserstream consumer does ZERO parsing and ZERO DB writes. It receives raw bytes, wraps them in a minimal envelope `{signature, slot, raw_bytes}`, and pushes to SQS via `SendMessageBatch`. Its only failure modes are Laserstream disconnect (auto-reconnect built into library) and SQS unavailability (in-memory buffer + exponential retry). This is the thinnest possible layer.

**P2: SQS decouples ingestion from processing** — If the write service is down, the DB is slow, or parsing fails — the consumer keeps running and SQS buffers everything (14-day retention, unlimited depth). The write service can be restarted, scaled, or debugged without losing a single transaction. SQS is fully managed with 99.999999999% durability.

**P3: Handler registry for extensibility** — Each Umbra instruction type maps to a handler function in the write service. Adding a new instruction = adding one handler + one migration. Zero changes to the consumer or SQS configuration.

**P4: Idempotent writes** — Every write uses `ON CONFLICT DO NOTHING` keyed on `(signature, instruction_index)`. Safe to replay any range. SQS delivers at-least-once — idempotent writes handle duplicates naturally.

**P5: MPC lifecycle is first-class** — Queue and callback instructions are linked via `computation_offset`. A background worker in the write service marks computations as `timeout` after 180 blocks with no callback.

**P6: Store raw + parsed** — Raw transaction bytes are stored alongside parsed/structured event data. This allows re-parsing if the schema evolves without needing to re-fetch from chain.

---

## 3. Consumer Service (Laserstream → SQS)

The consumer is intentionally minimal. Its entire job is: receive gRPC stream → send to SQS.

### 3.1 Data Source: Helius Laserstream

Reuse the proven `indexer-framework` crate with `MixerProducer`-style streaming, but **subscribe to ALL transactions** for the Umbra program ID (not just mixer instructions).

```rust
// Subscribe to all Umbra program transactions
LaserStreamConfig {
    program_ids: vec![UMBRA_PROGRAM_ID],
    commitment: CommitmentLevel::Confirmed,
    // No instruction filter — capture everything
}
```

**Why Laserstream over alternatives:**
- Sub-200ms latency (vs. ~2-5s for RPC polling)
- Guaranteed delivery with built-in reconnection
- Already battle-tested in the existing utxo-indexer
- Regional endpoints (EWR) for low-latency US East deployment

### 3.2 SQS Message Format

The consumer extracts the absolute minimum from the gRPC payload and sends it to SQS:

```rust
// What gets sent to SQS — minimal envelope, no parsing
struct SqsTransactionEnvelope {
    signature: String,          // TX signature (base58)
    slot: u64,                  // Slot number
    block_time: Option<i64>,    // Unix timestamp (if available)
    raw_data: Vec<u8>,          // Full serialized transaction bytes
}
// Serialized with bincode (compact binary) → base64 for SQS message body
// Solana TXs are max ~1232 bytes — well within SQS 256KB limit
```

**SQS Configuration:**
- Queue name: `umbra-indexer-txs`
- Queue type: Standard (not FIFO — idempotent writes handle duplicates, avoids 3K msg/s FIFO limit)
- Visibility timeout: 60 seconds (time for write service to process a batch)
- Message retention: 14 days (maximum — provides ample buffer for outages)
- Dead-letter queue: `umbra-indexer-txs-dlq` (after 3 receive attempts)
- Encryption: SSE-SQS (server-side encryption at rest)

### 3.3 Consumer Internal Buffering

Even within the consumer, a small in-memory buffer protects against SQS hiccups:

```
Laserstream gRPC → tokio::mpsc (unbounded) → SQS Batch Sender task
                                               │
                                               ├─ Batches up to 10 messages per SendMessageBatch
                                               ├─ Flushes every 100ms or when batch is full
                                               ├─ On SQS error: exponential backoff (50ms → 5s)
                                               └─ On persistent failure: log + buffer in memory
```

SQS `SendMessageBatch` sends up to 10 messages per API call, reducing round-trips.

### 3.4 Checkpoint Tracking

The consumer tracks its progress in DynamoDB (zero PostgreSQL dependency):

```rust
// DynamoDB table: indexer-checkpoints
// Partition key: id (String)
struct ConsumerCheckpoint {
    id: String,                // "laserstream_consumer"
    last_processed_slot: u64,
    last_signature: String,
    updated_at: u64,           // Unix timestamp
}
```

On restart, the consumer reads the checkpoint from DynamoDB and tells Laserstream to resume from `last_processed_slot`. Any overlap is fine — idempotent writes downstream handle duplicates.

---

## 4. Write Service (SQS → Parse → PostgreSQL)

The write service is the heavy lifter. It long-polls from SQS, parses transactions, routes to handlers, and batch-writes to PostgreSQL.

### 4.1 SQS Long-Polling

```rust
// Long-poll SQS — blocks up to 20 seconds waiting for messages
// Receives up to 10 messages per call (SQS maximum)
loop {
    let messages = sqs.receive_message()
        .queue_url(&queue_url)
        .max_number_of_messages(10)
        .wait_time_seconds(20)          // long-polling (reduces API calls)
        .visibility_timeout(60)         // 60s to process
        .send()
        .await?;

    if let Some(msgs) = messages.messages {
        let batch = parse_and_handle(msgs);
        write_batch_to_db(batch).await?;

        // Delete successfully processed messages
        sqs.delete_message_batch()
            .queue_url(&queue_url)
            .set_entries(Some(delete_entries))
            .send()
            .await?;
    }
}
```

### 4.2 Message Failure Handling

SQS handles crash recovery automatically:

- If the write service crashes mid-batch, undeleted messages become visible again after the visibility timeout (60s)
- After 3 failed receive attempts (tracked by SQS `ApproximateReceiveCount`), messages move to the DLQ
- No manual reclaim logic needed — SQS manages this natively
- Multiple write service instances can long-poll from the same queue — SQS distributes messages automatically

### 4.3 Transaction Parser

For each message from SQS:

1. Deserialize the `SqsTransactionEnvelope` from the message body (base64 → bincode)
2. Extract transaction metadata (signature, slot, block_time, fee, compute_units, success/failure)
3. Walk the instruction list and inner instructions
4. For each instruction targeting the Umbra program:
   - Read the 8-byte Anchor discriminator
   - Route to the matching handler via the Handler Registry
   - Decode instruction data (borsh) and accounts
5. Extract CPI events from transaction logs (Anchor event discriminator)
6. Accumulate all DB operations from the batch

### 4.4 Handler Registry Pattern

```rust
pub trait InstructionHandler: Send + Sync {
    /// Unique Anchor discriminator for this instruction
    fn discriminator(&self) -> [u8; 8];

    /// Human-readable instruction name for the `instructions` table
    fn instruction_name(&self) -> &'static str;

    /// Parse the instruction and return DB write operations
    fn handle(
        &self,
        ctx: &InstructionContext,  // signature, slot, accounts, data, logs
    ) -> Result<Vec<DbOperation>>;
}

// Registry is a HashMap<[u8; 8], Box<dyn InstructionHandler>>
pub struct HandlerRegistry {
    handlers: HashMap<[u8; 8], Box<dyn InstructionHandler>>,
}
```

**Adding a new instruction** requires:
1. Implement `InstructionHandler` for the new instruction
2. Register it in the write service registry startup
3. Add a migration for any new event-specific columns
4. Done — no changes to consumer or SQS

### 4.5 Batch Writing

The write service processes SQS messages in batches:

- **SQS batch**: Up to 10 messages per `ReceiveMessage` call (SQS maximum)
- **DB batch**: All operations from those 10 TXs are written in a single PostgreSQL transaction
- **Atomicity**: If any write fails, the entire batch rolls back. Messages remain visible in SQS (not deleted) and will be re-received.
- **Poison messages**: If a message fails processing 3 times (tracked by SQS `ApproximateReceiveCount`), SQS automatically moves it to the dead-letter queue
- **Concurrency**: Multiple write service instances long-poll the same queue — SQS distributes messages automatically, each message goes to exactly one consumer

---

## 5. Database Schema

### 5.1 Core Tables

```sql
-- Tracks indexer progress for crash recovery
CREATE TABLE indexer_checkpoints (
    id              TEXT PRIMARY KEY,       -- e.g. 'laserstream_main'
    last_slot       BIGINT NOT NULL,
    last_signature  TEXT NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Every transaction touching the Umbra program
CREATE TABLE transactions (
    signature       TEXT PRIMARY KEY,
    slot            BIGINT NOT NULL,
    block_time      TIMESTAMPTZ,
    success         BOOLEAN NOT NULL,
    fee_lamports    BIGINT NOT NULL,
    compute_units   INTEGER,
    raw_data        BYTEA,                  -- full transaction bytes for re-parsing
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_transactions_slot ON transactions (slot);
CREATE INDEX idx_transactions_block_time ON transactions (block_time);

-- Every Umbra instruction within a transaction
CREATE TABLE instructions (
    id                  BIGSERIAL PRIMARY KEY,
    signature           TEXT NOT NULL REFERENCES transactions(signature),
    instruction_index   SMALLINT NOT NULL,
    inner_index         SMALLINT,            -- NULL for top-level
    instruction_type    TEXT NOT NULL,        -- e.g. 'public_token_deposit_into_existing_mxe_balance_v3'
    instruction_category TEXT NOT NULL,       -- e.g. 'deposit', 'claim', 'transfer', 'utxo', 'config'
    is_callback         BOOLEAN NOT NULL DEFAULT FALSE,
    is_mpc              BOOLEAN NOT NULL DEFAULT FALSE,
    accounts            JSONB NOT NULL,       -- ordered list of account pubkeys
    raw_data            BYTEA,                -- raw instruction data
    slot                BIGINT NOT NULL,
    block_time          TIMESTAMPTZ,
    success             BOOLEAN NOT NULL,

    UNIQUE (signature, instruction_index, COALESCE(inner_index, -1))
);

CREATE INDEX idx_instructions_type ON instructions (instruction_type);
CREATE INDEX idx_instructions_category ON instructions (instruction_category);
CREATE INDEX idx_instructions_slot ON instructions (slot);
CREATE INDEX idx_instructions_block_time ON instructions (block_time);
```

### 5.2 MPC Computation Lifecycle

```sql
CREATE TYPE mpc_status AS ENUM ('pending', 'confirmed', 'failed', 'timeout');

CREATE TABLE mpc_computations (
    computation_offset  TEXT NOT NULL,
    queue_signature     TEXT NOT NULL REFERENCES transactions(signature),
    queue_slot          BIGINT NOT NULL,
    queue_block_time    TIMESTAMPTZ,
    instruction_type    TEXT NOT NULL,        -- the queue instruction name
    instruction_category TEXT NOT NULL,

    -- Callback info (NULL until callback arrives)
    callback_signature  TEXT REFERENCES transactions(signature),
    callback_slot       BIGINT,
    callback_block_time TIMESTAMPTZ,

    status              mpc_status NOT NULL DEFAULT 'pending',
    status_updated_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    -- Key participants
    user_address        TEXT,
    mint_address        TEXT,
    receiver_address    TEXT,                 -- for transfers/deposits to others

    -- Amounts (public where available)
    amount              BIGINT,              -- deposit/transfer/withdraw amount if public
    protocol_fees       BIGINT,
    relayer_fees        BIGINT,

    -- Callback outputs (from MPC)
    callback_outputs    JSONB,               -- structured MPC outputs

    PRIMARY KEY (computation_offset, queue_signature)
);

CREATE INDEX idx_mpc_status ON mpc_computations (status) WHERE status = 'pending';
CREATE INDEX idx_mpc_user ON mpc_computations (user_address);
CREATE INDEX idx_mpc_queue_slot ON mpc_computations (queue_slot);
CREATE INDEX idx_mpc_category ON mpc_computations (instruction_category);
```

### 5.3 Event-Specific Tables (Per Category)

Each major operation category gets its own table for structured querying. These are populated by the handlers alongside the generic `instructions` table.

```sql
-- ============================================================
-- DEPOSITS (public token → encrypted balance)
-- ============================================================
CREATE TABLE deposit_events (
    id                  BIGSERIAL PRIMARY KEY,
    signature           TEXT NOT NULL,
    instruction_index   SMALLINT NOT NULL,
    slot                BIGINT NOT NULL,
    block_time          TIMESTAMPTZ,

    -- Participants
    depositor_address   TEXT NOT NULL,
    receiver_address    TEXT NOT NULL,
    mint_address        TEXT NOT NULL,

    -- Amounts
    transfer_amount     BIGINT NOT NULL,
    deposit_amount      BIGINT NOT NULL,

    -- Type info
    target_mode         TEXT NOT NULL,        -- 'mxe' or 'shared'
    is_new_account      BOOLEAN NOT NULL,     -- new vs existing token account
    computation_offset  TEXT NOT NULL,

    -- Fees
    priority_fees       BIGINT,
    protocol_fees       BIGINT,              -- populated on callback

    -- Metadata
    purpose             INTEGER,
    optional_data       BYTEA,

    -- MPC status
    mpc_status          mpc_status NOT NULL DEFAULT 'pending',
    callback_signature  TEXT,
    callback_slot       BIGINT,

    UNIQUE (signature, instruction_index)
);

CREATE INDEX idx_deposits_depositor ON deposit_events (depositor_address);
CREATE INDEX idx_deposits_receiver ON deposit_events (receiver_address);
CREATE INDEX idx_deposits_mint ON deposit_events (mint_address);
CREATE INDEX idx_deposits_slot ON deposit_events (slot);
CREATE INDEX idx_deposits_status ON deposit_events (mpc_status) WHERE mpc_status = 'pending';

-- ============================================================
-- TRANSFERS (encrypted balance → encrypted balance)
-- ============================================================
CREATE TABLE transfer_events (
    id                  BIGSERIAL PRIMARY KEY,
    signature           TEXT NOT NULL,
    instruction_index   SMALLINT NOT NULL,
    slot                BIGINT NOT NULL,
    block_time          TIMESTAMPTZ,

    sender_address      TEXT NOT NULL,
    receiver_address    TEXT NOT NULL,
    mint_address        TEXT NOT NULL,

    target_mode         TEXT NOT NULL,        -- 'mxe' or 'shared'
    is_new_account      BOOLEAN NOT NULL,
    computation_offset  TEXT NOT NULL,

    -- Encrypted amount (Rescue ciphertext)
    rescue_encryption_nonce BYTEA,
    rescue_encrypted_amount BYTEA,

    -- Fees / Metadata
    priority_fees       BIGINT,
    optional_data       BYTEA,

    mpc_status          mpc_status NOT NULL DEFAULT 'pending',
    callback_signature  TEXT,
    callback_slot       BIGINT,

    UNIQUE (signature, instruction_index)
);

CREATE INDEX idx_transfers_sender ON transfer_events (sender_address);
CREATE INDEX idx_transfers_receiver ON transfer_events (receiver_address);
CREATE INDEX idx_transfers_mint ON transfer_events (mint_address);
CREATE INDEX idx_transfers_slot ON transfer_events (slot);

-- ============================================================
-- WITHDRAWALS (encrypted balance → public ATA)
-- ============================================================
CREATE TABLE withdrawal_events (
    id                  BIGSERIAL PRIMARY KEY,
    signature           TEXT NOT NULL,
    instruction_index   SMALLINT NOT NULL,
    slot                BIGINT NOT NULL,
    block_time          TIMESTAMPTZ,

    withdrawer_address  TEXT NOT NULL,
    destination_address TEXT,                -- if different from withdrawer
    mint_address        TEXT NOT NULL,

    withdrawal_amount   BIGINT NOT NULL,     -- public (plaintext)
    computation_offset  TEXT NOT NULL,

    optional_data       BYTEA,

    mpc_status          mpc_status NOT NULL DEFAULT 'pending',
    callback_signature  TEXT,
    callback_slot       BIGINT,

    UNIQUE (signature, instruction_index)
);

CREATE INDEX idx_withdrawals_withdrawer ON withdrawal_events (withdrawer_address);
CREATE INDEX idx_withdrawals_mint ON withdrawal_events (mint_address);
CREATE INDEX idx_withdrawals_slot ON withdrawal_events (slot);

-- ============================================================
-- UTXO CREATION (balance → mixer tree)
-- ============================================================
CREATE TABLE utxo_creation_events (
    id                  BIGSERIAL PRIMARY KEY,
    signature           TEXT NOT NULL,
    instruction_index   SMALLINT NOT NULL,
    slot                BIGINT NOT NULL,
    block_time          TIMESTAMPTZ,

    depositor_address   TEXT NOT NULL,
    mint_address        TEXT NOT NULL,
    source_type         TEXT NOT NULL,        -- 'confidential' or 'public'
    computation_offset  TEXT NOT NULL,

    -- UTXO commitments (from handler event data section)
    insertion_h2_commitment  TEXT,
    insertion_timestamp      BIGINT,
    aes_encrypted_data       BYTEA,

    -- Callback outputs
    tree_index               INTEGER,
    insertion_index          BIGINT,
    final_commitment         TEXT,
    h1_hash                  TEXT,
    h2_hash                  TEXT,
    h1_circuit_provable_hash TEXT,
    h1_smart_program_provable_hash TEXT,
    depositor_x25519_public_key BYTEA,

    optional_data       BYTEA,
    purpose             INTEGER,

    mpc_status          mpc_status NOT NULL DEFAULT 'pending',
    callback_signature  TEXT,
    callback_slot       BIGINT,

    UNIQUE (signature, instruction_index)
);

CREATE INDEX idx_utxo_creation_depositor ON utxo_creation_events (depositor_address);
CREATE INDEX idx_utxo_creation_mint ON utxo_creation_events (mint_address);
CREATE INDEX idx_utxo_creation_tree ON utxo_creation_events (tree_index, insertion_index);
CREATE INDEX idx_utxo_creation_commitment ON utxo_creation_events (final_commitment);
CREATE INDEX idx_utxo_creation_slot ON utxo_creation_events (slot);

-- ============================================================
-- CLAIMS (mixer tree → balance or public ATA)
-- ============================================================
CREATE TABLE claim_events (
    id                  BIGSERIAL PRIMARY KEY,
    signature           TEXT NOT NULL,
    instruction_index   SMALLINT NOT NULL,
    slot                BIGINT NOT NULL,
    block_time          TIMESTAMPTZ,

    relayer_address     TEXT,
    receiver_address    TEXT NOT NULL,
    mint_address        TEXT NOT NULL,
    claim_target        TEXT NOT NULL,        -- 'existing_mxe', 'existing_shared', 'new_mxe', 'new_shared', 'public'
    computation_offset  TEXT NOT NULL,

    number_of_burns     SMALLINT,

    -- ZK proof data (stored if available)
    has_zk_proof        BOOLEAN NOT NULL DEFAULT FALSE,

    optional_data       BYTEA,

    mpc_status          mpc_status NOT NULL DEFAULT 'pending',
    callback_signature  TEXT,
    callback_slot       BIGINT,

    -- Callback: nullifiers burned
    nullifiers_burned   JSONB,               -- array of nullifier hashes

    UNIQUE (signature, instruction_index)
);

CREATE INDEX idx_claims_receiver ON claim_events (receiver_address);
CREATE INDEX idx_claims_relayer ON claim_events (relayer_address);
CREATE INDEX idx_claims_mint ON claim_events (mint_address);
CREATE INDEX idx_claims_slot ON claim_events (slot);
CREATE INDEX idx_claims_target ON claim_events (claim_target);

-- ============================================================
-- USER ACCOUNT EVENTS
-- ============================================================
CREATE TABLE user_account_events (
    id                  BIGSERIAL PRIMARY KEY,
    signature           TEXT NOT NULL,
    instruction_index   SMALLINT NOT NULL,
    slot                BIGINT NOT NULL,
    block_time          TIMESTAMPTZ,

    event_type          TEXT NOT NULL,        -- 'init_user', 'init_token', 'update_user', 'update_token',
                                             -- 'register_x25519', 'register_anonymous', 'register_token_key',
                                             -- 'update_seed', 'claim_staged_sol', 'claim_staged_spl'
    user_address        TEXT NOT NULL,
    mint_address        TEXT,                -- for token-specific events
    accounts            JSONB,
    args                JSONB,               -- instruction-specific arguments

    -- MPC fields (only for register_anonymous)
    computation_offset  TEXT,
    mpc_status          mpc_status,
    callback_signature  TEXT,
    callback_slot       BIGINT,

    UNIQUE (signature, instruction_index)
);

CREATE INDEX idx_user_events_user ON user_account_events (user_address);
CREATE INDEX idx_user_events_type ON user_account_events (event_type);
CREATE INDEX idx_user_events_slot ON user_account_events (slot);

-- ============================================================
-- COMPLIANCE GRANT EVENTS
-- ============================================================
CREATE TABLE compliance_events (
    id                  BIGSERIAL PRIMARY KEY,
    signature           TEXT NOT NULL,
    instruction_index   SMALLINT NOT NULL,
    slot                BIGINT NOT NULL,
    block_time          TIMESTAMPTZ,

    event_type          TEXT NOT NULL,        -- 'create_network_mxe', 'create_network_shared',
                                             -- 'create_user', 'delete_network_mxe',
                                             -- 'delete_network_shared', 'delete_user'
    granter_x25519_hex  TEXT,
    receiver_x25519_hex TEXT,
    receiver_address    TEXT,
    nonce               TEXT,
    grant_pda           TEXT,

    computation_offset  TEXT,
    mpc_status          mpc_status,
    callback_signature  TEXT,
    callback_slot       BIGINT,

    UNIQUE (signature, instruction_index)
);

CREATE INDEX idx_compliance_receiver ON compliance_events (receiver_address);
CREATE INDEX idx_compliance_type ON compliance_events (event_type);
CREATE INDEX idx_compliance_slot ON compliance_events (slot);

-- ============================================================
-- CONVERSION EVENTS (MXE→Shared, Re-encrypt)
-- ============================================================
CREATE TABLE conversion_events (
    id                  BIGSERIAL PRIMARY KEY,
    signature           TEXT NOT NULL,
    instruction_index   SMALLINT NOT NULL,
    slot                BIGINT NOT NULL,
    block_time          TIMESTAMPTZ,

    event_type          TEXT NOT NULL,        -- 'mxe_to_shared', 'reencrypt'
    user_address        TEXT NOT NULL,
    mint_address        TEXT NOT NULL,
    computation_offset  TEXT NOT NULL,

    mpc_status          mpc_status NOT NULL DEFAULT 'pending',
    callback_signature  TEXT,
    callback_slot       BIGINT,

    UNIQUE (signature, instruction_index)
);

CREATE INDEX idx_conversion_user ON conversion_events (user_address);
CREATE INDEX idx_conversion_slot ON conversion_events (slot);

-- ============================================================
-- CONFIG / ADMIN EVENTS (pool, relayer, fees, zk keys, etc.)
-- ============================================================
CREATE TABLE config_events (
    id                  BIGSERIAL PRIMARY KEY,
    signature           TEXT NOT NULL,
    instruction_index   SMALLINT NOT NULL,
    slot                BIGINT NOT NULL,
    block_time          TIMESTAMPTZ,

    event_type          TEXT NOT NULL,        -- 'init_program', 'init_pool', 'update_pool',
                                             -- 'activate_confidentiality', 'activate_mixer',
                                             -- 'init_relayer', 'update_relayer', 'init_fees',
                                             -- 'update_fees', 'init_zk_key', 'init_mixer_tree',
                                             -- 'init_treap', 'init_wallet_specifier', etc.
    actor_address       TEXT NOT NULL,       -- who performed the action
    target_account      TEXT,                -- the account being configured
    mint_address        TEXT,                -- if mint-specific
    accounts            JSONB,
    args                JSONB,               -- all instruction arguments

    -- MPC fields (only for fee pool init/collection)
    computation_offset  TEXT,
    mpc_status          mpc_status,
    callback_signature  TEXT,
    callback_slot       BIGINT,

    UNIQUE (signature, instruction_index)
);

CREATE INDEX idx_config_type ON config_events (event_type);
CREATE INDEX idx_config_slot ON config_events (slot);
CREATE INDEX idx_config_actor ON config_events (actor_address);

-- ============================================================
-- RELAYER FEE EVENTS
-- ============================================================
CREATE TABLE relayer_fee_events (
    id                  BIGSERIAL PRIMARY KEY,
    signature           TEXT NOT NULL,
    instruction_index   SMALLINT NOT NULL,
    slot                BIGINT NOT NULL,
    block_time          TIMESTAMPTZ,

    event_type          TEXT NOT NULL,        -- 'init_relayer_pool', 'collect_relayer_fees',
                                             -- 'collect_protocol_fees', 'withdraw_staging_sol'
    relayer_address     TEXT,
    mint_address        TEXT,
    computation_offset  TEXT,

    mpc_status          mpc_status,
    callback_signature  TEXT,
    callback_slot       BIGINT,

    accounts            JSONB,
    args                JSONB,

    UNIQUE (signature, instruction_index)
);

CREATE INDEX idx_relayer_fee_relayer ON relayer_fee_events (relayer_address);
CREATE INDEX idx_relayer_fee_slot ON relayer_fee_events (slot);
```

### 5.4 Materialized Views (for fast queries)

```sql
-- Daily volume per mint (refreshed every 5 minutes)
CREATE MATERIALIZED VIEW daily_volume AS
SELECT
    date_trunc('day', block_time) AS day,
    mint_address,
    COUNT(*) FILTER (WHERE TRUE) AS total_operations,
    COUNT(*) FILTER (WHERE mpc_status = 'confirmed') AS confirmed_operations,
    COUNT(*) FILTER (WHERE mpc_status = 'failed' OR mpc_status = 'timeout') AS failed_operations,
    SUM(deposit_amount) FILTER (WHERE mpc_status = 'confirmed') AS total_deposit_volume
FROM deposit_events
WHERE block_time IS NOT NULL
GROUP BY 1, 2;

-- User activity summary
CREATE MATERIALIZED VIEW user_activity AS
SELECT
    user_address,
    COUNT(*) AS total_instructions,
    MAX(block_time) AS last_active,
    MIN(block_time) AS first_seen,
    COUNT(DISTINCT signature) AS total_transactions
FROM (
    SELECT depositor_address AS user_address, block_time, signature FROM deposit_events
    UNION ALL
    SELECT sender_address, block_time, signature FROM transfer_events
    UNION ALL
    SELECT withdrawer_address, block_time, signature FROM withdrawal_events
    UNION ALL
    SELECT depositor_address, block_time, signature FROM utxo_creation_events
    UNION ALL
    SELECT receiver_address, block_time, signature FROM claim_events
) combined
GROUP BY user_address;
```

---

## 6. MPC Computation Timeout Worker

A background Tokio task that runs every 30 seconds:

```
┌─────────────────────────────────────────────┐
│             MPC Timeout Worker              │
│                                             │
│  1. Query pending computations where        │
│     current_slot - queue_slot > 180         │
│                                             │
│  2. Batch update status → 'timeout'         │
│                                             │
│  3. Update corresponding event tables       │
│     (deposit_events, transfer_events, etc.) │
│                                             │
│  4. Emit metric: mpc_timeouts_total         │
│                                             │
│  Runs every: 30 seconds                     │
│  Batch size: 500 per tick                   │
└─────────────────────────────────────────────┘
```

**Timeout logic:**
- Current slot is tracked from the latest SQS messages processed (the write service sees slot numbers as it processes)
- `status = 'timeout'` when `latest_slot - queue_slot > 180` (~72 seconds at 400ms/slot)
- Separate from `'failed'` which is set when the transaction itself reverts

**Callback linking:**
- When a callback instruction arrives, the handler looks up the `mpc_computations` row by matching `computation_offset` from the callback's accounts
- Updates `callback_signature`, `callback_slot`, `status = 'confirmed'`, and `callback_outputs`
- Also updates the corresponding event table row (deposit_events, transfer_events, etc.)

---

## 7. Service Architecture (Deployable Binaries on AWS)

### 7.1 Consumer Service (`umbra-indexer-consumer`)

Ultra-light binary. Only two dependencies: Helius Laserstream + AWS SQS SDK.

```
Responsibilities:
  - Connect to Helius Laserstream gRPC
  - Receive raw transaction bytes
  - Wrap in minimal envelope (signature, slot, raw_bytes)
  - SendMessageBatch to SQS (up to 10 per API call)
  - Track checkpoint in DynamoDB
  - NO parsing, NO DB writes

Crash recovery:
  - On startup, read checkpoint from DynamoDB
  - Resume Laserstream from last_processed_slot
  - Any duplicates handled by idempotent writes downstream

Health checks:
  - /health/live   → process is running
  - /health/ready  → Laserstream connected AND SQS reachable
```

**Deployment**: ECS Fargate task (0.5 vCPU, 1GB RAM is more than enough).
Always exactly 1 instance — Laserstream is a single-consumer stream.

### 7.2 Write Service (`umbra-indexer-write`)

Heavy lifter. Long-polls from SQS, parses, routes to handlers, writes to RDS PostgreSQL.

```
Responsibilities:
  - Long-poll SQS (ReceiveMessage with WaitTimeSeconds=20)
  - Parse transactions (borsh decode, discriminator routing)
  - Route to handler registry
  - Batch-write to PostgreSQL (RDS)
  - Delete processed messages from SQS
  - Run MPC timeout worker as background task

Crash recovery:
  - Undeleted messages become visible again after visibility timeout (60s)
  - SQS moves poison messages to DLQ after 3 failed attempts
  - Idempotent writes handle re-processing

Health checks:
  - /health/live   → process is running
  - /health/ready  → SQS reachable AND db connected
  - /health/lag    → SQS ApproximateNumberOfMessagesVisible
```

**Deployment**: ECS Fargate task (2 vCPU, 4GB RAM).
Can scale horizontally — multiple instances long-poll the same queue, SQS distributes messages automatically.

### 7.3 API Service (`umbra-indexer-api`)

Read-only REST API backed by the RDS read replica.

```
Endpoints:

  GET /v1/transactions?slot_start=&slot_end=&limit=&offset=
  GET /v1/transactions/:signature
  GET /v1/transactions/:signature/instructions

  GET /v1/instructions?type=&category=&slot_start=&slot_end=&limit=&offset=

  GET /v1/deposits?depositor=&receiver=&mint=&status=&limit=&offset=
  GET /v1/transfers?sender=&receiver=&mint=&status=&limit=&offset=
  GET /v1/withdrawals?withdrawer=&mint=&status=&limit=&offset=
  GET /v1/utxos?depositor=&mint=&tree_index=&status=&limit=&offset=
  GET /v1/claims?receiver=&relayer=&mint=&target=&status=&limit=&offset=
  GET /v1/conversions?user=&mint=&status=&limit=&offset=

  GET /v1/mpc/:computation_offset       → MPC computation status
  GET /v1/mpc/pending?category=         → all pending computations

  GET /v1/users/:address/activity       → all operations for a user
  GET /v1/users/:address/accounts       → user account events
  GET /v1/mints/:mint/stats             → volume, operation counts

  GET /v1/config/events?type=           → config/admin events
  GET /v1/compliance?receiver=&type=    → compliance grant events
  GET /v1/relayer/:address/fees         → relayer fee events

  GET /v1/stats/global                  → global stats (materialized view)
  GET /v1/stats/daily?mint=             → daily volume (materialized view)

  WS /v1/ws/events?category=            → live event stream via WebSocket

Response format: JSON (with optional Protobuf via Accept header)
```

### 7.4 WebSocket Service (embedded in API or separate)

Push real-time events to subscribers:

- Subscribe by category: `deposits`, `transfers`, `claims`, `utxos`, `mpc_status_changes`
- Subscribe by user address
- Subscribe by mint
- MPC status change notifications (pending → confirmed/timeout)

Implementation: `tokio-tungstenite` with fan-out from PostgreSQL LISTEN/NOTIFY (the write service issues `pg_notify` after each batch write). No extra infrastructure needed — PostgreSQL handles the pub/sub.

---

## 8. Extensibility: Adding a New Instruction

When a new instruction is added to the Umbra program, the indexer needs:

**Step 1: Handler file** (~50 lines)
```rust
// src/handlers/confidential/new_instruction.rs
pub struct NewInstructionHandler;

impl InstructionHandler for NewInstructionHandler {
    fn discriminator(&self) -> [u8; 8] {
        // Anchor discriminator for the new instruction
        [0x12, 0x34, ...]
    }

    fn instruction_name(&self) -> &'static str {
        "new_instruction_v3"
    }

    fn handle(&self, ctx: &InstructionContext) -> Result<Vec<DbOperation>> {
        let event = decode_event::<NewInstructionV3Event>(&ctx.logs)?;
        let accounts = ctx.accounts();

        Ok(vec![
            // Insert into instructions table (automatic)
            DbOperation::InsertInstruction { ... },
            // Insert into the appropriate event table
            DbOperation::InsertDepositEvent { ... },
            // Link to MPC computation if applicable
            DbOperation::UpsertMpcComputation { ... },
        ])
    }
}
```

**Step 2: Register handler** (1 line)
```rust
registry.register(Box::new(NewInstructionHandler));
```

**Step 3: Migration** (if new columns needed)
```sql
ALTER TABLE deposit_events ADD COLUMN new_field TEXT;
```

That's it. The consumer, SQS, and API all work automatically. Only the write service handler registry changes.

---

## 9. Reliability & Fault Tolerance

### 9.1 Crash Recovery

```
Scenario                          Recovery
─────────────────────────────────────────────────────────────────────
Consumer crashes                  ECS restarts it → reads DynamoDB checkpoint → resumes Laserstream
                                  SQS already has all messages since last send — no data loss

Write service crashes             ECS restarts it → undeleted messages become visible after 60s
                                  Multiple instances can poll same queue — automatic load distribution
                                  Idempotent writes handle any re-processing

DB write fails                    Message not deleted from SQS → becomes visible again after 60s
                                  After 3 failed attempts → moved to DLQ automatically

Laserstream disconnects           Auto-reconnect with exponential backoff (built into library)
                                  Consumer is stateless aside from DynamoDB checkpoint — restarts cleanly

RDS goes down                     Write service polls SQS but writes fail → messages stay in queue
                                  SQS has 14-day retention → write service catches up when RDS recovers

SQS goes down                     Consumer buffers in-memory (bounded) + exponential retry
                                  SQS has 99.999999999% durability and is replicated across AZs
                                  Outages are extremely rare (<5 minutes/year historically)

Read replica lag                  Monitor via CloudWatch → alert if > 5s

Corrupted parse                   Write service logs + message moves to DLQ after 3 attempts →
                                  raw_data in DB for re-parse
```

### 9.2 Monitoring & Alerting

CloudWatch metrics (native for AWS services) + Prometheus metrics on `/metrics` for custom app metrics:

**CloudWatch (automatic):**
- SQS `ApproximateNumberOfMessagesVisible` — queue depth (backlog)
- SQS `ApproximateAgeOfOldestMessage` — how stale the oldest message is
- SQS `NumberOfMessagesSent`, `NumberOfMessagesReceived` — throughput
- RDS `ReplicaLag`, `CPUUtilization`, `FreeableMemory`
- DynamoDB `ConsumedReadCapacityUnits`, `ConsumedWriteCapacityUnits`
- ECS task health, restarts, CPU/memory utilization

**Custom Prometheus metrics:**
- `indexer_slot_current` — latest slot processed by write service
- `indexer_slot_lag` — difference between chain head and processed slot
- `indexer_instructions_total{type, category}` — counter per instruction type
- `indexer_mpc_pending_count` — gauge of pending computations
- `indexer_mpc_timeouts_total` — counter of timed-out computations
- `indexer_write_batch_duration_seconds` — histogram of write latency
- `indexer_sqs_messages_processed_total` — counter of SQS messages handled
- `indexer_sqs_dlq_total` — counter of messages sent to DLQ
- `indexer_laserstream_reconnects_total` — counter of reconnections

**Alert conditions (CloudWatch Alarms):**
- SQS `ApproximateNumberOfMessagesVisible > 1000` for > 2 minutes → P1 (write service falling behind)
- SQS `ApproximateAgeOfOldestMessage > 300` (5 min) → P1 (processing stalled)
- SQS DLQ `ApproximateNumberOfMessagesVisible > 0` → P2 (poison messages)
- `indexer_mpc_pending_count > 1000` → P2 (potential Arcium issue)
- RDS `ReplicaLag > 5s` → P2
- ECS task stopped unexpectedly → P1

### 9.3 Backfill Strategy

For historical data or gap recovery:

```
┌──────────────────────────────────────┐
│         Backfill Worker              │
│         (ECS Fargate task)           │
│                                      │
│  Uses Helius getSignaturesForAddress │
│  + getTransaction RPC calls          │
│                                      │
│  Sends to same SQS queue            │
│  (reuses write service pipeline)     │
│                                      │
│  Processes slot ranges in parallel   │
│  (8 concurrent RPC requests)         │
│                                      │
│  Idempotent — safe to overlap with   │
│  live ingestion                      │
│                                      │
│  Run as one-off ECS task, not        │
│  long-running service                │
└──────────────────────────────────────┘
```

---

## 10. Performance Characteristics

### 10.1 Expected Throughput

- **Consumer → SQS**: >10,000 messages/second (SendMessageBatch, 10 msgs/call)
- **Write service**: >5,000 transactions/second (batch writes to RDS)
- **API reads**: >10,000 req/s per replica (indexed RDS PostgreSQL)
- **End-to-end latency**: <500ms from on-chain confirmation to API availability
  - Laserstream: ~200ms, SQS: ~5ms, Parse+Write: ~50ms, Replication: ~100ms

### 10.2 Storage Estimates

Per instruction: ~500 bytes (core tables) + ~2KB (raw_data) ≈ 2.5KB
At 100 Umbra instructions/day: ~250KB/day, ~91MB/year
At 10,000 instructions/day: ~25MB/day, ~9.1GB/year
At 100,000 instructions/day: ~250MB/day, ~91GB/year

RDS PostgreSQL handles this comfortably. gp3 storage auto-scales.

### 10.3 Table Partitioning (for scale)

Partition large tables by slot range when they exceed ~100M rows:

```sql
CREATE TABLE instructions (
    ...
) PARTITION BY RANGE (slot);

CREATE TABLE instructions_p0 PARTITION OF instructions
    FOR VALUES FROM (0) TO (300000000);
CREATE TABLE instructions_p1 PARTITION OF instructions
    FOR VALUES FROM (300000000) TO (600000000);
-- etc.
```

---

## 11. Technology Stack

- **Language**: Rust (consistent with existing crates)
- **Async runtime**: Tokio
- **Web framework**: Axum (consistent with existing services)
- **Database**: RDS PostgreSQL 16 (primary + read replica), Diesel ORM for migrations, sqlx for async queries
- **Message queue**: AWS SQS Standard (+ DLQ)
- **Checkpoint store**: DynamoDB (single-table, single-digit ms)
- **Data source**: Helius Laserstream (gRPC)
- **Serialization**: Borsh (Solana/Anchor), JSON (API), bincode (SQS envelope)
- **AWS SDK**: `aws-sdk-sqs`, `aws-sdk-dynamodb` (official Rust SDK)
- **Metrics**: CloudWatch (native) + prometheus-client (custom)
- **Logging**: tracing + tracing-subscriber → CloudWatch Logs
- **WebSocket**: tokio-tungstenite
- **Container**: Docker multi-stage build → ECR
- **Compute**: ECS Fargate (serverless containers)
- **Config**: TOML + environment variables + AWS Secrets Manager for sensitive values

---

## 12. Crate Structure

```
crates/
├── indexer-framework/              # EXISTING — reuse producer/pipeline traits
│
├── indexer-consumer/               # NEW — ultra-light Laserstream → SQS consumer
│   ├── src/
│   │   ├── main.rs                # Entry point, Laserstream setup
│   │   ├── config.rs              # TOML + env config
│   │   ├── producer.rs            # Laserstream producer (reuses indexer-framework)
│   │   ├── sqs.rs                 # SQS batch sender (SendMessageBatch)
│   │   ├── checkpoint.rs          # DynamoDB checkpoint read/write
│   │   └── metrics.rs             # Prometheus metrics
│   ├── config.toml
│   ├── Dockerfile
│   └── Cargo.toml
│
├── indexer-write-service/          # NEW — SQS → Parse → PostgreSQL write service
│   ├── src/
│   │   ├── main.rs                # Entry point, SQS polling loop
│   │   ├── config.rs              # TOML + env config
│   │   ├── sqs_poller.rs          # SQS long-polling + batch receive
│   │   ├── parser.rs              # Transaction parsing, discriminator routing
│   │   ├── registry.rs            # Handler registry
│   │   ├── writer.rs              # Batch DB writer (PostgreSQL)
│   │   ├── mpc_worker.rs          # MPC timeout background task
│   │   ├── metrics.rs             # Prometheus metrics
│   │   ├── handlers/
│   │   │   ├── mod.rs
│   │   │   ├── common.rs          # Shared handler utilities
│   │   │   ├── deposits.rs        # All 4 deposit variants
│   │   │   ├── transfers.rs       # All 4 transfer variants
│   │   │   ├── withdrawals.rs     # Withdrawal handler
│   │   │   ├── utxo_creation.rs   # Confidential + public UTXO creation
│   │   │   ├── claims.rs          # All 5 claim variants
│   │   │   ├── conversions.rs     # MXE→Shared, re-encrypt
│   │   │   ├── user_accounts.rs   # Init, update, register, seed updates
│   │   │   ├── compliance.rs      # All grant create/delete variants
│   │   │   ├── config.rs          # Program init, pools, ZK keys, fees
│   │   │   ├── relayer.rs         # Relayer accounts, fees, collection
│   │   │   ├── mixer.rs           # Tree init, treap init, tree update
│   │   │   ├── callbacks.rs       # All callback handlers (match to queue)
│   │   │   └── proof_accounts.rs  # Proof account create/close
│   │   └── backfill.rs            # Historical backfill (pushes to SQS)
│   ├── migrations/                # Diesel migrations
│   ├── config.toml
│   ├── Dockerfile
│   └── Cargo.toml
│
├── indexer-api/                    # NEW — read API binary
│   ├── src/
│   │   ├── main.rs
│   │   ├── config.rs
│   │   ├── routes/
│   │   │   ├── transactions.rs
│   │   │   ├── instructions.rs
│   │   │   ├── deposits.rs
│   │   │   ├── transfers.rs
│   │   │   ├── withdrawals.rs
│   │   │   ├── utxos.rs
│   │   │   ├── claims.rs
│   │   │   ├── conversions.rs
│   │   │   ├── mpc.rs
│   │   │   ├── users.rs
│   │   │   ├── compliance.rs
│   │   │   ├── config_events.rs
│   │   │   ├── relayer.rs
│   │   │   ├── stats.rs
│   │   │   └── websocket.rs
│   │   ├── models.rs              # Response types
│   │   ├── pagination.rs          # Cursor/offset pagination
│   │   └── metrics.rs
│   ├── Cargo.toml
│   └── Dockerfile
│
├── indexer-common/                 # NEW — shared DB models, migrations, types
│   ├── src/
│   │   ├── lib.rs
│   │   ├── schema.rs              # Diesel schema
│   │   ├── models/                # DB models per table
│   │   ├── queries/               # Reusable query builders
│   │   ├── sqs_types.rs           # SqsTransactionEnvelope (shared between consumer + write)
│   │   └── types.rs               # Shared types
│   ├── migrations/
│   └── Cargo.toml
```

---

## 13. AWS Deployment Topology

```
Production:
─────────────────────────────────────────────────────────────
  ECS Fargate:
    1x  Consumer Service     (0.5 vCPU, 1GB RAM)    ← always exactly 1
    1-3x Write Service       (2 vCPU, 4GB RAM)       ← auto-scale on SQS backlog
    2x  API Service          (1 vCPU, 2GB RAM)       ← behind ALB, auto-scale on CPU

  SQS:
    1x  Standard Queue       (umbra-indexer-txs)
    1x  Dead-letter Queue    (umbra-indexer-txs-dlq)

  RDS PostgreSQL 16:
    1x  Primary              (db.r6g.xlarge — 4 vCPU, 32GB RAM, gp3 SSD)
    1x  Read Replica         (db.r6g.large — 2 vCPU, 16GB RAM)

  DynamoDB:
    1x  Table                (indexer-checkpoints, on-demand capacity)

  ECR:
    3x  Repositories         (consumer, write-service, api)

Staging:
─────────────────────────────────────────────────────────────
  ECS Fargate:
    1x  Consumer             (0.25 vCPU, 0.5GB RAM)
    1x  Write Service        (1 vCPU, 2GB RAM)
    1x  API Service          (0.5 vCPU, 1GB RAM)

  SQS: Same (SQS is free up to 1M requests/month)

  RDS: db.t4g.medium (2 vCPU, 4GB RAM), no replica
```

**Auto-scaling triggers:**
- Write service: Scale up when SQS `ApproximateNumberOfMessagesVisible > 1000` for 2 minutes
- API service: Scale up when ALB target response time > 200ms or CPU > 70%

---

## 14. Migration Path from Existing Indexer

The existing `utxo-indexer` only indexes mixer/UTXO operations. The comprehensive indexer:

- **Replaces**: utxo-indexer (its UTXO data is a subset of the comprehensive indexer)
- **Coexists with**: notification pipeline (different concern — push notifications vs. data storage)
- **Reuses**: `indexer-framework` crate (producer/pipeline traits for the consumer)
- **Reuses**: Helius Laserstream infrastructure and API keys

The existing `utxo_data` and `merkle_nodes` tables can be migrated into the new schema, or the comprehensive indexer can backfill from genesis using the backfill worker (pushes historical TXs to the same SQS queue).

---

## 15. Summary of Key Decisions

- **Consumer / Write Service split** — The Laserstream consumer is ultra-light (no parsing, no DB). SQS decouples it from all downstream concerns. The write service does the heavy lifting and can crash/restart/scale independently.
- **SQS Standard over FIFO** — Idempotent writes handle at-least-once delivery naturally, avoiding FIFO's 3K msg/s throughput limit. SQS Standard is fully managed with zero ops, 14-day retention, built-in DLQ, and 99.999999999% durability. No infrastructure to manage.
- **RDS PostgreSQL over GraphQL/Subgraph** — Direct SQL gives maximum query flexibility, lower operational complexity, and better performance. A GraphQL layer can be added on top later (e.g., with Hasura or PostGraphile for zero-code GraphQL from PostgreSQL).
- **Handler registry** — Maximum extensibility with minimum boilerplate per instruction. New instruction = new handler file + register line + optional migration.
- **Raw data storage** — Future-proofs against schema changes. Can always re-parse from `raw_data` column.
- **RDS read replica** — Separates read/write load. API service never impacts write service throughput.
- **ECS Fargate** — Serverless containers, no EC2 instances to manage. Auto-scaling on SQS backlog and ALB metrics.
- **DynamoDB for consumer checkpoint** — The consumer has zero PostgreSQL dependency. DynamoDB gives single-digit ms checkpoint reads/writes with zero ops.
