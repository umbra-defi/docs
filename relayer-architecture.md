# Umbra Relayer Microservice Architecture

## 1. System Overview

The Umbra Relayer is a horizontally-scalable, microservice-based system that accepts anonymous claim requests from users, constructs and submits Solana transactions on their behalf, monitors MPC computation callbacks, and cleans up temporary accounts. The relayer earns fees (SOL + SPL) for providing this service.

**Design Principles:**
- Rust for all services (high performance, memory safety)
- AWS SQS for inter-service communication (durability, at-least-once delivery)
- Internal tokio::mpsc channels within each service for fine-grained concurrency
- Helius Laserstream (gRPC) for real-time callback monitoring — single stream filtered by program ID
- RPC polling fallback when Laserstream is unavailable
- DynamoDB for request lifecycle tracking (single-digit ms latency)
- Horizontal scaling: multiple instances share the same relayer keypair behind a load balancer
- Stateless services where possible; all durable state in DynamoDB/SQS

**Supported Claim Variants (flag-based):**

- `ConfidentialExistingMxe` — existing encrypted token account, MXE-only
- `ConfidentialExistingShared` — existing encrypted token account, shared encryption
- `ConfidentialNewMxe` — new encrypted token account, MXE-only
- `ConfidentialNewShared` — new encrypted token account, shared encryption
- `PublicMxe` — public SPL token account (ATA)

---

## 2. Microservice Topology

```
                                    +------------------+
                                    |   User / Client  |
                                    +--------+---------+
                                             |
                                         HTTP POST
                                             |
                                             v
                                 +-----------+----------+
                                 |   1. API Gateway     |
                                 |   (Ingestion &       |
                                 |    Deduplication)     |
                                 +-----------+----------+
                                             |
                                      SQS: claim-requests
                                             |
                                             v
                                 +-----------+----------+
                                 |  2. Preflight        |
                                 |     Validator        |
                                 +-----------+----------+
                                             |
                                      SQS: validated-claims
                                             |
                                             v
                                 +-----------+----------+
                                 |  3. Transaction      |
                                 |     Pipeline         |
                                 +-----------+----------+
                                             |
                                      SQS: pending-callbacks
                                             |
                                             v
                                 +-----------+----------+
                                 |  4. Callback         |
                                 |     Monitor          |
                                 +-----------+----------+
                                             |
                                      SQS: completed-claims
                                             |
                                             v
                                 +-----------+----------+
                                 |  5. Cleanup          |
                                 |     Service          |
                                 +-----------+----------+
                                             |
                                      Updates DynamoDB
                                             |
                                             v
                                 +-----------+----------+
                                 |  (User polls         |
                                 |   GET /claims/{id})  |
                                 +-----------+----------+


         +-------------------+
         | 6. Relayer Admin  |    (lower priority, separate deployment)
         |    Service        |
         +-------------------+

         +-------------------+
         | Shared Infra:     |
         | - DynamoDB        |
         | - Redis           |
         | - AWS KMS / HSM   |
         +-------------------+
```

---

## 3. Microservice Definitions

### 3.1 API Gateway Service

**Purpose:** Accept user claim requests, validate schema, deduplicate offsets, persist to DB, enqueue for processing.

**Scaling:** Horizontally behind ALB. Stateless except for Redis bloom filter lookups.

**Endpoints:**

```
POST /v1/claims
  Body: ClaimRequest (see Section 5)
  Response: { request_id: UUID, status: "accepted" }

GET /v1/claims/{request_id}
  Response: ClaimStatus (see Section 8)

GET /v1/health
  Response: { status: "ok", supported_mints: [...] }
```

**Internal Architecture:**

```
HTTP Listener (axum / actix-web)
        |
        v
+-------------------+
| Request Validator |  <-- Schema validation, field range checks
+-------------------+
        |
        v
+-------------------+
| Offset Dedup      |  <-- Redis bloom filter + DynamoDB conditional write
| (bloom + DB)      |
+-------------------+
        |
        v
+-------------------+
| Config Checker    |  <-- Verify mint is supported, claim variant enabled
+-------------------+
        |
        v
+-------------------+
| DB Writer         |  <-- DynamoDB PutItem (status: "accepted")
+-------------------+
        |
        v
+-------------------+
| SQS Publisher     |  <-- Enqueue to `claim-requests`
+-------------------+
```

**Offset Deduplication Strategy:**

The user provides all offsets (computation_offset, proof_account_offset, mpc_callback_data_offset, unified_fees_pool_account_offset, burner_account_offset). The relayer must ensure no two in-flight requests share the same offsets (which would cause PDA collisions).

- **Layer 1 — Redis Bloom Filter:** Fast probabilistic check. False positives are acceptable (we reject and ask user to retry with new offsets). False negatives are impossible.
- **Layer 2 — DynamoDB Conditional Write:** For each offset, write to `active-offsets` table with `attribute_not_exists(pk)` condition. This is the authoritative dedup. Offsets are removed when the claim reaches terminal state (completed/failed).

**Configuration (loaded from file):**

```toml
# relayer-config.toml

[relayer]
keypair_path = "/secrets/relayer-keypair.json"  # or KMS ARN
program_id = "C6KsXC5aFhffHVnaHn6LxQzM3SJGmdW6mB6FWNbwJ2Kr"
arcium_program_id = "..."
cluster_offset = 10000

[network]
rpc_url = "https://mainnet.helius-rpc.com/?api-key=..."
laserstream_url = "grpc+tls://mainnet.laserstream.helius.dev"
commitment = "confirmed"

[supported_mints]
# Each entry enables that mint for relaying
[supported_mints.SOL]
address = "So11111111111111111111111111111111111111112"
enabled = true

[supported_mints.USDC]
address = "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v"
enabled = true

[claim_variants]
confidential_existing_mxe = true
confidential_existing_shared = true
confidential_new_mxe = true
confidential_new_shared = true
public_mxe = true

[limits]
max_concurrent_claims = 10000
request_ttl_seconds = 3600
max_priority_fees_lamports = 100000
```

---

### 3.2 Preflight Validator Service

**Purpose:** Perform all off-chain validations that the on-chain program would check, to avoid wasting SOL on transactions that will fail.

**Scaling:** Horizontally. Each instance consumes from `claim-requests` SQS queue independently. RPC-heavy but stateless.

**Internal Architecture:**

```
SQS Consumer (long-poll, batch_size=10)
        |
        v
+---------------------+     tokio::mpsc
| Dispatch Channel    | ----+----+----+----+
+---------------------+     |    |    |    |
                             v    v    v    v
                     +-------+----+----+----+-------+
                     |  Validation Worker Pool      |
                     |  (N concurrent workers)      |
                     +-------+----------------------+
                             |
                     For each claim:
                             |
               +-------------+-------------+
               |             |             |
               v             v             v
        +------+--+  +------+---+  +------+------+
        | Timestamp|  | Merkle   |  | Nullifier   |
        | Check   |  | Root     |  | Check       |
        +---------+  | Check    |  +-------------+
                     +----------+
               |             |             |
               v             v             v
        +------+--+  +------+---+  +------+------+
        | Account |  | Proof    |  | Fee Config  |
        | State   |  | Account  |  | Existence   |
        | Check   |  | Validity |  | Check       |
        +---------+  +----------+  +-------------+
               |             |             |
               +------+------+-------------+
                      |
                      v
              +-------+--------+
              | All passed?    |
              | YES -> SQS:    |
              |  validated-    |
              |  claims        |
              | NO -> DB:      |
              |  status=       |
              |  "rejected"    |
              +----------------+
```

**Validations Performed:**

1. **Timestamp Validation**
   - `tvk_timestamp >= 0`
   - `tvk_timestamp <= Clock::now()`
   - `(Clock::now() - tvk_timestamp) <= program_info.maximum_timestamp_difference`
   - **RPC calls:** `getAccountInfo(ProgramInformation)`, Solana clock

2. **Merkle Root Validation**
   - Fetch MixerTree account at the specified `mixer_tree_account_offset`
   - Verify `merkle_root` exists in the tree's root history ring buffer
   - **RPC calls:** `getAccountInfo(MixerTree PDA)`

3. **Nullifier Checks**
   - No duplicate nullifiers within the request's UTXO set
   - For each nullifier, check it doesn't exist in any of the 5 treap accounts
   - **RPC calls:** `getMultipleAccounts([treap_1..treap_5])`, deserialize + search

4. **Account State Checks**
   - `RelayerAccount` — is_initialised, is_active
   - `RelayerFeesConfiguration` — is_initialised, is_active, matches instruction variant
   - `ProtocolFeesConfiguration` — is_active
   - `UnifiedFeesPool` — exists and is_valid; if not initialized, flag for creation (Service 6)
   - `Pool` — check_confidentiality_validity, check_mixer_validity
   - `ProgramInformation` — is_valid
   - **RPC calls:** `getMultipleAccounts([relayer_account, relayer_fees_config, protocol_fees_config, unified_fees_pool, pool, program_info])`

5. **ZK Verifying Key Check**
   - Verify the ZK verifying key account exists at the expected PDA
   - For confidential: seed = `ZK_CLAIM_INTO_CONFIDENTIAL_BALANCE_BASE_SEED + number_of_burns`
   - For public: seed = `ZK_CLAIM_INTO_PUBLIC_BALANCE_SEED`
   - **RPC calls:** `getAccountInfo(ZkVerifyingKey PDA)`

6. **Proof Data Validity**
   - All field elements are valid BN254 scalars (< field modulus)
   - All PoseidonHash values are valid field elements
   - All RescueCiphertext values are valid

7. **Amount Bounds (Public Claims Only)**
   - `amount >= protocol_lower_bound && amount <= protocol_upper_bound`
   - `amount >= relayer_lower_bound && amount <= relayer_upper_bound`
   - Fee Merkle proofs verify against `protocol_fees_configuration.fees_root` and `relayer_fees_configuration.fees_root`

8. **Receiver Account Checks**
   - Confidential existing: `ArciumEncryptedTokenAccount` exists with MXE or shared balance
   - Confidential new: `ArciumEncryptedTokenAccount` does NOT exist (will be created)
   - Public: receiver ATA derived correctly; `init_if_needed` handles creation

**RPC Optimization:** Batch all account fetches into 2-3 `getMultipleAccounts` calls rather than individual `getAccountInfo` calls. Cache `ProgramInformation` and `Pool` accounts with short TTL (5-10s) since they change infrequently.

---

### 3.3 Transaction Pipeline Service

**Purpose:** Build, sign, send, and confirm all transactions for a validated claim. This is the core workhorse that orchestrates the multi-transaction sequence.

**Scaling:** Horizontally. Each instance consumes from `validated-claims` queue. Since Solana uses blockhash-based tx lifetimes (not account nonces like Ethereum), multiple instances can send transactions with the same fee payer keypair concurrently without nonce collisions.

**Internal Architecture:**

```
SQS Consumer (long-poll, batch_size=5)
        |
        v
+---------------------+       tokio::mpsc (bounded, backpressure)
| Dispatch Channel    | ----+----+----+
+---------------------+     |    |    |
                             v    v    v
                     +-------+----+----+-------+
                     |  Pipeline Worker Pool   |
                     |  (N concurrent workers) |
                     +-------+-----------------+
                             |
                     Per claim, sequential stages:
                             |
                 +-----------+-----------+
                 |                       |
                 v                       v
         [Confidential Path]     [Public Path]
                 |                       |
    +------------+----------+    +-------+--------+
    |                       |    |                |
    v                       v    v                v
 Stage 1:              Stage 1:              Stage 1:
 Burner Updates        Burner Update         Proof Account
 (2-4 txs serial)     (1 tx)                Create (1 tx)
    |                       |                     |
    v                       v                     v
 Stage 2:              Stage 2:              Stage 2:
 Proof Account         Proof Account         Claim Tx
 Create (1 tx)         Create (1 tx)         (1 tx)
    |                       |
    v                       v
 Stage 3:              Stage 3:
 Claim Tx              Claim Tx
 (1 tx)                (1 tx)
    |                       |
    +----------+------------+
               |
               v
        SQS: pending-callbacks
```

**Stage Details:**

**Stage 1a — Burner Account Updates (Confidential, 2-4 txs):**

The user provides up to 4 UTXOs. Each `update_confidential_utxo_burner` instruction can insert 1-2 UTXOs per transaction.

- TX 1: Create burner account + insert UTXOs 0-1 (or just UTXO 0 if odd count)
  - First call: allocates account, initializes header, inserts slot(s)
  - PDA: `[CONFIDENTIAL_UTXO_BURNER_SEED, relayer_key, user_address, burner_account_offset]`
- TX 2: Insert UTXOs 2-3 (if 3-4 UTXOs)
- Each TX must confirm before next (sequential — header nonce changes)

**Stage 1b — Burner Account Update (Public, 1 tx):**

- Single UTXO, single `update_public_utxo_burner` call
- PDA: `[PUBLIC_UTXO_BURNER_SEED, relayer_key, user_address, burner_account_offset]`

**Stage 2 — Proof Account Create (1 tx):**

- Confidential: `create_confidential_claim_proof_account`
  - PDA: `[SHA256("ConfidentialClaimProofAccount"), relayer_key, proof_account_offset]`
  - Stores: rescue encryption data, Groth16 proof (A/B/C), merkle root, relayer fees, TVK timestamp
- Public: `create_public_claim_proof_account`
  - PDA: `[SHA256("PublicClaimProofAccount"), relayer_key, proof_account_offset]`
  - Stores: everything above PLUS protocol/relayer fee Merkle proofs (bounds, path[4], leaf_index)

**Stage 3 — Claim Instruction (1 tx):**

Builds the main claim instruction. The proof account is **closed inline** by the on-chain program (`close = relayer`). The MPC callback data account is **initialized** (`init, payer = fee_payer`).

Accounts required (confidential existing MXE example):
- Arcium accounts: sign_pda, mxe, mempool, execpool, computation, comp_def, cluster, arcium_pool, clock
- Proof account (closed inline)
- Receiver: receiver_address, receiver_token_account
- Relayer: relayer, fee_payer, relayer_account, relayer_fees_configuration
- Fees: protocol_fees_configuration, unified_fees_pool
- Pool: pool, mint, program_information
- Mixer: mixer_tree
- Nullifiers: nullifier_storing_account (burner), treap_1..treap_5
- ZK: zk_verifying_key_account
- MPC: mpc_callback_data (initialized here)
- System: system_program, arcium_program

Uses Address Lookup Tables (ALTs) to fit within Solana's account limit.

**Transaction Signing:**

All instances share the relayer keypair. Options:
- **AWS KMS:** Sign via KMS API call (adds ~10-50ms latency per sign). Most secure.
- **In-memory:** Load keypair at startup from encrypted config. Faster but requires secure deployment.

**Transaction Confirmation:**

- Send with `skipPreflight: false` for immediate simulation errors
- Poll `getSignatureStatuses` with 1s interval, 60s timeout
- On blockhash expiry: rebuild with fresh blockhash and retry (max 3 retries)
- On simulation failure: mark claim as `"tx_failed"` in DB, publish to `failed-claims` SQS for cleanup

**Failure Handling:**

If any stage fails, the pipeline must handle partial state:
- Burner created but claim not sent → Cleanup service closes burner (refunds rent to relayer)
- Proof account created but claim not sent → Cleanup service closes proof account
- Claim sent but callback fails → Callback monitor detects timeout, cleanup service handles

The pipeline stores each completed tx signature in DynamoDB so the cleanup service knows what was created.

---

### 3.4 Callback Monitor Service

**Purpose:** Monitor all Arcium callback transactions to the Umbra program via a single Helius Laserstream gRPC stream. Detect success (at least one callback finalizes the computation) or failure (no successful callback within 150 confirmed blocks).

**Arcium Callback Model:**

For every queued computation, Arcium's MPC nodes send **multiple callbacks** (one per node). The outcomes are:

- **Success:** At least one callback transaction succeeds → `ComputationAccount.status` transitions to `Finalized`
- **Failure (all callbacks fail):** Multiple callback transactions land but all fail (revert). The `ComputationAccount.status` remains `Queued`. The computation is lost.
- **Failure (no callbacks):** No callback transactions arrive within 150 confirmed blocks. The computation is lost.

In both failure cases, the computation account and MPC callback data account must be cleaned up. The computation cannot be retried — the user must submit a new claim request.

**Scaling:** This service runs as a **singleton** (or active-passive pair for HA). A single Laserstream connection receives ALL transactions touching the Umbra program — no per-account subscriptions needed. The service maintains an in-memory map of pending computation accounts and matches incoming callback transactions against it.

**Why Laserstream over WebSocket per-account subscriptions:**

- **1 gRPC stream vs ~1,250+ WebSocket subscriptions** — Laserstream filters by program ID at the source. No subscription lifecycle management.
- **Zero churn** — No subscribe/unsubscribe on every new computation. At 250 TPS, WebSocket would require ~500 subscription operations/second. Laserstream has zero.
- **No race window** — WebSocket subscriptions have a gap between queuing the computation and subscribing (could miss a fast callback). Laserstream streams everything continuously.
- **Simpler failure model** — One connection to manage instead of a pool of WebSocket connections with per-subscription state.
- **Slot context built-in** — Laserstream provides the slot for each transaction, eliminating the need for a separate `slotSubscribe` or `getSlot` call.

**Internal Architecture:**

```
SQS Consumer (long-poll)
        |
        v
+---------------------+       tokio::mpsc (unbounded)
| Registration        | -----> Watch Registry (DashMap<Pubkey, WatchEntry>)
| Channel             |
+---------------------+

+---------------------+
| Laserstream gRPC    |  Single connection filtered by Umbra program ID
| Stream Handler      |  Receives ALL transactions touching the program
+----------+----------+
           |
           | For each transaction:
           v
+----------+-----------+
| Transaction Filter   |
| 1. Extract inner     |
|    instructions      |
| 2. Check for         |
|    arcium_callback   |
|    discriminator     |
| 3. Extract comp      |
|    account address   |
|    from accounts     |
+----------+-----------+
           |
           | Matching callback found
           v
+----------+-----------+
| Watch Matcher        |
| 1. Lookup comp       |
|    account in        |
|    Watch Registry    |
| 2. If not tracked    |
|    → skip (not ours) |
| 3. If tracked →      |
|    evaluate status   |
+----------+-----------+
           |
    +------+------+
    |             |
    v             v
[Success]    [Failed callback]
    |         (tx err != null)
    |             |
    v             v
 SQS:         Log + continue
 completed-   watching (timeout
 claims       sweeper handles
              deadline)

+---------------------+
| Timeout Sweeper     |  Background task every 5s
| Checks all watches  |  Uses slot from latest Laserstream update
| against deadline    |  Publishes to SQS: failed-callbacks
+---------------------+
```

**Laserstream gRPC Client:**

```rust
use yellowstone_grpc_client::GeyserGrpcClient;
use yellowstone_grpc_proto::prelude::*;

/// Single Laserstream gRPC connection that streams all transactions
/// involving the Umbra program. Filters are applied server-side,
/// so only relevant transactions are sent over the wire.
struct LaserstreamClient {
    /// gRPC endpoint (e.g., "https://mainnet.laserstream.helius.dev")
    endpoint: String,
    /// Helius API key for authentication
    api_key: String,
    /// The Umbra program ID to filter by
    program_id: Pubkey,
}

impl LaserstreamClient {
    /// Connect to Laserstream and subscribe to all transactions
    /// that include the Umbra program in their account keys.
    async fn connect(&self) -> Result<impl Stream<Item = Result<SubscribeUpdate>>> {
        let mut client = GeyserGrpcClient::build_from_shared(self.endpoint.clone())?
            .x_token(Some(self.api_key.clone()))?
            .connect()
            .await?;

        // Subscribe to transactions that reference the Umbra program.
        // This captures all callback transactions since they invoke
        // the Umbra program's arcium_callback handler.
        let mut transactions_filter = HashMap::new();
        transactions_filter.insert(
            "umbra_callbacks".to_string(),
            SubscribeRequestFilterTransactions {
                vote: Some(false),
                failed: Some(true),  // Include failed txs to detect failed callbacks
                account_include: vec![self.program_id.to_string()],
                account_exclude: vec![],
                account_required: vec![],
                signature: None,
            },
        );

        let (_, stream) = client
            .subscribe_once(
                HashMap::new(),         // no slot filters (slot comes with tx context)
                HashMap::new(),         // no account filters
                transactions_filter,
                HashMap::new(),         // no block filters
                HashMap::new(),         // no block_meta filters
                Some(CommitmentLevel::Confirmed),
                vec![],                 // no account_data filters
                None,                   // no ping
            )
            .await?;

        Ok(stream)
    }
}
```

**Stream Processing Loop:**

```rust
/// Main loop: processes every transaction from the Laserstream stream.
/// For each transaction:
/// 1. Parse the transaction to find arcium_callback instructions
/// 2. Extract the computation account address from the instruction's accounts
/// 3. Match against the watch registry
/// 4. Evaluate success/failure
async fn process_stream(
    stream: impl Stream<Item = Result<SubscribeUpdate>>,
    watch_registry: &DashMap<Pubkey, WatchEntry>,
    latest_slot: &AtomicU64,
    sqs_client: &SqsClient,
    rpc_client: &RpcClient,
    db_client: &DbClient,
) {
    pin_mut!(stream);

    while let Some(update) = stream.next().await {
        let update = match update {
            Ok(u) => u,
            Err(e) => {
                tracing::error!("Laserstream stream error: {e}");
                break; // Triggers reconnection logic
            }
        };

        let Some(update_oneof) = update.update_oneof else { continue };

        match update_oneof {
            UpdateOneof::Transaction(tx_update) => {
                let slot = tx_update.slot;
                latest_slot.store(slot, Ordering::Relaxed);

                let Some(tx_info) = tx_update.transaction else { continue };
                let is_success = tx_info.meta
                    .as_ref()
                    .map(|m| m.err.is_none())
                    .unwrap_or(false);

                // Parse the transaction for arcium_callback instructions
                let callback_accounts = extract_callback_computation_accounts(
                    &tx_info,
                    &UMBRA_PROGRAM_ID,
                );

                for comp_account_address in callback_accounts {
                    // Check if this computation account is one we're watching
                    let Some(watch) = watch_registry.get(&comp_account_address) else {
                        continue; // Not our computation — skip
                    };

                    if is_success {
                        // SUCCESS: Callback transaction succeeded.
                        // The computation is finalized.
                        let tx_sig = tx_info.signature_as_string();

                        tracing::info!(
                            request_id = %watch.request_id,
                            computation_account = %comp_account_address,
                            callback_tx = %tx_sig,
                            slot = slot,
                            "Successful callback detected via Laserstream"
                        );

                        sqs_client.publish_completed(CompletedClaimMessage {
                            request_id: watch.request_id,
                            computation_account_address: comp_account_address,
                            computation_offset: watch.computation_offset,
                            cluster_offset: watch.cluster_offset,
                            callback_detected_at: now_millis(),
                            callback_tx_signature: Some(tx_sig),
                        }).await;

                        db_client.update_status(&watch.request_id, "callback_done").await;
                        drop(watch); // Release DashMap ref before remove
                        watch_registry.remove(&comp_account_address);
                    } else {
                        // FAILED callback transaction. Don't remove the watch —
                        // other MPC nodes may still send successful callbacks.
                        // The timeout sweeper handles the hard deadline.
                        tracing::debug!(
                            request_id = %watch.request_id,
                            computation_account = %comp_account_address,
                            slot = slot,
                            "Failed callback detected — still waiting for successful callback"
                        );
                    }
                }
            }
            _ => {
                // Ignore non-transaction updates (slots, pings, etc.)
            }
        }
    }
}

/// Parse a transaction to find arcium_callback instructions targeting the
/// Umbra program, and extract the computation account address from each.
///
/// The arcium_callback instruction has a known 8-byte discriminator.
/// The computation account is at a known index in the instruction's accounts list.
fn extract_callback_computation_accounts(
    tx_info: &SubscribeUpdateTransactionInfo,
    umbra_program_id: &Pubkey,
) -> Vec<Pubkey> {
    let Some(tx) = &tx_info.transaction else { return vec![] };
    let Some(msg) = &tx.message else { return vec![] };

    let account_keys: Vec<Pubkey> = msg.account_keys
        .iter()
        .chain(
            tx_info.meta.as_ref()
                .map(|m| m.loaded_writable_addresses.iter().chain(m.loaded_readonly_addresses.iter()))
                .into_iter()
                .flatten()
        )
        .filter_map(|k| Pubkey::try_from(k.as_slice()).ok())
        .collect();

    let mut comp_accounts = Vec::new();

    for ix in &msg.instructions {
        let program_idx = ix.program_id_index as usize;
        if program_idx >= account_keys.len() { continue; }
        if account_keys[program_idx] != *umbra_program_id { continue; }

        // Check if this is an arcium_callback instruction by discriminator
        if ix.data.len() >= 8 && ix.data[..8] == ARCIUM_CALLBACK_DISCRIMINATOR {
            // The computation account is at a known position in the accounts list.
            // This position is consistent across all callback variants since the
            // #[callback_accounts(...)] macro places Arcium accounts first.
            const COMP_ACCOUNT_INDEX: usize = 4; // Position in the callback's account list
            if let Some(&account_idx) = ix.accounts.get(COMP_ACCOUNT_INDEX) {
                if let Some(addr) = account_keys.get(account_idx as usize) {
                    comp_accounts.push(*addr);
                }
            }
        }
    }

    comp_accounts
}
```

**Watch Entry & Slot-Based Timeout:**

```rust
struct WatchEntry {
    request_id: Uuid,
    computation_offset: u64,
    cluster_offset: u32,
    /// The confirmed slot at which the claim transaction was confirmed.
    /// Used to compute the 150-block deadline.
    claim_confirmed_slot: u64,
    /// Absolute slot deadline: claim_confirmed_slot + 150.
    /// If current confirmed slot exceeds this and no successful callback
    /// has been detected, the computation is declared failed.
    deadline_slot: u64,
    /// For cleanup: offsets of accounts that need closing on failure
    mpc_callback_data_offset: u128,
    burner_account_offset: u128,
}

/// The deadline is a hard Arcium protocol guarantee:
/// If no successful callback arrives within 150 confirmed blocks
/// after the claim transaction, the computation is lost.
const ARCIUM_CALLBACK_DEADLINE_BLOCKS: u64 = 150;

impl WatchEntry {
    fn new(msg: &PendingCallbackMessage) -> Self {
        Self {
            request_id: msg.request_id,
            computation_offset: msg.computation_offset,
            cluster_offset: msg.cluster_offset,
            claim_confirmed_slot: msg.claim_confirmed_slot,
            deadline_slot: msg.claim_confirmed_slot + ARCIUM_CALLBACK_DEADLINE_BLOCKS,
            mpc_callback_data_offset: msg.mpc_callback_data_offset,
            burner_account_offset: msg.burner_account_offset,
        }
    }
}
```

**Timeout Sweeper:**

```rust
/// Background task that runs every 5 seconds, checking all active watches
/// against the latest slot observed from the Laserstream stream.
///
/// If latest_slot > watch.deadline_slot and no successful callback has been
/// detected, the computation is declared FAILED:
/// - No successful callback arrived within 150 confirmed blocks
/// - The computation is lost and cannot be retried
/// - Cleanup is required: computation account + MPC callback data account
///   + burner account (which wasn't closed since no successful callback ran)
async fn timeout_sweeper(
    watches: &DashMap<Pubkey, WatchEntry>,
    latest_slot: &AtomicU64,
    sqs_client: &SqsClient,
    db_client: &DbClient,
) {
    let mut interval = tokio::time::interval(Duration::from_secs(5));
    loop {
        interval.tick().await;
        let current_slot = latest_slot.load(Ordering::Relaxed);

        // Collect expired watches (can't mutate DashMap while iterating)
        let expired: Vec<(Pubkey, WatchEntry)> = watches
            .iter()
            .filter(|entry| current_slot > entry.value().deadline_slot)
            .map(|entry| (*entry.key(), entry.value().clone()))
            .collect();

        for (address, watch) in expired {
            tracing::warn!(
                request_id = %watch.request_id,
                deadline_slot = watch.deadline_slot,
                current_slot = current_slot,
                "Computation callback timeout: no successful callback within 150 confirmed blocks"
            );

            sqs_client.publish_failed_callback(FailedCallbackMessage {
                request_id: watch.request_id,
                failure_reason: "callback_timeout_150_blocks".to_string(),
                computation_account_address: address,
                computation_offset: watch.computation_offset,
                cluster_offset: watch.cluster_offset,
                created_accounts: CreatedAccounts {
                    mpc_callback_data_offset: Some(watch.mpc_callback_data_offset),
                    burner_offset: Some(watch.burner_account_offset),
                    ..Default::default()
                },
            }).await;

            db_client.update_status(&watch.request_id, "callback_timeout").await;
            watches.remove(&address);
        }
    }
}
```

**Reconnection & Replay:**

```rust
/// Laserstream reconnection with replay-from-slot.
/// On disconnect, reconnect and replay from the last observed slot
/// to avoid missing any callbacks during the gap.
///
/// The watch registry is preserved across reconnections since it's
/// in-memory state independent of the gRPC connection.
async fn run_with_reconnection(
    client: &LaserstreamClient,
    watch_registry: &DashMap<Pubkey, WatchEntry>,
    latest_slot: &AtomicU64,
    sqs_client: &SqsClient,
    rpc_client: &RpcClient,
    db_client: &DbClient,
) {
    loop {
        let stream = match client.connect().await {
            Ok(s) => {
                tracing::info!("Laserstream connected");
                s
            }
            Err(e) => {
                tracing::error!("Laserstream connection failed: {e}");
                tokio::time::sleep(Duration::from_secs(2)).await;
                continue;
            }
        };

        // Process stream until it disconnects
        process_stream(stream, watch_registry, latest_slot, sqs_client, rpc_client, db_client).await;

        // Stream ended — reconnect
        tracing::warn!(
            last_slot = latest_slot.load(Ordering::Relaxed),
            "Laserstream disconnected, reconnecting..."
        );
        tokio::time::sleep(Duration::from_millis(500)).await;
    }
}
```

**Fallback: RPC Polling Mode**

If Laserstream is persistently unavailable, the service degrades to RPC polling:
- Fetch all watched computation accounts via batched `getMultipleAccounts` (100 accounts per call) every 2 seconds
- Decode `ComputationAccount.status` for each — same evaluation logic
- Simultaneously fetch current slot via `getSlot` for timeout detection
- Automatically attempt Laserstream reconnection in background

**Capacity Planning at 250 TPS:**

- **Single gRPC connection** handles all monitoring — no per-account subscriptions
- Typical MPC callback time: ~2-5 seconds (most finalize within a few seconds)
- Upper bound: 60 seconds (hard protocol cap: 150 confirmed blocks)
- Concurrent watches in registry (typical): 250 TPS x 5s = ~1,250 entries
- Concurrent watches in registry (worst case): 250 TPS x 60s = ~15,000 entries
- Watch registry is a DashMap — O(1) lookup per incoming transaction
- Laserstream bandwidth: only Umbra program transactions streamed (not full chain)
- Singleton deployment: 1 instance (active), 1 standby for HA
- No shard coordination needed (unlike WebSocket approach)

---

### 3.5 Cleanup Service

**Purpose:** Close computation accounts (reclaim rent), handle failed claims (close orphaned burner/proof accounts), update final DB status.

**Scaling:** Horizontally. Consumes from `completed-claims` and `failed-callbacks` queues.

**Internal Architecture:**

```
SQS Consumers (2 queues)
        |
        v
+----------------------------+
| Cleanup Dispatch Channel   | ----+----+
+----------------------------+     |    |
                                   v    v
                           +-------+----+-------+
                           | Cleanup Worker Pool |
                           +-------+-------------+
                                   |
                    +--------------+--------------+
                    |                             |
                    v                             v
           [Successful Claim]            [Failed Claim / Timeout]
                    |                             |
                    v                             v
           +-------+--------+           +--------+-----------+
           | Close Comp     |           | Close ALL Orphaned |
           | Account        |           | Accounts:          |
           | (claimComp-    |           | - Comp Account     |
           |  utationRent)  |           | - MPC Callback     |
           +-------+--------+           |   Data Account     |
                    |                    | - Burner Account   |
                    v                    | - Proof Account    |
           +-------+--------+           |   (if claim tx     |
           | Release offsets |           |    never sent)     |
           | DB: status=     |           +--------+-----------+
           | "completed"     |                    |
           +----------------+                     v
                                         +--------+-----------+
                                         | Release offsets    |
                                         | DB: status=        |
                                         | "failed_cleaned"   |
                                         +--------------------+
```

**Cleanup Paths:**

There are two distinct cleanup paths depending on whether the claim succeeded or failed:

**Path A — Successful Claim (from `completed-claims` queue):**

The MPC callback succeeded, so the on-chain callback handler already:
- Closed the MPC callback data account (rent → relayer)
- Closed the burner account (rent → relayer)
- Burnt nullifiers into treaps
- Updated encrypted balances / transferred SPL tokens

The cleanup service only needs to:
1. Close the computation account via `claimComputationRent`
2. Release offsets from the `active-offsets` DynamoDB table
3. Update DB status to `"completed"`

**Path B — Failed Claim (from `failed-callbacks` queue):**

No successful callback arrived within 150 confirmed blocks. The computation is **lost and cannot be retried**. The following accounts are still open and must be cleaned up:

1. **Computation account** — Close via `claimComputationRent` (Arcium instruction)
2. **MPC callback data account** — This account was `init`'d in the claim instruction but only `close = relayer`'d in the callback struct. Since no successful callback ran, it remains open. **The relayer cannot close it via a program instruction** — there is no dedicated close instruction for MPC callback data in the current program. This rent is lost unless a close instruction is added.
   - **Recommendation:** Add a `close_mpc_callback_data` admin instruction to the Umbra program that allows the relayer to close the callback data account after the 150-block deadline has passed.
3. **Burner account** — Same issue: closed only by the callback handler. No dedicated close instruction exists.
   - **Recommendation:** Add a `close_utxo_burner` admin instruction to the Umbra program.
4. **Proof account** — Only relevant if the claim tx was never sent (mid-pipeline failure). The proof account IS closeable via `close_confidential_claim_proof_account` / `close_public_claim_proof_account` since these instructions exist.

**Computation Account Closure:**

Uses the Arcium `claimComputationRent` instruction:
- Accounts: signer (relayer, writable), comp (computation account, writable), system_program
- Parameters: `comp_offset: u64`, `cluster_offset: u32`
- The Arcium program closes the account and refunds rent to signer

```rust
// Build claimComputationRent instruction
fn build_claim_computation_rent_ix(
    relayer: &Pubkey,
    computation_account: &Pubkey,
    comp_offset: u64,
    cluster_offset: u32,
) -> Instruction {
    // Instruction data: discriminator + comp_offset (u64 LE) + cluster_offset (u32 LE)
    let mut data = Vec::with_capacity(14);
    data.extend_from_slice(&CLAIM_COMPUTATION_RENT_DISCRIMINATOR);
    data.extend_from_slice(&comp_offset.to_le_bytes());
    data.extend_from_slice(&cluster_offset.to_le_bytes());

    Instruction {
        program_id: ARCIUM_PROGRAM_ID,
        accounts: vec![
            AccountMeta::new(*relayer, true),      // signer
            AccountMeta::new(*computation_account, false), // comp
            AccountMeta::new_readonly(system_program::ID, false),
        ],
        data,
    }
}
```

**Orphaned Account Summary:**

| Account | Closed by on success | Closeable on failure? | Action needed |
|---|---|---|---|
| Computation Account | Not auto-closed | Yes — `claimComputationRent` | Cleanup service sends tx |
| MPC Callback Data | Callback (`close = relayer`) | **No** — no close instruction | Rent lost; add close IX to program |
| Burner Account | Callback (zeroed + refunded) | **No** — no close instruction | Rent lost; add close IX to program |
| Proof Account | Claim handler (`close = relayer`) | Yes — dedicated close IX | Cleanup service sends tx (if claim tx never sent) |

**Note:** The MPC callback data and burner account rent loss on failure is a known limitation. At current Solana rent rates, the cost per failed claim is:
- MPC callback data: ~0.002-0.005 SOL (depending on variant size)
- Burner account: ~0.002-0.01 SOL (depending on UTXO count)

Adding close instructions to the program would eliminate this cost entirely.

---

### 3.6 Relayer Admin Service (Lower Priority)

**Purpose:** Administrative operations, health monitoring, account initialization, and future automated fee collection.

**Scaling:** Single instance (admin operations are infrequent).

**Responsibilities:**

1. **Account Initialization on Startup:**
   - For each supported mint in config, verify existence of:
     - `RelayerAccount` PDA — `[SHA256("RelayerAccount"), relayer_key, mint]`
     - `RelayerFeesConfiguration` PDAs for each enabled claim variant
     - `ProtocolFeesConfiguration` PDAs (read-only, must exist — admin creates these)
     - `UnifiedFeesPool` PDAs for each instruction variant + mint + offset
   - If any are missing, log warnings or auto-initialize where authorized

2. **Health Monitoring:**
   - SOL balance of relayer wallet (alerts if below threshold)
   - Staging SOL balance in `RelayerAccount` per mint
   - Queue depths for all SQS queues
   - DynamoDB request counts by status
   - Laserstream connection health (connected, last slot received, reconnection count)

3. **Future: Automated Fee Collection:**
   - Monitor `UnifiedFeesPool.encrypted_transaction_count`
   - When threshold met, trigger `collect_relayer_fees_from_unified_pool_v3`
   - Withdraw collected fees to relayer wallet

---

## 4. SQS Queue Topology

```
+-------------------+     +-----------------------+     +---------------------+
| claim-requests    | --> | validated-claims      | --> | pending-callbacks   |
| (Standard Queue)  |     | (Standard Queue)      |     | (Standard Queue)    |
+-------------------+     +-----------------------+     +---------------------+
                                                                |
                                                                v
                          +---------------------+     +---------------------+
                          | failed-callbacks    |     | completed-claims    |
                          | (Standard Queue)    |     | (Standard Queue)    |
                          +---------------------+     +---------------------+
                                    |                           |
                                    v                           v
                          +---------------------+     +---------------------+
                          | (both consumed by   |     | (consumed by        |
                          |  Cleanup Service)   |     |  Cleanup Service)   |
                          +---------------------+     +---------------------+

                          +---------------------+
                          | dead-letter-queue   |  <-- Failed messages from all queues
                          | (DLQ, Standard)     |      after max retries
                          +---------------------+
```

**Queue Configuration:**

- `claim-requests`: visibility_timeout=120s, max_receive_count=3, DLQ after 3 failures
- `validated-claims`: visibility_timeout=300s (tx pipeline can take time), max_receive_count=2
- `pending-callbacks`: visibility_timeout=600s (MPC can take minutes), max_receive_count=1
- `completed-claims`: visibility_timeout=120s, max_receive_count=3
- `failed-callbacks`: visibility_timeout=120s, max_receive_count=3
- `dead-letter-queue`: retention=14 days, no consumer (manual inspection)

**Message Schemas:**

### claim-requests

```rust
struct ClaimRequestMessage {
    request_id: Uuid,
    claim_type: ClaimVariant,       // enum: 5 variants
    received_at: u64,               // unix millis
    // Forwarded directly from user request (see Section 5)
    payload: ClaimRequestPayload,
}
```

### validated-claims

```rust
struct ValidatedClaimMessage {
    request_id: Uuid,
    claim_type: ClaimVariant,
    payload: ClaimRequestPayload,
    // Pre-fetched account state (avoids re-fetching in tx pipeline)
    preflight_state: PreflightState,
}

struct PreflightState {
    relayer_account_bump: u8,
    relayer_fees_config_bump: u8,
    protocol_fees_config_bump: u8,
    unified_fees_pool_bump: u8,
    pool_bump: u8,
    program_info_bump: u8,
    zk_verifying_key_bump: u8,
    mixer_tree_bump: u8,
    current_imt_index: u128,
    // Treap bumps for PDA reconstruction
    treap_bumps: [u8; 5],
    // Burner header bump (if account exists from prior attempt)
    burner_bump: Option<u8>,
}
```

### pending-callbacks

```rust
struct PendingCallbackMessage {
    request_id: Uuid,
    claim_type: ClaimVariant,
    computation_account_address: Pubkey,
    computation_offset: u64,
    cluster_offset: u32,
    claim_tx_signature: String,
    // The confirmed slot at which the claim tx landed.
    // Used by the callback monitor to compute the 150-block deadline.
    claim_confirmed_slot: u64,
    // Timestamp when claim tx was confirmed (for logging/metrics)
    claim_confirmed_at: u64,
    // Offsets needed for cleanup on failure (accounts that the callback
    // would have closed on success but remain open if no callback arrives)
    mpc_callback_data_offset: u128,
    burner_account_offset: u128,
}
```

### completed-claims

```rust
struct CompletedClaimMessage {
    request_id: Uuid,
    computation_account_address: Pubkey,
    computation_offset: u64,
    cluster_offset: u32,
    callback_detected_at: u64,
    // The successful callback tx signature (if retrieved via getSignaturesForAddress)
    callback_tx_signature: Option<String>,
}
```

### failed-callbacks

```rust
struct FailedCallbackMessage {
    request_id: Uuid,
    failure_reason: String,         // "callback_timeout_150_blocks" | "tx_pipeline_failure"
    computation_account_address: Pubkey,
    computation_offset: u64,
    cluster_offset: u32,
    // What was created (for cleanup)
    created_accounts: CreatedAccounts,
}

struct CreatedAccounts {
    burner_tx_signature: Option<String>,
    proof_account_tx_signature: Option<String>,
    claim_tx_signature: Option<String>,
    burner_offset: Option<u128>,
    proof_account_offset: Option<u128>,
}
```

---

## 5. User Request Schema

### Confidential Claim Request

```rust
struct ConfidentialClaimRequest {
    // === Claim Variant ===
    claim_type: ConfidentialClaimVariant,
    // One of: ExistingMxe, ExistingShared, NewMxe, NewShared

    // === Receiver ===
    receiver_address: Pubkey,
    // For new token variants, the token account will be created

    // === Token ===
    mint: Pubkey,

    // === Offsets (all user-generated, must be unique) ===
    computation_offset: u64,
    proof_account_offset: u128,
    mpc_callback_data_offset: u128,
    unified_fees_pool_account_offset: u128,
    burner_account_offset: u128,
    mixer_tree_account_offset: u128,

    // === UTXO Data (1-4 UTXOs) ===
    utxos: Vec<ConfidentialUtxoData>,   // len: 1..=4

    // === ZK Proof ===
    groth16_proof_a: [u8; 64],      // G1 point
    groth16_proof_b: [u8; 128],     // G2 point
    groth16_proof_c: [u8; 64],      // G1 point

    // === Rescue Encryption Data ===
    rescue_encryption_public_key: [u8; 32],
    encryption_nonce: u128,
    rescue_encrypted_total_amount_ciphertext: [u8; 32],
    rescue_encrypted_relayer_commission_fee_ciphertext: [u8; 32],
    rescue_encrypted_protocol_commission_fee_ciphertext: [u8; 32],
    encrypted_master_viewing_key_low: [u8; 32],
    encrypted_master_viewing_key_high: [u8; 32],
    rescue_encrypted_random_blinding_factor_low_ciphertext: [u8; 32],
    rescue_encrypted_random_blinding_factor_high_ciphertext: [u8; 32],
    rescue_encryption_commitment: [u8; 32],     // PoseidonHash
    encryption_validation_polynomial: [u8; 32], // FieldElement25519

    // === Merkle Root ===
    merkle_root: [u8; 32],

    // === Fees ===
    total_relayer_fees: u64,        // SOL lamports

    // === TVK Timestamp ===
    tvk_timestamp: i64,

    // === Priority Fees ===
    priority_fees: u64,

    // === Optional Data ===
    optional_data: [u8; 16],

    // === New Token Variants Only ===
    random_generation_seed: Option<[u8; 32]>,
}

struct ConfidentialUtxoData {
    slot_index: u32,
    nullifier: [u8; 32],                     // PoseidonHash
    linker_encryptions: [[u8; 32]; 6],       // 6 PoseidonHash values
    linker_key_commitments: [[u8; 32]; 6],   // 6 PoseidonHash values
}
```

### Public Claim Request

```rust
struct PublicClaimRequest {
    // === Receiver ===
    receiver_address: Pubkey,

    // === Token ===
    mint: Pubkey,

    // === Offsets ===
    computation_offset: u64,
    proof_account_offset: u128,
    mpc_callback_data_offset: u128,
    unified_fees_pool_account_offset: u128,
    utxo_burner_account_offset: u128,
    mixer_tree_index: u128,

    // === UTXO Data (exactly 1 UTXO) ===
    utxo: PublicUtxoData,

    // === ZK Proof ===
    groth16_proof_a: [u8; 64],
    groth16_proof_b: [u8; 128],
    groth16_proof_c: [u8; 64],

    // === Rescue Encryption Data (subset — no amount/fee ciphertexts) ===
    rescue_encryption_pubkey: [u8; 32],
    rescue_encryption_nonce: u128,
    rescue_encrypted_master_viewing_key_low: [u8; 32],
    rescue_encrypted_master_viewing_key_high: [u8; 32],
    rescue_encrypted_blinding_factor_low: [u8; 32],
    rescue_encrypted_blinding_factor_high: [u8; 32],
    rescue_encryption_commitment: [u8; 32],
    encryption_validation_polynomial: [u8; 32],

    // === Merkle Root ===
    merkle_root: [u8; 32],

    // === Amounts & Fees ===
    amount: u64,
    relayer_fixed_sol_fees: u64,

    // === TVK Timestamp ===
    tvk_timestamp: i64,

    // === Priority Fees ===
    priority_fees: u64,

    // === Optional Data ===
    optional_data: [u8; 16],

    // === Fee Merkle Proofs (Public Claims Only) ===
    protocol_fees_amount_lower_bound: u64,
    protocol_fees_amount_upper_bound: u64,
    protocol_fees_base_fees_in_spl: u64,
    protocol_fees_commission_fee_in_spl: u64,
    protocol_fees_merkle_path: [[u8; 32]; 4],
    protocol_fees_leaf_index: u32,

    relayer_fees_amount_lower_bound: u64,
    relayer_fees_amount_upper_bound: u64,
    relayer_fees_base_fees_in_spl: u64,
    relayer_fees_commission_fee_in_spl: u64,
    relayer_fees_merkle_path: [[u8; 32]; 4],
    relayer_fees_leaf_index: u32,
}

struct PublicUtxoData {
    slot_index: u32,
    nullifier: [u8; 32],
    linker_encryptions: [[u8; 32]; 5],       // 5 PoseidonHash values (not 6)
    linker_key_commitments: [[u8; 32]; 5],
}
```

---

## 6. DynamoDB Schema

### Table: `umbra-relayer-claims`

**Partition Key:** `request_id` (String, UUID)

**Attributes:**

```
request_id:           S   (UUID, PK)
status:               S   (accepted | validating | validated | rejected |
                           building_txs | sending_burner | sending_proof |
                           sending_claim | awaiting_callback | callback_done |
                           cleaning_up | completed | failed | failed_cleaned)
claim_type:           S   (confidential_existing_mxe | confidential_existing_shared |
                           confidential_new_mxe | confidential_new_shared | public_mxe)
mint:                 S   (Pubkey base58)
receiver_address:     S   (Pubkey base58)
created_at:           N   (unix millis)
updated_at:           N   (unix millis)
ttl:                  N   (unix seconds, DynamoDB TTL for auto-cleanup)

# Offsets (for dedup and cleanup)
computation_offset:               N
proof_account_offset:             S  (u128 as string)
mpc_callback_data_offset:         S  (u128 as string)
unified_fees_pool_account_offset: S  (u128 as string)
burner_account_offset:            S  (u128 as string)

# Transaction signatures (populated as txs land)
burner_tx_signatures:     L  (list of strings)
proof_account_tx_sig:     S
claim_tx_sig:             S
cleanup_tx_sig:           S

# Computation tracking
computation_account_addr: S  (Pubkey base58)

# Error info (populated on failure)
error_message:            S
error_stage:              S  (validation | burner | proof | claim | callback | cleanup)

# Callback result
callback_detected_at:     N  (unix millis)
```

### Table: `umbra-relayer-active-offsets`

**Purpose:** Authoritative offset deduplication. Prevents two concurrent claims from using the same PDA seeds.

**Partition Key:** `offset_key` (String)

Format: `{offset_type}:{offset_value}` — e.g., `proof:12345678901234567890`

```
offset_key:     S   (PK, e.g. "computation:12345")
request_id:     S   (which claim owns this offset)
created_at:     N   (unix millis)
ttl:            N   (unix seconds, auto-expire after 1 hour)
```

**Write pattern:** `PutItem` with `attribute_not_exists(offset_key)` condition. If the condition fails, the offset is already in use.

### Table: `umbra-relayer-config` (optional)

For storing relayer operational state that survives restarts:

```
config_key:     S   (PK, e.g. "last_fee_collection_tx_count:USDC")
config_value:   S
updated_at:     N
```

---

## 7. Internal Queue Patterns (Within Each Service)

Each microservice uses tokio::mpsc channels for internal work distribution. This provides:
- Backpressure (bounded channels prevent memory exhaustion)
- Concurrency control (worker pool size = max concurrent operations)
- Graceful shutdown (drop sender, workers drain)

### Generic Pattern

```rust
use tokio::sync::mpsc;

struct ServiceRuntime<T: Send + 'static> {
    // Bounded channel for backpressure
    work_tx: mpsc::Sender<T>,
    // Worker handles for graceful shutdown
    workers: Vec<JoinHandle<()>>,
}

impl<T: Send + 'static> ServiceRuntime<T> {
    fn new(
        worker_count: usize,
        channel_capacity: usize,
        handler: impl Fn(T) -> BoxFuture<'static, Result<()>> + Clone + Send + 'static,
    ) -> Self {
        let (work_tx, work_rx) = mpsc::channel::<T>(channel_capacity);
        let work_rx = Arc::new(Mutex::new(work_rx));

        let workers = (0..worker_count)
            .map(|_| {
                let rx = work_rx.clone();
                let handler = handler.clone();
                tokio::spawn(async move {
                    loop {
                        let item = {
                            let mut rx = rx.lock().await;
                            rx.recv().await
                        };
                        match item {
                            Some(work) => {
                                if let Err(e) = handler(work).await {
                                    tracing::error!("Worker error: {e}");
                                }
                            }
                            None => break, // Channel closed, shutdown
                        }
                    }
                })
            })
            .collect();

        Self { work_tx, workers }
    }
}
```

### Transaction Pipeline — Multi-Stage Internal Channels

```rust
// Within the Transaction Pipeline Service:

struct TxPipelineInternal {
    // Stage 1: Burner creation
    burner_tx: mpsc::Sender<BurnerWork>,
    // Stage 2: Proof account creation
    proof_tx: mpsc::Sender<ProofWork>,
    // Stage 3: Claim submission
    claim_tx: mpsc::Sender<ClaimWork>,
    // Output: completed claims
    output_tx: mpsc::Sender<CompletedWork>,
}

// Each stage's worker, on completion, sends work to the next stage's channel.
// This creates a pipeline where each claim flows through stages sequentially,
// but multiple claims can be in different stages concurrently.

// Stage 1 Worker (N=20 concurrent):
async fn burner_worker(work: BurnerWork, next: mpsc::Sender<ProofWork>) {
    // Build and send 1-4 burner update txs sequentially
    for utxo_batch in work.utxo_batches() {
        let tx = build_burner_update_tx(&work, &utxo_batch);
        send_and_confirm(tx).await?;
        update_db_status(&work.request_id, "sending_burner").await;
    }
    // Forward to next stage
    next.send(ProofWork::from(work)).await?;
}

// Stage 2 Worker (N=20 concurrent):
async fn proof_worker(work: ProofWork, next: mpsc::Sender<ClaimWork>) {
    let tx = build_proof_account_tx(&work);
    send_and_confirm(tx).await?;
    update_db_status(&work.request_id, "sending_proof").await;
    next.send(ClaimWork::from(work)).await?;
}

// Stage 3 Worker (N=20 concurrent):
async fn claim_worker(work: ClaimWork, output: mpsc::Sender<CompletedWork>) {
    let tx = build_claim_tx(&work);
    let sig = send_and_confirm(tx).await?;
    update_db_status(&work.request_id, "awaiting_callback").await;
    output.send(CompletedWork { request_id: work.request_id, claim_tx_sig: sig, ... }).await?;
}
```

### Callback Monitor — Watch Registration Channel

```rust
struct CallbackMonitorInternal {
    // New watch registrations from SQS consumer
    register_tx: mpsc::Sender<WatchRegistration>,
    // Completed watches (ready for SQS publish)
    completed_tx: mpsc::Sender<WatchResult>,
}

// Single registration consumer + single Laserstream gRPC stream handler
// Multiple SQS publishers (fan-out from completed_tx)
```

---

## 8. User Notification Strategy

**Recommended: REST Polling with Webhook Option**

**Primary: Polling Endpoint**

```
GET /v1/claims/{request_id}
```

Response:

```json
{
    "request_id": "550e8400-e29b-41d4-a716-446655440000",
    "status": "awaiting_callback",
    "claim_type": "confidential_existing_mxe",
    "mint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
    "created_at": 1710288000000,
    "updated_at": 1710288030000,
    "transactions": {
        "burner": ["5UBe...sig1", "3Kpq...sig2"],
        "proof_account": "7Xmn...sig3",
        "claim": "9Abc...sig4",
        "cleanup": null
    },
    "error": null
}
```

**Why polling over SSE/WebSocket:**

- **Privacy:** No persistent connection to correlate requests. Each poll is independent.
- **Scalability:** Stateless. Any API Gateway instance can serve any status query from DynamoDB.
- **Reliability:** No connection drops, no reconnection logic. Simple HTTP GET.
- **Client simplicity:** `while (status !== "completed") { sleep(2s); poll(); }`

**Secondary (Optional): Webhook**

If the user provides a `callback_url` in the request, the Cleanup Service POSTs the final status:

```json
POST {callback_url}
{
    "request_id": "...",
    "status": "completed",
    "claim_tx_signature": "9Abc...sig4",
    "cleanup_tx_signature": "2Def...sig5"
}
```

**Privacy consideration:** Webhooks reveal the relayer's IP to the callback server. For maximum privacy, polling is preferred.

---

## 9. Horizontal Scaling Design

### Shared Keypair Architecture

All service instances share the same relayer Solana keypair. This works because:

1. **Solana has no account nonces.** Unlike Ethereum, each Solana tx just needs a recent blockhash. Multiple instances can send txs concurrently without nonce sequencing.

2. **PDA collisions are prevented by user-provided offsets.** Each claim uses unique offsets, so no two claims will try to create/close the same PDA.

3. **The relayer keypair is only used for:**
   - Fee payer (signing transactions)
   - PDA seed (relayer key is part of proof account, burner, fees pool PDAs)

### Scaling Characteristics Per Service

- **API Gateway:** Fully stateless. Scale to N instances behind ALB.
- **Preflight Validator:** Stateless (RPC reads only). Scale to N instances. Bottleneck: RPC rate limits.
- **Transaction Pipeline:** Stateless per-claim. Scale to N instances. Bottleneck: RPC send rate, blockhash freshness.
- **Callback Monitor:** Singleton (active-passive HA). Single Laserstream connection handles all monitoring. Bottleneck: transaction parsing throughput (unlikely at 250 TPS).
- **Cleanup Service:** Stateless. Scale to N instances. Low throughput requirement.
- **Admin Service:** Single instance. No scaling needed.

### RPC Load Distribution

```
              +------------------+
              | RPC Load Balancer|
              | (Round-robin     |
              |  across nodes)   |
              +--------+---------+
                       |
          +------------+------------+
          |            |            |
    +-----+-----+ +---+---+ +-----+-----+
    | Helius    | | Helius | | Helius    |
    | Node 1    | | Node 2 | | Node 3    |
    | (read)    | | (read) | | (send)    |
    +-----------+ +--------+ +-----------+
```

- Separate RPC endpoints for reads (getAccountInfo, getMultipleAccounts) vs writes (sendTransaction)
- Read endpoints can be heavily cached (especially for ProgramInformation, Pool, fees configs)
- Write endpoint should use dedicated Helius staked connection for priority landing

---

## 10. Complete Claim Lifecycle (End-to-End)

```
Time    User                API GW          Validator       Tx Pipeline       Monitor         Cleanup
─────────────────────────────────────────────────────────────────────────────────────────────────────
t=0     POST /claims ────>  Validate
                            schema
                            Dedup offsets
                            DB: accepted
                            SQS: claim-req
        <──── 202 {id}

t=1                                         Consume SQS
                                            RPC: fetch
                                            accounts
                                            Check:
                                            - timestamps
                                            - merkle root
                                            - nullifiers
                                            - account state
                                            SQS: validated

t=2                                                         Consume SQS
                                                            Build burner tx 1
                                                            Sign + send
                                                            Confirm
                                                            DB: sending_burner

t=3                                                         Build burner tx 2
                                                            Sign + send
                                                            Confirm

t=4                                                         Build proof tx
                                                            Sign + send
                                                            Confirm
                                                            DB: sending_proof

t=5                                                         Build claim tx
                                                            Sign + send
                                                            Confirm
                                                            DB: awaiting_cb
                                                            SQS: pending-cb

t=6                                                                           Consume SQS
                                                                              Subscribe via
                                                                              Laserstream

t=7..N                                                                        Waiting for
        GET /claims/{id}                                                      MPC callback...
        <──── {status:
         awaiting_callback}

t=N+1                                                                         Laserstream:
                                                                              Finalized!
                                                                              DB: callback_done
                                                                              SQS: completed

t=N+2                                                                                         Consume SQS
                                                                                              Build rent
                                                                                              claim tx
                                                                                              Sign + send
                                                                                              Confirm
                                                                                              Remove offsets
                                                                                              DB: completed

t=N+3   GET /claims/{id}
        <──── {status:
         completed,
         txs: {...}}
```

---

## 11. Error Handling & Retry Strategy

### Transaction Send Failures

```
Attempt 1: Send tx with blockhash B1
  ├─ Simulation error (e.g., InvalidAccountData) → FAIL FAST, mark claim failed
  ├─ Network error → retry
  └─ Timeout → check signature status
        ├─ Confirmed → proceed to next stage
        └─ Not found → Attempt 2

Attempt 2: Rebuild tx with fresh blockhash B2, send
  ├─ Same as above
  └─ Timeout → Attempt 3

Attempt 3: Final attempt with fresh blockhash B3
  └─ Failure → mark claim failed, trigger cleanup
```

### SQS Retry & Dead Letter

Each queue has a DLQ. Messages that fail processing N times go to DLQ for manual inspection.

- `claim-requests` → 3 retries → DLQ
- `validated-claims` → 2 retries → DLQ (fewer retries since tx failures are expensive)
- `pending-callbacks` → 1 retry → DLQ (callback is idempotent to monitor)
- `completed-claims` → 3 retries → DLQ
- `failed-callbacks` → 3 retries → DLQ

### Idempotency

All operations are idempotent by design:
- DynamoDB writes use `request_id` as PK — duplicate writes are no-ops
- Offset dedup uses conditional writes — duplicate attempts fail safely
- `claimComputationRent` on already-closed account fails gracefully
- Burner account creation with existing account fails safely (Anchor `init` checks)

---

## 12. Monitoring & Observability

### Metrics (Prometheus/CloudWatch)

- `relayer_claims_total{status, claim_type, mint}` — Counter
- `relayer_claim_duration_seconds{stage}` — Histogram (per-stage latency)
- `relayer_tx_send_attempts{stage, result}` — Counter
- `relayer_sol_balance_lamports` — Gauge
- `relayer_sqs_queue_depth{queue}` — Gauge
- `relayer_laserstream_connected` — Gauge (0 or 1)
- `relayer_laserstream_last_slot` — Gauge
- `relayer_callback_watches_active` — Gauge
- `relayer_callback_wait_seconds` — Histogram
- `relayer_offsets_active` — Gauge

### Structured Logging (tracing)

All services use `tracing` with JSON output:

```rust
tracing::info!(
    request_id = %request.id,
    claim_type = %request.claim_type,
    stage = "claim_tx_sent",
    tx_signature = %sig,
    "Claim transaction confirmed"
);
```

### Alerts

- SOL balance < 1 SOL → CRITICAL
- SQS queue depth > 1000 → WARNING
- Callback wait time > 30s → WARNING
- DLQ messages > 0 → ALERT
- Laserstream disconnected > 30s → CRITICAL
- Claim failure rate > 5% in 5 min → WARNING

---

## 13. Security Considerations

### Keypair Management
- Store relayer keypair in AWS KMS or HashiCorp Vault
- Each service instance retrieves the keypair at startup via IAM role
- Never log or expose the private key
- Rotate keypair via admin service (requires re-initializing all relayer PDAs)

### Request Validation
- Rate limit per IP: 10 claims/minute
- Rate limit per receiver_address: 5 claims/minute
- Max request body size: 16KB
- All field elements validated as valid BN254 scalars before forwarding
- Offsets validated as non-zero

### DDoS Protection
- ALB with AWS WAF in front of API Gateway
- SQS provides natural buffering against burst traffic
- Bounded internal channels prevent memory exhaustion
- DynamoDB auto-scaling handles read/write spikes

### Privacy
- No logging of ZK proof data, nullifiers, or rescue encryption values
- Request IDs are random UUIDs (no sequential information leak)
- Polling endpoint requires knowing the UUID (unguessable)
- No correlation between claims in logs or metrics

---

## 14. Deployment Architecture

```
AWS Region: us-east-1
├── VPC
│   ├── Public Subnet
│   │   └── ALB → API Gateway (ECS Fargate, 2-10 tasks)
│   │
│   ├── Private Subnet
│   │   ├── Preflight Validator (ECS Fargate, 2-10 tasks)
│   │   ├── Transaction Pipeline (ECS Fargate, 2-10 tasks)
│   │   ├── Callback Monitor (ECS Fargate, 1 active + 1 standby)
│   │   ├── Cleanup Service (ECS Fargate, 1-3 tasks)
│   │   └── Admin Service (ECS Fargate, 1 task)
│   │
│   └── Data Subnet
│       ├── DynamoDB (on-demand capacity)
│       ├── ElastiCache Redis (cluster mode, 2 shards)
│       └── SQS Queues (6 standard + 1 DLQ)
│
├── AWS KMS (keypair encryption)
├── CloudWatch (metrics, logs, alarms)
└── ECR (container registry)
```

Each service is a separate container image with independent scaling policies. ECS auto-scaling based on:
- API Gateway: CPU + ALB request count
- Preflight Validator: SQS queue depth
- Transaction Pipeline: SQS queue depth
- Callback Monitor: fixed (1 active + 1 standby)
- Cleanup Service: SQS queue depth

---

## 15. Crate Structure (Rust)

```
umbra-relayer/
├── Cargo.toml (workspace)
├── crates/
│   ├── relayer-common/          # Shared types, config, DB client, SQS client
│   │   ├── src/
│   │   │   ├── config.rs        # Config file parsing (toml)
│   │   │   ├── types.rs         # ClaimVariant, request/response types
│   │   │   ├── db.rs            # DynamoDB client wrapper
│   │   │   ├── sqs.rs           # SQS producer/consumer
│   │   │   ├── redis.rs         # Redis bloom filter client
│   │   │   ├── rpc.rs           # Solana RPC client (read + send)
│   │   │   ├── pda.rs           # PDA derivation (mirrors SDK umbra.ts)
│   │   │   ├── signing.rs       # Transaction signing (KMS or in-memory)
│   │   │   └── lib.rs
│   │   └── Cargo.toml
│   │
│   ├── relayer-api-gateway/     # Service 1
│   │   ├── src/
│   │   │   ├── main.rs
│   │   │   ├── routes.rs        # HTTP endpoints
│   │   │   ├── validation.rs    # Schema validation
│   │   │   ├── dedup.rs         # Offset deduplication
│   │   │   └── handlers.rs      # Request handlers
│   │   └── Cargo.toml
│   │
│   ├── relayer-validator/       # Service 2
│   │   ├── src/
│   │   │   ├── main.rs
│   │   │   ├── consumer.rs      # SQS consumer loop
│   │   │   ├── checks/
│   │   │   │   ├── timestamp.rs
│   │   │   │   ├── merkle_root.rs
│   │   │   │   ├── nullifiers.rs
│   │   │   │   ├── account_state.rs
│   │   │   │   ├── proof_validity.rs
│   │   │   │   ├── fee_merkle.rs
│   │   │   │   └── mod.rs
│   │   │   └── worker.rs        # Validation worker pool
│   │   └── Cargo.toml
│   │
│   ├── relayer-tx-pipeline/     # Service 3
│   │   ├── src/
│   │   │   ├── main.rs
│   │   │   ├── consumer.rs      # SQS consumer loop
│   │   │   ├── stages/
│   │   │   │   ├── burner.rs    # Stage 1: burner account updates
│   │   │   │   ├── proof.rs     # Stage 2: proof account creation
│   │   │   │   ├── claim.rs     # Stage 3: claim instruction
│   │   │   │   └── mod.rs
│   │   │   ├── builder/
│   │   │   │   ├── confidential.rs  # TX builders for confidential variants
│   │   │   │   ├── public.rs        # TX builder for public variant
│   │   │   │   ├── accounts.rs      # Account list construction
│   │   │   │   └── mod.rs
│   │   │   ├── sender.rs       # TX send + confirm logic
│   │   │   └── pipeline.rs     # Multi-stage pipeline orchestration
│   │   └── Cargo.toml
│   │
│   ├── relayer-callback-monitor/ # Service 4
│   │   ├── src/
│   │   │   ├── main.rs
│   │   │   ├── consumer.rs      # SQS consumer for new watches
│   │   │   ├── laserstream.rs   # Laserstream gRPC client
│   │   │   ├── watcher.rs       # Watch registry + timeout sweeper
│   │   │   ├── decoder.rs       # Computation account status decoder
│   │   │   └── fallback.rs      # RPC polling fallback
│   │   └── Cargo.toml
│   │
│   ├── relayer-cleanup/         # Service 5
│   │   ├── src/
│   │   │   ├── main.rs
│   │   │   ├── consumer.rs      # SQS consumer (2 queues)
│   │   │   ├── rent_claim.rs    # claimComputationRent builder
│   │   │   ├── orphan_cleanup.rs # Close orphaned proof/burner accounts
│   │   │   └── offset_release.rs # Remove offsets from active-offsets table
│   │   └── Cargo.toml
│   │
│   └── relayer-admin/           # Service 6
│       ├── src/
│       │   ├── main.rs
│       │   ├── health.rs        # Health checks & balance monitoring
│       │   ├── init_accounts.rs # Initialize missing relayer accounts
│       │   └── cli.rs           # Admin CLI commands
│       └── Cargo.toml
│
├── config/
│   ├── relayer-config.mainnet.toml
│   └── relayer-config.devnet.toml
│
└── docker/
    ├── Dockerfile.api-gateway
    ├── Dockerfile.validator
    ├── Dockerfile.tx-pipeline
    ├── Dockerfile.callback-monitor
    ├── Dockerfile.cleanup
    └── Dockerfile.admin
```

---

## 16. Key Dependencies (Rust)

```toml
# relayer-common/Cargo.toml
[dependencies]
# Solana
solana-sdk = "2.2"
solana-client = "2.2"
solana-transaction-status = "2.2"
anchor-lang = "0.32.1"

# AWS
aws-sdk-dynamodb = "1"
aws-sdk-sqs = "1"
aws-config = "1"

# Redis
redis = { version = "0.27", features = ["tokio-comp", "cluster"] }

# Async runtime
tokio = { version = "1", features = ["full"] }

# HTTP (API Gateway)
axum = "0.8"
tower = "0.5"

# Serialization
serde = { version = "1", features = ["derive"] }
serde_json = "1"
toml = "0.8"

# Crypto
sha2 = "0.10"          # SHA-256 for PDA seeds
bs58 = "0.5"

# Observability
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["json"] }
prometheus = "0.13"

# gRPC (Laserstream)
tonic = "0.12"
prost = "0.13"

# Misc
uuid = { version = "1", features = ["v4"] }
dashmap = "6"           # Concurrent HashMap for callback monitor
```
