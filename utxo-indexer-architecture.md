# UTXO Indexer — High-Performance Architecture Plan

## Executive Summary

A redesigned UTXO indexer that replaces direct HTTP/HMAC transport with a **dual-queue architecture**: an in-process DropOldest buffer for burst absorption and priority, plus AWS SQS Standard for durable inter-service communication. All parsing, ordering, and Merkle tree logic moves to the write service, making the consumer ultra-light. The write service handles out-of-order SQS delivery with an ordering buffer and existing gap-backfill logic.

---

## 1. High-Level Architecture

```
  ┌─────────────────────┐
  │   Helius Laserstream │
  │   (gRPC stream)      │
  └──────────┬──────────┘
             │
  ┌──────────▼──────────────────────────────────────────┐
  │   Consumer Service (single Rust binary)              │
  │                                                      │
  │   ┌──────────────┐    ┌──────────────┐    ┌───────┐ │
  │   │ Laserstream   │    │ DropOldest   │    │ Disc. │ │
  │   │ Receiver Task │───▶│ Buffer       │───▶│ Filter│ │
  │   │               │    │ (10K cap)    │    │ (8B)  │ │
  │   └──────────────┘    └──────────────┘    └───┬───┘ │
  │                                               │     │
  │                        ┌──────────────┐       │     │
  │                        │ SQS Batch    │◀──────┘     │
  │                        │ Sender Task  │             │
  │                        │ (10 msg/call)│             │
  │                        └──────┬───────┘             │
  │                               │                     │
  │   Checkpoint: DynamoDB        │                     │
  └───────────────────────────────┼─────────────────────┘
                                  │
  ┌───────────────────────────────▼─────────────────────┐
  │   AWS SQS Standard                                   │
  │                                                      │
  │   Queue: utxo-indexer-txs                            │
  │   DLQ:   utxo-indexer-txs-dlq (after 3 attempts)    │
  │   Retention: 14 days                                 │
  │   Visibility timeout: 120 seconds                    │
  └───────────────────────────────┬─────────────────────┘
                                  │
  ┌───────────────────────────────▼─────────────────────┐
  │   Write Service (single Rust binary)                 │
  │                                                      │
  │   ┌──────────────┐    ┌──────────────┐              │
  │   │ SQS Long-    │    │ TX Parser    │              │
  │   │ Poller       │───▶│ (full borsh  │              │
  │   │ (20s wait)   │    │  decode)     │              │
  │   └──────────────┘    └──────┬───────┘              │
  │                              │                      │
  │   ┌──────────────────────────▼───────────────┐      │
  │   │ Ordering Buffer                           │      │
  │   │                                           │      │
  │   │  - Buffers out-of-order insertion events  │      │
  │   │  - Sorts by (tree_index, insertion_index) │      │
  │   │  - Flushes sequentially when contiguous   │      │
  │   │  - Gap detection → backfill trigger       │      │
  │   └──────────────────────────┬───────────────┘      │
  │                              │                      │
  │   ┌──────────────────────────▼───────────────┐      │
  │   │ Merkle Tree Inserter                      │      │
  │   │                                           │      │
  │   │  - Sequential leaf insertion              │      │
  │   │  - Per-tree DashMap<Mutex> locking        │      │
  │   │  - Atomic PostgreSQL writes               │      │
  │   │  - UTXO data storage                      │      │
  │   └──────────────────────────────────────────┘      │
  │                                                      │
  │   Background: Gap Backfill Worker (Helius RPC)       │
  │   Checkpoint: DynamoDB (last processed insertion)     │
  └───────────────────────────────┬─────────────────────┘
                                  │
  ┌───────────────────────────────▼─────────────────────┐
  │   PostgreSQL 16 (RDS)                                │
  │                                                      │
  │   - merkle_nodes     (tree state)                    │
  │   - merkle_metadata  (leaf counts)                   │
  │   - utxo_data        (UTXO details)                  │
  └───────────────────────────────┬─────────────────────┘
                                  │ WAL replication
  ┌───────────────────────────────▼─────────────────────┐
  │   PostgreSQL 16 (Read Replica)                       │
  └───────────────────────────────┬─────────────────────┘
                                  │
  ┌───────────────────────────────▼─────────────────────┐
  │   Read Service (Axum REST API)                       │
  │                                                      │
  │   - GET /v1/trees/:index/proof/:insertion_index      │
  │   - GET /v1/utxos?start=&limit=                      │
  │   - GET /v1/stats                                    │
  │   - WS  /v1/ws/insertions (live)                     │
  └─────────────────────────────────────────────────────┘
```

---

## 2. Design Principles

**P1: Consumer is ultra-light** — The consumer does three things: receive raw bytes from Laserstream, check 8-byte discriminators on log messages, and send matching TXs to SQS. No borsh decoding, no event extraction, no DB writes. The only "parsing" is the discriminator check — a memcmp on base64-decoded log prefixes (~microseconds).

**P2: Dual-queue for burst absorption + durability** — The in-process DropOldest buffer (10K cap) absorbs Laserstream burst traffic. If the SQS sender falls behind, the buffer drops the oldest messages (preserving the newest/freshest). SQS provides durable inter-service delivery with 14-day retention.

**P3: Write service owns all parsing and ordering** — Full transaction parsing (borsh decode, event extraction, Merkle tree insertion) happens in the write service. This means the consumer can be restarted, redeployed, or even replaced without touching any business logic.

**P4: Ordering is handled in the write service** — SQS Standard does not guarantee ordering. The write service maintains an ordering buffer that collects insertion events and flushes them sequentially by `(tree_index, insertion_index)`. Gaps trigger backfill from Helius RPC. This is an evolution of the existing gap-detection logic.

**P5: Idempotent writes** — Every Merkle tree insertion checks `insertion_index == current_leaf_count` before inserting. Duplicate or already-processed insertions are skipped. Safe to replay any SQS message.

**P6: No HTTP/HMAC between services** — The processor → write-service HTTP transport with HMAC signing is eliminated. SQS replaces it entirely. This removes reqwest, HMAC key management, and HTTP error handling from the processor, and removes the HTTP server + signature verification from the write service.

---

## 3. Consumer Service

### 3.1 Laserstream Receiver

Reuses the existing `indexer-framework` `LaserStreamProducer` pattern:

```rust
LaserStreamConfig {
    program_ids: vec![UMBRA_PROGRAM_ID],
    commitment: CommitmentLevel::Confirmed,
    // No instruction filter — capture all Umbra TXs
}
```

The receiver task pushes raw `SolanaTransaction` objects into the DropOldest buffer via a non-blocking emit closure. The producer never blocks — if the buffer is full, the oldest entry is dropped.

### 3.2 DropOldest Internal Buffer

Same proven mechanism from the current indexer, with the same parameters:

```
Buffer: VecDeque<SolanaTransaction>  (Arc<Mutex>)
Capacity: 10,000 items
Strategy: DropOldest (newest preserved on overflow)
Metrics: emitted count, dropped count (atomic counters)

Drainer task:
  - Continuously pop_front() from buffer
  - Send to discriminator filter channel (unbounded mpsc)
  - Tracks throughput every 100 items
```

**Why keep the internal buffer:**
- Laserstream can burst during slot catch-up or network recovery
- The buffer absorbs burst without applying backpressure to the gRPC stream
- DropOldest ensures the freshest transactions always survive (stale data is less valuable)
- Without the buffer, a slow SQS sender would cause the Laserstream gRPC stream to stall

### 3.3 Discriminator Filter

The lightest possible filtering — reuses the existing `has_insert_events()` logic:

```rust
// ~microseconds per TX — no borsh, no allocation, just byte comparison
fn has_insert_events(tx: &SolanaTransaction) -> bool {
    for log_msg in &tx.meta.log_messages {
        if let Some(data) = log_msg.strip_prefix("Program data: ") {
            if let Ok(decoded) = base64::decode(data) {
                if decoded.len() >= 8 {
                    let disc = &decoded[..8];
                    if disc == DEPOSIT_EVENT_DISCRIMINATOR
                        || disc == CALLBACK_EVENT_DISCRIMINATOR
                    {
                        return true;
                    }
                }
            }
        }
    }
    false
}
```

This filters out ~90% of Umbra transactions that are not UTXO-related (deposits to encrypted balances, transfers, config operations, etc.), reducing SQS message volume significantly.

### 3.4 SQS Batch Sender

```rust
// Sends up to 10 messages per SendMessageBatch call
// Flushes every 200ms or when 10 messages are buffered
struct SqsBatchSender {
    client: aws_sdk_sqs::Client,
    queue_url: String,
    batch: Vec<SendMessageBatchRequestEntry>,  // max 10
    last_flush: Instant,
}

// Message body: bincode-serialized envelope → base64
struct SqsUtxoEnvelope {
    signature: String,
    slot: u64,
    block_time: Option<i64>,
    raw_data: Vec<u8>,       // raw transaction bytes
    log_messages: Vec<String>, // transaction log messages (needed for event parsing)
}
```

**SQS message attributes** (for routing/filtering if needed later):
- `slot` (Number): Solana slot number
- `signature` (String): Transaction signature

### 3.5 Checkpoint Tracking

```rust
// DynamoDB table: indexer-checkpoints
// Key: { id: "utxo-consumer" }
struct ConsumerCheckpoint {
    id: String,
    last_processed_slot: u64,
    last_signature: String,
    messages_sent: u64,       // total SQS messages sent (lifetime)
    updated_at: u64,
}
// Written every 100 messages or every 10 seconds (whichever comes first)
```

On restart: read checkpoint from DynamoDB → resume Laserstream from `last_processed_slot`. Overlap is fine — idempotent writes downstream.

---

## 4. Write Service

The write service is the single brain of the UTXO indexer. It owns ALL parsing, ordering, Merkle tree logic, and database writes.

### 4.1 SQS Long-Poller

```rust
loop {
    let response = sqs.receive_message()
        .queue_url(&queue_url)
        .max_number_of_messages(10)
        .wait_time_seconds(20)         // long-poll (reduces API calls to near-zero when idle)
        .visibility_timeout(120)       // 2 min to process (Merkle insertion can be slow)
        .send()
        .await?;

    if let Some(messages) = response.messages {
        let results = process_batch(&messages).await;

        // Delete only successfully processed messages
        let successful: Vec<_> = results.iter()
            .filter(|r| r.is_ok())
            .map(|r| r.receipt_handle())
            .collect();

        if !successful.is_empty() {
            sqs.delete_message_batch()
                .queue_url(&queue_url)
                .set_entries(Some(delete_entries))
                .send()
                .await?;
        }

        // Failed messages stay in queue → become visible after visibility timeout
        // After 3 failures → SQS moves to DLQ automatically
    }
}
```

### 4.2 Full Transaction Parser

All parsing that was previously split between the processor and write service now lives here:

```rust
fn parse_insertion_event(envelope: &SqsUtxoEnvelope) -> Result<Vec<InsertionEvent>> {
    let mut events = Vec::new();

    for log_msg in &envelope.log_messages {
        if let Some(data) = log_msg.strip_prefix("Program data: ") {
            let decoded = base64::decode(data)?;
            if decoded.len() < 8 { continue; }

            let disc = &decoded[..8];
            if disc == DEPOSIT_EVENT_DISCRIMINATOR {
                events.push(parse_deposit_event(&decoded[8..], &envelope)?);
            } else if disc == CALLBACK_EVENT_DISCRIMINATOR {
                events.push(parse_callback_event(&decoded[8..], &envelope)?);
            }
        }
    }

    Ok(events)
}
```

The parser extracts the full `InsertionEvent` struct (tree_index, insertion_index, h1 fields, h2 hash, final_commitment, AES data, depositor key, etc.) — exactly what the current write service does after receiving the HTTP request.

### 4.3 Ordering Buffer

This is the key component that handles SQS Standard's lack of ordering guarantees. The Merkle tree requires sequential insertion (insertion_index N must be inserted before N+1).

```rust
struct OrderingBuffer {
    // Per-tree buffer of parsed events, keyed by insertion_index
    pending: HashMap<u64, BTreeMap<u64, InsertionEvent>>,
    // tree_index → BTreeMap<insertion_index, event>

    // Per-tree: the next expected insertion_index
    next_expected: HashMap<u64, u64>,  // tree_index → next insertion_index

    // Max time to wait for a gap before triggering backfill
    gap_timeout: Duration,  // e.g. 30 seconds

    // Track when each gap was first detected
    gap_detected_at: HashMap<(u64, u64), Instant>,  // (tree_index, gap_start) → when detected
}

impl OrderingBuffer {
    /// Add a parsed event to the buffer. Returns events ready for sequential insertion.
    fn insert(&mut self, event: InsertionEvent) -> Vec<InsertionEvent> {
        let tree = event.tree_index;
        let idx = event.insertion_index;

        self.pending
            .entry(tree)
            .or_default()
            .insert(idx, event);

        // Drain contiguous events starting from next_expected
        self.drain_contiguous(tree)
    }

    /// Drain all contiguous events starting from next_expected[tree]
    fn drain_contiguous(&mut self, tree: u64) -> Vec<InsertionEvent> {
        let next = self.next_expected.entry(tree).or_insert(0);
        let mut ready = Vec::new();

        if let Some(tree_buf) = self.pending.get_mut(&tree) {
            while let Some(event) = tree_buf.remove(next) {
                ready.push(event);
                *next += 1;
            }
        }

        ready
    }

    /// Check for gaps that have exceeded the timeout → trigger backfill
    fn check_gap_timeouts(&mut self) -> Vec<GapRange> {
        let mut gaps = Vec::new();
        let now = Instant::now();

        for (&tree, buf) in &self.pending {
            let next = *self.next_expected.get(&tree).unwrap_or(&0);
            if buf.is_empty() { continue; }

            let first_buffered = *buf.keys().next().unwrap();
            if first_buffered > next {
                // There's a gap: next..first_buffered
                let gap_key = (tree, next);
                let detected = self.gap_detected_at
                    .entry(gap_key)
                    .or_insert(now);

                if now.duration_since(*detected) > self.gap_timeout {
                    gaps.push(GapRange {
                        tree_index: tree,
                        from_index: next,
                        to_index: first_buffered,
                    });
                }
            }
        }

        gaps
    }
}
```

**How it works:**
1. SQS messages arrive in arbitrary order
2. Each parsed InsertionEvent is added to the per-tree BTreeMap (sorted by insertion_index)
3. After each insert, `drain_contiguous()` checks if there's a run of sequential events starting from `next_expected`
4. If yes: those events are returned for immediate Merkle tree insertion
5. If there's a gap: the buffer holds the events until the missing ones arrive (from SQS or backfill)
6. If a gap persists for >30 seconds: trigger backfill from Helius RPC

**Example:**
```
next_expected = 100
Received: [103, 100, 105, 101, 104, 102]

After 100 arrives: drain 100 → next_expected = 101
After 101 arrives: drain 101 → next_expected = 102
After 102 arrives: drain 102, 103, 104, 105 → next_expected = 106
```

### 4.4 Gap Backfill Worker

Background task that fills gaps when the ordering buffer detects missing insertion indices:

```rust
async fn backfill_gap(gap: GapRange, helius: &HeliusClient, sqs_url: &str) {
    // 1. Fetch transaction signatures for the Umbra program around the missing slots
    let sigs = helius.get_signatures_for_address(
        &UMBRA_PROGRAM_ID,
        // Use slot range estimation from surrounding known events
    ).await?;

    // 2. Fetch full transactions
    let txs = helius.get_transactions(&sigs).await?;

    // 3. Parse and filter for the missing insertion indices
    let missing: Vec<InsertionEvent> = txs.iter()
        .filter_map(|tx| parse_insertion_event(tx).ok())
        .flatten()
        .filter(|e| e.tree_index == gap.tree_index
                  && e.insertion_index >= gap.from_index
                  && e.insertion_index < gap.to_index)
        .collect();

    // 4. Feed directly into the ordering buffer (bypass SQS for backfill)
    for event in missing {
        ordering_buffer.insert(event);
    }
}
```

The backfill worker runs as a background Tokio task. It only activates when the ordering buffer reports gaps that have exceeded the timeout threshold. Backfilled events go directly into the ordering buffer (not back through SQS) to avoid circular message flow.

### 4.5 Merkle Tree Inserter

Unchanged from the current implementation — sequential leaf insertion with per-tree locking:

```rust
struct MerkleTreeStorage {
    pool: DbPool,
    trees: Arc<DashMap<u64, Arc<Mutex<MerkleTree>>>>,
}

impl MerkleTreeStorage {
    async fn insert_leaf(&self, event: &InsertionEvent) -> Result<InsertResponse> {
        let tree_lock = self.trees
            .entry(event.tree_index)
            .or_insert_with(|| Arc::new(Mutex::new(
                MerkleTree::load_from_db(&self.pool, event.tree_index)
            )));

        let mut tree = tree_lock.lock().await;

        // Idempotent: skip if already inserted
        if event.insertion_index < tree.leaf_count() {
            return Ok(InsertResponse::AlreadyExists);
        }

        // Verify sequential: must be exactly the next expected leaf
        assert_eq!(event.insertion_index, tree.leaf_count());

        // Insert leaf and update PostgreSQL atomically
        let result = tree.insert_leaf(event.final_commitment)?;
        store_utxo_data(&self.pool, event)?;

        Ok(InsertResponse::Inserted {
            tree_index: event.tree_index,
            insertion_index: event.insertion_index,
            new_root: result.root,
        })
    }
}
```

### 4.6 Write Service Checkpoint

```rust
// DynamoDB table: indexer-checkpoints
// Key: { id: "utxo-write-service" }
struct WriteServiceCheckpoint {
    id: String,
    trees: HashMap<u64, TreeCheckpoint>,  // per-tree progress
    total_inserted: u64,
    updated_at: u64,
}

struct TreeCheckpoint {
    tree_index: u64,
    last_insertion_index: u64,
    leaf_count: u64,
}
// Written after every successful batch of Merkle insertions
```

On restart: read checkpoint → set `next_expected` for each tree in the ordering buffer → resume SQS polling.

---

## 5. Read Service

Largely unchanged from current implementation. Connects to the RDS read replica instead of primary.

```
Endpoints:

  GET /v1/trees/:tree_index                        → tree metadata (leaf count, root)
  GET /v1/trees/:tree_index/proof/:insertion_index  → Merkle inclusion proof
  GET /v1/trees/:tree_index/root                    → current root hash

  GET /v1/utxos?start=&limit=                       → paginated UTXOs (by absolute_index)
  GET /v1/utxos/:absolute_index                      → single UTXO
  GET /v1/trees/:tree_index/utxos?start=&limit=      → UTXOs by tree

  GET /v1/stats                                      → aggregate statistics

  GET /health                                        → basic health
  GET /health/detailed                               → db connectivity, replication lag
  GET /health/liveness                               → process alive
  GET /health/readiness                              → ready to serve

  WS /v1/ws/insertions                               → live insertion stream

Middleware:
  - Rate limiting (token bucket)
  - GZIP/Brotli compression
  - Request logging with trace IDs
```

**WebSocket live stream** (new): The write service issues `pg_notify('utxo_insertions', ...)` after each Merkle insertion. The read service listens and fans out to WebSocket subscribers. This replaces polling for clients that need real-time data.

---

## 6. Database Schema

### 6.1 Existing Tables (unchanged)

```sql
-- Merkle tree node storage (incremental tree)
CREATE TABLE merkle_nodes (
    tree_index  BIGINT NOT NULL,
    level       INTEGER NOT NULL,
    idx         BIGINT NOT NULL,
    node        BYTEA NOT NULL,          -- 32-byte Poseidon hash
    PRIMARY KEY (tree_index, level, idx)
);

-- Tree metadata (leaf counts, etc.)
CREATE TABLE merkle_metadata (
    tree_index  BIGINT NOT NULL,
    key         TEXT NOT NULL,
    value       BYTEA NOT NULL,
    PRIMARY KEY (tree_index, key)
);

-- UTXO data per insertion
CREATE TABLE utxo_data (
    tree_index                      BIGINT NOT NULL,
    insertion_index                 BIGINT NOT NULL,
    final_commitment                BYTEA NOT NULL,      -- 32-byte Poseidon hash
    h1_version                      BYTEA NOT NULL,      -- u128 LE
    h1_commitment_index             BYTEA NOT NULL,      -- u128 LE
    h1_sender_address               BYTEA NOT NULL,      -- 32-byte
    h1_mint_address                 BYTEA NOT NULL,      -- 32-byte
    h1_relayer_fixed_sol_fees       BIGINT NOT NULL,
    h1_year                         INTEGER NOT NULL,
    h1_month                        SMALLINT NOT NULL,
    h1_day                          SMALLINT NOT NULL,
    h1_hour                         SMALLINT NOT NULL,
    h1_minute                       SMALLINT NOT NULL,
    h1_second                       SMALLINT NOT NULL,
    h1_pool_volume_spl              BIGINT NOT NULL,
    h1_pool_volume_sol              BIGINT NOT NULL,
    h1_purpose                      INTEGER NOT NULL,
    h1_circuit_provable_hash        BYTEA NOT NULL,      -- 32-byte
    h1_smart_program_provable_hash  BYTEA NOT NULL,      -- 32-byte
    h1_hash                         BYTEA NOT NULL,      -- 32-byte
    h2_hash                         BYTEA NOT NULL,      -- 32-byte
    aes_encrypted_data              BYTEA NOT NULL,      -- 96-byte AES-GCM
    timestamp                       BIGINT NOT NULL,     -- Unix seconds
    slot                            BIGINT NOT NULL,     -- Solana slot
    event_type                      TEXT NOT NULL,        -- 'deposit' | 'callback'
    depositor_x25519_public_key     BYTEA NOT NULL,      -- 32-byte
    PRIMARY KEY (tree_index, insertion_index)
);

CREATE INDEX idx_utxo_data_slot ON utxo_data (slot);
CREATE INDEX idx_utxo_data_commitment ON utxo_data (final_commitment);
CREATE INDEX idx_utxo_data_sender ON utxo_data (h1_sender_address);
CREATE INDEX idx_utxo_data_mint ON utxo_data (h1_mint_address);
```

### 6.2 New Table: Ordering Buffer State (crash recovery)

```sql
-- Persisted ordering buffer state for crash recovery
-- On startup, load pending events and resume ordering
CREATE TABLE ordering_buffer_pending (
    tree_index       BIGINT NOT NULL,
    insertion_index  BIGINT NOT NULL,
    raw_event        BYTEA NOT NULL,       -- bincode-serialized InsertionEvent
    received_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (tree_index, insertion_index)
);
-- Cleaned up after events are flushed to Merkle tree
```

This table ensures that if the write service crashes while holding out-of-order events in the ordering buffer, those events aren't lost. On startup, the write service loads any pending events from this table into the in-memory ordering buffer.

---

## 7. Reliability & Fault Tolerance

### 7.1 Crash Recovery

```
Scenario                          Recovery
─────────────────────────────────────────────────────────────────────
Consumer crashes                  ECS restarts → read DynamoDB checkpoint → resume Laserstream
                                  SQS already has all sent messages — no data loss
                                  DropOldest buffer is lost (acceptable — Laserstream replays)

Write service crashes             ECS restarts → load ordering_buffer_pending from DB
                                  Undeleted SQS messages become visible after 120s
                                  next_expected loaded from DynamoDB checkpoint
                                  Idempotent Merkle insertion handles replays

SQS goes down                     Consumer buffers in DropOldest (10K cap)
                                  SQS is multi-AZ with 99.999999999% durability
                                  Outages extremely rare (<5 min/year)

RDS goes down                     Write service polls SQS but insertions fail
                                  Messages stay in SQS (14-day retention)
                                  Catches up when RDS recovers

Gap in insertion indices           Ordering buffer holds out-of-order events
                                  After 30s timeout → backfill worker fetches from Helius RPC
                                  Backfilled events go directly into ordering buffer

Poison message                    After 3 SQS receive attempts → moved to DLQ
                                  Alert fires on DLQ depth > 0
                                  Manual inspection via DLQ redrive
```

### 7.2 Monitoring & Alerting

**CloudWatch (automatic):**
- SQS `ApproximateNumberOfMessagesVisible` — queue depth
- SQS `ApproximateAgeOfOldestMessage` — staleness
- SQS DLQ `ApproximateNumberOfMessagesVisible` — poison messages
- RDS `ReplicaLag`, `CPUUtilization`, `FreeableMemory`
- ECS task health, restarts, CPU/memory

**Custom Prometheus metrics:**
- `utxo_consumer_emitted_total` — TXs received from Laserstream
- `utxo_consumer_dropped_total` — TXs dropped by DropOldest buffer
- `utxo_consumer_filtered_total` — TXs that passed discriminator filter
- `utxo_consumer_sqs_sent_total` — messages sent to SQS
- `utxo_write_parsed_total` — events parsed from SQS messages
- `utxo_write_inserted_total` — Merkle tree insertions completed
- `utxo_write_ordering_buffer_size` — current size of ordering buffer
- `utxo_write_gaps_detected_total` — gaps detected in insertion sequence
- `utxo_write_backfills_triggered_total` — backfill runs initiated
- `utxo_write_backfills_completed_total` — backfill runs completed
- `utxo_write_insertion_latency_seconds` — histogram of per-insertion time
- `utxo_read_proof_latency_seconds` — histogram of proof generation time

**Alert conditions:**
- SQS `ApproximateNumberOfMessagesVisible > 500` for > 2 min → P1
- SQS `ApproximateAgeOfOldestMessage > 60` (seconds) → P1
- SQS DLQ depth > 0 → P2
- `utxo_write_ordering_buffer_size > 1000` → P2 (ordering buffer growing — possible persistent gap)
- `utxo_consumer_dropped_total` increasing → P3 (burst overflow — may need buffer size increase)
- RDS `ReplicaLag > 5s` → P2
- ECS task stopped → P1

---

## 8. Performance Characteristics

### 8.1 Throughput

- **Consumer → SQS**: >5,000 filtered messages/second (discriminator check is ~μs, SQS batch send is ~5ms per 10 messages)
- **Write service parse**: >10,000 events/second (borsh decode is fast, no I/O)
- **Merkle tree insertion**: ~1,000-2,000 insertions/second per tree (bottleneck: PostgreSQL writes for tree nodes)
- **Read service proofs**: >5,000 proofs/second (indexed PostgreSQL reads, most data cached)

### 8.2 Latency (End-to-End)

```
Laserstream receive:    ~200ms (from on-chain confirmation)
Discriminator filter:   ~10μs
DropOldest buffer:      ~1ms  (drain latency)
SQS send + receive:     ~10-20ms (same-region Standard queue)
Full parse:             ~100μs (borsh decode)
Ordering buffer:        ~0ms  (if in-order) to ~30s (if gap → backfill)
Merkle tree insertion:  ~5-10ms (PostgreSQL write)
WAL replication:        ~50-100ms (to read replica)
────────────────────────────────────────────────
Total (no gap):         ~250-350ms
Total (with gap):       ~30s worst case (backfill)
```

### 8.3 Compared to Current Architecture

```
Metric                   Current (HTTP)        New (SQS)
─────────────────────────────────────────────────────────
Consumer complexity      Medium (filter+parse)  Low (filter only)
Transport durability     None (fire-and-forget) 14-day SQS retention
Crash recovery           Re-process from LS     SQS + ordering buffer
Horizontal scaling       Not possible           Multiple write instances*
Max throughput           ~500 inserts/sec        ~2,000 inserts/sec
Backpressure handling    DropOldest only        DropOldest + SQS buffer
HMAC key management      Required               Eliminated
HTTP server in write svc Required               Eliminated

* Multiple write instances require coordination on per-tree ordering.
  Single instance recommended unless throughput demands it.
```

---

## 9. Technology Stack

- **Language**: Rust
- **Async runtime**: Tokio
- **Web framework**: Axum (read service)
- **Database**: RDS PostgreSQL 16 (primary + read replica), Diesel ORM + r2d2 pool
- **Message queue**: AWS SQS Standard (+ DLQ)
- **Checkpoint store**: DynamoDB (on-demand capacity)
- **Data source**: Helius Laserstream (gRPC via indexer-framework)
- **Serialization**: Borsh (events), bincode (SQS envelope), JSON (API)
- **AWS SDK**: `aws-sdk-sqs`, `aws-sdk-dynamodb`
- **Metrics**: CloudWatch (native) + prometheus-client
- **Logging**: tracing + tracing-subscriber → CloudWatch Logs
- **Merkle tree**: rs-merkle-tree (existing crate, PostgreSQL-backed)
- **Container**: Docker multi-stage build → ECR
- **Compute**: ECS Fargate

---

## 10. Crate Structure

```
crates/
├── indexer-framework/                  # EXISTING — reuse producer/pipeline/DropOldest
│
├── utxo-indexer/                       # MODIFIED — consumer binary (slimmed down)
│   ├── src/
│   │   ├── main.rs                    # Entry point: Laserstream + DropOldest + SQS sender
│   │   ├── config.rs                  # TOML + env config
│   │   ├── producer.rs                # MixerProducer (unchanged — Laserstream receiver)
│   │   ├── filter.rs                  # Discriminator filter (has_insert_events, moved here)
│   │   ├── sqs_sender.rs             # NEW — SQS batch sender task
│   │   ├── checkpoint.rs             # NEW — DynamoDB checkpoint read/write
│   │   └── metrics.rs                # Prometheus metrics
│   ├── config.toml
│   ├── Dockerfile
│   └── Cargo.toml
│
├── utxo-indexer-write-service/         # MODIFIED — SQS poller + full parser + Merkle inserter
│   ├── src/
│   │   ├── main.rs                    # Entry point: SQS poller + ordering + Merkle insertion
│   │   ├── config.rs
│   │   ├── sqs_poller.rs             # NEW — SQS long-polling + batch receive
│   │   ├── parser.rs                 # NEW — full TX parsing (borsh, event extraction)
│   │   ├── ordering_buffer.rs        # NEW — per-tree ordering with gap detection
│   │   ├── backfill.rs               # MOVED — gap backfill worker (was in handler)
│   │   ├── inserter.rs               # EXISTING — Merkle tree insertion logic (from handler)
│   │   ├── checkpoint.rs             # NEW — DynamoDB checkpoint for write service
│   │   ├── metrics.rs
│   │   └── storage/                   # EXISTING — Merkle tree + UTXO PostgreSQL storage
│   │       ├── mod.rs
│   │       ├── merkle.rs
│   │       └── utxo.rs
│   ├── migrations/                    # Diesel migrations (add ordering_buffer_pending)
│   ├── config.toml
│   ├── Dockerfile
│   └── Cargo.toml
│
├── utxo-indexer-read-service/          # EXISTING — mostly unchanged
│   ├── src/
│   │   ├── main.rs
│   │   ├── routes/
│   │   │   ├── trees.rs
│   │   │   ├── utxos.rs
│   │   │   ├── stats.rs
│   │   │   ├── health.rs
│   │   │   └── websocket.rs          # NEW — live insertion stream
│   │   ├── models.rs
│   │   └── metrics.rs
│   ├── Cargo.toml
│   └── Dockerfile
│
├── utxo-indexer-common/                # EXISTING — shared types, DB models
│   ├── src/
│   │   ├── lib.rs
│   │   ├── schema.rs                  # Diesel schema (add ordering_buffer_pending)
│   │   ├── models.rs                  # DB models
│   │   ├── types.rs                   # InsertionEvent, EventType, etc.
│   │   ├── sqs_types.rs              # NEW — SqsUtxoEnvelope (shared)
│   │   └── discriminators.rs          # Event discriminator constants
│   ├── migrations/
│   └── Cargo.toml
```

**Key dependency changes:**
- `utxo-indexer` (consumer): Remove `reqwest`, HMAC crate. Add `aws-sdk-sqs`, `aws-sdk-dynamodb`.
- `utxo-indexer-write-service`: Remove `axum`, `tower-http`, HMAC verification. Add `aws-sdk-sqs`, `aws-sdk-dynamodb`. Add borsh parsing (moved from processor).

---

## 11. AWS Deployment Topology

```
Production:
─────────────────────────────────────────────────────────────
  ECS Fargate:
    1x  Consumer Service     (0.5 vCPU, 1GB RAM)    ← always exactly 1
    1x  Write Service        (2 vCPU, 4GB RAM)       ← single instance (tree ordering)
    2x  Read Service         (1 vCPU, 2GB RAM)       ← behind ALB, auto-scale on CPU

  SQS:
    1x  Standard Queue       (utxo-indexer-txs)
    1x  Dead-letter Queue    (utxo-indexer-txs-dlq)

  RDS PostgreSQL 16:
    1x  Primary              (db.r6g.large — 2 vCPU, 16GB RAM, gp3 SSD)
    1x  Read Replica         (db.r6g.medium — 2 vCPU, 8GB RAM)

  DynamoDB:
    1x  Table                (indexer-checkpoints, on-demand capacity)
                             (shared with comprehensive indexer)

  ECR:
    3x  Repositories         (utxo-consumer, utxo-write, utxo-read)

Staging:
─────────────────────────────────────────────────────────────
  ECS Fargate:
    1x  Consumer             (0.25 vCPU, 0.5GB RAM)
    1x  Write Service        (1 vCPU, 2GB RAM)
    1x  Read Service         (0.5 vCPU, 1GB RAM)

  SQS: Same (free tier)
  RDS: db.t4g.small (shared with comprehensive indexer)
```

**Auto-scaling:**
- Read service: Scale up when ALB response time > 200ms or CPU > 70%
- Write service: Single instance (Merkle tree ordering constraint). If throughput demands it, shard by tree_index across instances.
- Consumer: Always 1 instance (Laserstream single-consumer)

---

## 12. Migration Path from Current Architecture

### Phase 1: Add SQS sender to consumer (parallel with HTTP)
- Consumer sends to both SQS and HTTP (existing path)
- Write service continues reading from HTTP
- Validate SQS messages match HTTP requests

### Phase 2: Add SQS poller to write service
- Write service reads from both SQS and HTTP
- Verify Merkle tree state matches between both paths
- SQS path shadow-processes (logs but doesn't write)

### Phase 3: Switch primary path to SQS
- Consumer stops sending HTTP
- Write service stops HTTP listener
- SQS is now the only transport

### Phase 4: Remove HTTP code
- Remove reqwest from consumer
- Remove axum/HMAC from write service
- Clean up unused code

Each phase is independently deployable and reversible.

---

## 13. Summary of Key Decisions

- **Dual-queue architecture** — DropOldest internal buffer for burst absorption (producer never blocks), SQS Standard for durable inter-service transport. Each queue serves a distinct purpose.
- **Discriminator filter stays in consumer** — The 8-byte discriminator check is ~microseconds and filters out ~90% of irrelevant transactions. Moving it to the write service would waste SQS throughput on messages that will be immediately discarded.
- **All parsing moves to write service** — Consumer does zero borsh decoding, zero event extraction. The write service owns the full parsing pipeline, making it the single source of truth for business logic.
- **Ordering buffer in write service** — Handles SQS Standard's lack of ordering guarantees. BTreeMap per tree, drains contiguous sequences, triggers backfill on persistent gaps.
- **SQS Standard over FIFO** — FIFO has a 3K msg/s limit (300 without batching per group) and adds complexity. Standard queue + write-service ordering is simpler and higher throughput.
- **HTTP/HMAC eliminated** — SQS replaces the HTTP transport entirely. No more reqwest, HMAC key management, or HTTP error handling between services.
- **Single write service instance** — Merkle tree insertion must be sequential per tree. A single instance avoids distributed ordering complexity. Can shard by tree_index if throughput demands it later.
- **DynamoDB for checkpoints** — Both consumer and write service checkpoint to DynamoDB. Zero PostgreSQL dependency for the consumer. Shared table with the comprehensive indexer.
- **Phased migration** — Parallel HTTP+SQS paths allow gradual, reversible migration from current architecture.
