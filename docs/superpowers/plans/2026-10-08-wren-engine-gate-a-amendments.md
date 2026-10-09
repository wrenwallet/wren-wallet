# Wren engine plan — Gate A amendments

**Date:** 2026-10-08
**Applies to:** [`2026-09-28-wren-engine.md`](2026-09-28-wren-engine.md) (sub-plan 1 of 3)
**Why:** the Gate A review by @ca-ruz and @chrisguida. Every finding, its verdict and where it is resolved: [`../reviews/2026-10-08-gate-a.md`](../reviews/2026-10-08-gate-a.md).

**How to use this document.** These amendments are binding. Each task in the plan carries a pointer to its section here. Read the task, then its amendments; **where they conflict, the amendment wins.** Each task's amendments are grouped by review area. Within a task, apply the areas in the order shown; later areas build on interfaces defined by earlier ones (for example, Area B's `ServerError`, Area C's `Dialer`, Area D's `custody` module). New failing tests are written before the code they test, as in the rest of the plan.

**Before Task 5 starts:** the pinned constants under "Owner data" must be captured from a fork node and cross-checked against a BTC Electrum server.

## Product decisions

### Decided by the owner
- Claim success copy is SCOPED: "Done. The N coins listed are now split: their BTC and XBT are separate." + list of coins not claimed (with reason) + "Claim before you spend BTC from this wallet." Empty scan: "This server found no XBT for this account" + reasons it may miss coins + "This doesn't prove the wallet has none."
- Claim REQUIRES Tor, on a separate Arti-isolated Electrum session (scan + broadcast), never the hot-wallet session.
- Restore of a seed with pre-split history: BLOCK, do not store the seed. History found on a later sync: block all sending except one "send all" after acknowledgement.
- PIN: 6 digits by default + optional longer PIN (8-12 digits or alphanumeric passcode) in Settings. Honest risk wording.

### Engineering defaults adopted (owner may veto)
- Pre-split coins on a server without a cp_height proof: "unverified" (not counted as confirmed, not claimable). Claim therefore needs a cp_height-capable server (Fulcrum).
- No second-server tip cross-check in v1 (no API field). Expose tip_time; flag stale tip when tip time is more than 2 h behind wall clock.
- Nest guidance: 6 confirmations (Nest plan, not engine).
- Fees: MAX_ESTIMATE_RATE 500 sat/vB (estimates above -> floor + unreliable flag); MAX_FEE_RATE 1,000 sat/vB hard cap in payment, bump and claim; abnormal_fee when fee >= 10% of amount or rate > 200 sat/vB -> second confirmation in the app.
- Balance: WrenBalance { spendable, incoming_pending, immature }. Own unconfirmed change is spendable; unconfirmed coins from foreign parents are not.
- Broadcast outcomes: Sent / Rejected(kind) / Unknown. Replace offered only for the wallet's own unconfirmed tx and re-spends the same inputs.
- No raw server error text anywhere, no details view.
- Receive: MAX_UNUSED_RECEIVE = 10; after 10 unused revealed addresses Wren shows the oldest unused one. Spec wording changes from "fresh address each time".
- Self-send allowed, review shows to_self notice.
- Uneconomic claim coins skipped and reported; no "include anyway".
- Plaintext TCP only to .onion hosts (over Tor) and loopback on regtest. No LAN plaintext.
- TLS pin = SPKI SHA-256; CertificateChanged error; re-pin only with explicit user consent; no Start9 root-CA pinning in v1.
- Tor bootstrap deadline 120 s; any direct (non-Tor) fallback is user-initiated only.
- Electrum history cap 20,000 items per script; exceeding it is an error, never a truncation.
- Native PIN pad and native send-confirm sheet (Swift/Kotlin). Seed words cross JS only at Create, Restore, Reveal, and three of them typed back at the backup check. Engine locks on app background and after 5 min idle.
- Argon2id ~1 s per guess on slowest supported phone; keep the existing wait schedule incl. 24 h tail.
- Wallet from an earlier install: offer unlock with its PIN or explicit typed-confirmation delete; never silent wipe.
- No extra BIP44/49/86 probing on restore (privacy).
- Fresh destination per claim transaction (reused across rebuilds in one session); no per-coin mode in v1; review screen discloses clustering.
- SeedSigner: real-firmware fixtures + headless firmware PSBT parser in CI; emulator run at Gate B (amend spec §8 accordingly).
- MAX_CLAIM_PSBT_BYTES 50 kB placeholder, MAX_UR_FRAGMENTS 2,000; only_witness_utxo only after the firmware maintainer confirms.
- Bare-address scan/paste warning; label the coin "XBT (BLAKE2b chain)" in copy.
- UR import with any unsupported output: refuse the whole import, naming the type.
- CI: `check` and `integration` are required checks; arm64 runners; actions pinned by SHA.
- Core 29.1 used as the standard signer / legacy-chain oracle: recorded as a deviation in Task 22.
- Pinned constants that need the owner's fork node (header-merkle root at 961,631, chainwork at 961,632, times 961,622-961,631, real retarget tuples): plan carries placeholders + exact capture procedure; they must be filled and cross-checked against a BTC Electrum server before Task 5 starts.

## Cross-area reconciliation (binding; overrides the area blocks below where they differ)

The five areas were drafted in parallel. A consistency pass then resolved every clash between them. This section is the final word on shared types and signatures; the numbered resolutions after it explain each change.

### `EngineError` (error.rs; Task 1 declares all but the last seven, Task 12a appends those)
```rust
/// Variants never carry server-supplied text, and their messages contain no keys, addresses, amounts or txids.
#[derive(Debug, Error)]
pub enum EngineError {
    #[error("invalid block header: {0}")] Header(&'static str),
    #[error("header chain rejected: {0}")] Chain(&'static str),
    #[error("server is not on the XBT (BLAKE2b) chain")] WrongChain,
    #[error("signing refused: {0}")] Signing(&'static str),
    #[error("server error: {0}")] Server(ServerError),
    #[error("network error: {0}")] Network(&'static str),
    #[error("wallet error: {0}")] Wallet(&'static str),
    #[error("not enough XBT for this payment and its fee")] InsufficientFunds { available: Amount, needed: Amount },
    #[error("invalid input: {0}")] Input(&'static str),
    #[error("claim refused: {0}")] Claim(&'static str),
    #[error("not connected to a verified XBT server")] NotVerified,
    #[error("the operation was cancelled")] Cancelled,
    #[error("the server's certificate key changed")] CertificateChanged { spki_sha256: [u8; 32], cert_sha256: [u8; 32] },
    #[error("the signer made a normal signature; nothing was sent")] DangerousSignature,
    // Task 12a:
    #[error("Wren is locked")] Locked,
    #[error("wrong Wren PIN")] WrongPin,
    #[error("authorization was cancelled")] AuthCancelled,
    #[error("too many wrong PINs; try again later")] LockedOut { retry_after_secs: u64 },
    #[error("confirm your recovery phrase before sending")] BackupNotConfirmed,
    #[error("this recovery phrase was used for bitcoin before the split")] SeedHoldsBtc,
    #[error("secure storage is not available")] KeyVault,
}
#[derive(Debug, Clone, Copy, PartialEq, Eq, Error)]
pub enum ServerError {
    #[error("the server sent data Wren could not read")] Malformed,
    #[error("the server sent too much data")] TooLarge,            // reply/batch/unsolicited budgets
    #[error("the server is behind the chain")] Behind,
    #[error("the server returned no headers")] NoHeaders,
    #[error("the server sent an invalid transaction")] InvalidTransaction,
    #[error("the server speaks an unsupported Electrum version")] UnsupportedProtocol,
    #[error("an address has more history than Wren can load")] HistoryTooLarge,
    #[error("{0}")] Rejected(RejectKind),                           // JSON-RPC error reply; never poisons
}
// RejectKind: unchanged from Task 1 Area B.
```
Constructor rules:
- `electrum::bad() -> Server(Malformed)`.
- `net(_) -> Network("could not reach the server")`.
- Deadline → `Network("the server did not answer in time")`.
- Poisoned → `Network("connection lost")`.
- No `.into()`, `format!` or `to_string()` in any `EngineError` constructor.

### Task 20 public API (engine.rs)
```rust
pub struct EngineConfig { pub data_dir: PathBuf, pub params: ChainParams, pub server: ServerAddress, pub tor: bool /* initial value only */ }
#[derive(Clone, Debug, Default)]
pub struct ConnectionStatus { pub connected: bool, pub verified: bool, pub tip_height: Option<u32>, pub tip_time: Option<u64>,
    pub stale_tip: bool, pub protocol: Option<String>, pub route: Option<Route>, pub tor_progress: Option<u8> }
pub const STALE_TIP_SECS: u64 = 2 * 60 * 60;
pub const IDLE_LOCK: Duration = Duration::from_secs(300);
pub const CLAIM_NEEDS_TOR: &str = "Claim needs Tor. Turn on Tor in Settings.";
#[derive(Clone, Copy, Debug, PartialEq, Eq)] pub enum WalletPresence { None, Ready, SealedSeedOnly }
#[derive(Clone, Debug)] pub struct RestoreCheck { pub tx_count: usize }
#[derive(Clone, Debug, Default)] pub struct WalletFlags { pub backup_confirmed: bool, pub btc_history_detected: bool, pub btc_sweep_available: bool }
#[derive(Clone, Debug)] pub struct ClaimReviewTx { pub destination: String, pub inputs: usize, pub amount: Amount, pub fee: Amount, pub qr_frames: Vec<String> }
#[derive(Clone, Debug)] pub struct ClaimReview { pub claim_id: u64, pub total_in: Amount, pub total_fee: Amount, pub total_out: Amount,
    pub transactions: Vec<ClaimReviewTx>, pub not_claimed: Vec<NotClaimed>, pub abnormal_fee: bool }

impl WrenEngine {
    pub fn new(cfg: EngineConfig) -> Result<Self, EngineError>;              // installed vault else KeyVault; RoutePolicy(cfg.server, cfg.tor); corrupt headers.bin -> headers.bin.corrupt
    #[cfg(any(test, feature = "test-hooks"))] pub fn with_vault(cfg: EngineConfig, vault: Arc<dyn KeyVault>) -> Result<Self, EngineError>;
    // setup and custody
    pub async fn wallet_presence(&self) -> Result<WalletPresence, EngineError>;
    pub async fn create_wallet(&self) -> Result<Vec<String>, EngineError>;
    pub async fn restore_wallet(&self, words: String, passphrase: String) -> Result<(), EngineError>;
    pub async fn restore_check(&self) -> Result<RestoreCheck, EngineError>;        // cancellable(abort_net)
    pub async fn finish_setup(&self) -> Result<(), EngineError>;
    pub async fn abandon_setup(&self);
    pub async fn backup_challenge(&self) -> Result<[u8; 3], EngineError>;
    pub async fn confirm_backup(&self, answers: Vec<String>) -> Result<bool, EngineError>;
    pub async fn unlock(&self) -> Result<(), EngineError>;
    pub async fn lock(&self);                                                       // bumps abort_claim + abort_net first
    pub async fn is_unlocked(&self) -> bool;
    pub async fn reveal_words(&self) -> Result<Vec<String>, EngineError>;
    pub async fn change_pin(&self) -> Result<(), EngineError>;
    pub async fn wallet_flags(&self) -> Result<WalletFlags, EngineError>;
    pub async fn acknowledge_btc_seed(&self) -> Result<(), EngineError>;
    // network
    pub async fn set_server(&self, server: ServerAddress) -> Result<(), EngineError>;
    pub async fn set_tor(&self, on: bool) -> Result<(), EngineError>;
    pub async fn disconnect(&self);
    pub async fn connect(&self) -> Result<ConnectionStatus, EngineError>;          // cancellable(abort_net)
    pub async fn sync(&self) -> Result<(), EngineError>;                           // cancellable(abort_net); wallet only when unlocked
    pub async fn connection_status(&self) -> ConnectionStatus;                     // reads `status` snapshot only
    pub async fn probe_certificate(&self, host: &str, port: u16) -> Result<CertInfo, EngineError>;
    pub async fn fee_estimates(&self) -> Result<FeeEstimates, EngineError>;        // cancellable(abort_net)
    // wallet
    pub async fn balance(&self) -> Result<WrenBalance, EngineError>;
    pub async fn transactions(&self) -> Result<Vec<TxSummary>, EngineError>;
    pub async fn next_receive_address(&self) -> Result<String, EngineError>;
    pub async fn build_payment(&self, destination: &str, amount: Option<Amount>, send_all: bool, fee_rate: FeeRate) -> Result<ReviewSummary, EngineError>;
    pub async fn sign_and_broadcast(&self, review_id: u64) -> Result<BroadcastOutcome, EngineError>; // vault-authorized; not cancellable; broadcast outside the state lock
    pub async fn bump_fee(&self, txid: Txid, fee_rate: FeeRate) -> Result<ReviewSummary, EngineError>;
    pub fn parse_payment_link(&self, s: &str) -> Result<PaymentLink, EngineError>;
    pub fn make_payment_link(&self, address: &str, amount: Option<Amount>) -> Result<String, EngineError>; // + require_payable
    // claim
    pub fn claim_supported_types(&self) -> Vec<ClaimScriptType>;
    pub async fn claim_start(&self, xpub: ScannedPayload) -> Result<u64, EngineError>;
    pub async fn claim_scan(&self, claim_id: u64, search_further: bool) -> Result<FoundSummary, EngineError>; // cancellable(abort_net, abort_claim)
    pub async fn claim_build(&self, claim_id: u64, destination: Option<&str>, fee_rate: FeeRate) -> Result<ClaimReview, EngineError>;
    pub async fn claim_finish(&self, claim_id: u64, signed: Vec<ScannedPayload>) -> Result<Vec<BroadcastOutcome>, EngineError>; // slot dropped only if all Sent
    pub async fn claim_cancel(&self, claim_id: u64);
    pub async fn claim_active(&self) -> bool;
    // crate / test
    pub(crate) async fn connect_with_stream(&self, stream: BoxStream, route: Route) -> Result<ConnectionStatus, EngineError>;
    #[cfg(test)] pub(crate) fn with_test_dialer(self, f: Arc<dyn Fn(Purpose) -> Arc<dyn Dialer> + Send + Sync>) -> Self;
    #[cfg(any(test, feature = "test-hooks"))] pub async fn debug_allow_pre_split_history(&self);
    #[cfg(feature = "test-hooks")] pub async fn debug_sign_review(&self, review_id: u64) -> Result<Transaction, EngineError>;
}
```
Removed: `load_wallet`.

Claim check order: `touch` (Locked) → Tor rule (`st.tor || claim_direct_allowed(server)`, else `Claim(CLAIM_NEEDS_TOR)`) → `verified` → session id.

State and fields:
```rust
pub struct WrenEngine {
    cfg: EngineConfig,
    vault: Arc<dyn KeyVault>,
    st: tokio::sync::Mutex<State>,
    status: std::sync::Mutex<ConnectionStatus>,   // snapshot; written by publish()
    abort_net: tokio::sync::watch::Sender<u64>,   // set_server, set_tor, disconnect, lock
    abort_claim: tokio::sync::watch::Sender<u64>, // claim_cancel, lock
    #[cfg(feature = "tor")] tor: tokio::sync::OnceCell<Arc<Tor>>,
    #[cfg(test)] test_dialer: Option<Arc<dyn Fn(Purpose) -> Arc<dyn Dialer> + Send + Sync>>,
}
struct State {
    server: ServerAddress, tor: bool, chain: HeaderChain,
    client: Option<Arc<ElectrumClient>>, verified: bool, route: Option<Route>,
    wallet: Option<WrenWallet>, pending: PendingPayments,
    claim: Option<ClaimSlot>, next_claim_id: u64,
    setup: Option<PendingSetup>, last_activity: Option<tokio::time::Instant>,
    challenge: Option<[u8; 3]>, allow_pre_split_history: bool,
}
struct ClaimSlot { id: u64, session: ClaimSession, client: Option<Arc<ElectrumClient>> }
struct PendingSetup { seed: SeedSecret, kind: SetupKind, backup_confirmed: bool, checked: bool }
enum SetupKind { Created, Restored }
#[derive(Clone, Copy, PartialEq, Eq)] pub(crate) enum Purpose { Wallet, Claim, Probe }
```

### Shared signatures
```rust
// params.rs (Task 6 A + B + M9)
#[derive(Clone, Debug)]
pub struct ChainParams {
    pub network: Network, pub split_height: u32, pub blake2b_height: u32,
    pub anchor: Checkpoint, pub checkpoints: Vec<Checkpoint>, pub pow_limit: U256,
    pub fixed_bits: Option<u32>, pub blake2b_shift: u32,
    pub anchor_prev_times: Vec<u32>, pub pre_anchor_chain_work: U256, pub min_chain_work: U256,
    pub pre_anchor_root: Option<HeaderCheckpointRoot>,
    pub long_maturity_start: u32, pub long_maturity_end: u32,
    pub coinbase_maturity_long: u32,   // 6_480 mainnet, 100 regtest; consensus long window AND Knots relay rule for every coinbase
}
// builders: with_anchor, with_anchor_prev_times, with_min_chain_work, with_pre_anchor_root, with_long_maturity(start, end, blocks)
// Task 22: ChainParams::regtest(SPLIT).with_long_maturity(210, 330, 120)

// headerchain.rs: original API plus
#[derive(Clone)] pub struct HeaderChain;
pub const MAX_FUTURE_BLOCK_TIME: u64 = 7200;
pub fn extend(&mut self, start: u32, raws: &[Vec<u8>]) -> Result<(), EngineError>;            // atomic, no clock
pub fn extend_at(&mut self, start: u32, raws: &[Vec<u8>], now: u64) -> Result<(), EngineError>; // atomic, 2 h bound
pub fn work_above(&self, height: u32) -> U256;
pub fn chain_work(&self) -> U256;
pub fn meets_min_work(&self) -> bool;
pub fn reached_activation(&self) -> bool;

// sync.rs
pub struct HeaderSyncOutcome { pub tip_height: u32, pub tip_time: u32, pub reorg_from: Option<u32> }
pub async fn sync_headers(client: &ElectrumClient, chain: &mut HeaderChain, known: &[(u32, [u8; 32])], now: u64) -> Result<HeaderSyncOutcome, EngineError>;
pub struct WalletSyncReport { pub pre_split_history: bool, pub unverified_pre_split: Vec<(Txid, u32)> }
pub(crate) struct FetchResult {
    pub update: Update,
    pub own_missing: Vec<Arc<Transaction>>,       // rebroadcast by sync_wallet
    pub unverified_pre_split: Vec<(Txid, u32)>,
    pub held_back: Vec<Arc<Transaction>>,         // txid-checked, never inserted: unverified pre-split, and unconfirmed when !admit_unconfirmed
}
pub(crate) async fn fetch_confirmed_update(client: &ElectrumClient, chain: &HeaderChain, wallet: &Wallet,
    found: &HashMap<Txid, i64>, admit_unconfirmed: bool, now: u64) -> Result<FetchResult, EngineError>;
pub async fn sync_wallet(client: &ElectrumClient, chain: &HeaderChain, wallet: &mut Wallet, now: u64) -> Result<WalletSyncReport, EngineError>;
pub(crate) fn verify_our_p2wpkh_inputs(tx: &Transaction, spent: &[TxOut], ours: &dyn Fn(&Script) -> bool) -> Result<(), EngineError>;
// pre_anchor_header: only Err(Server(ServerError::Rejected(_))) -> Ok(None); all other errors propagate.

// electrum
pub fn verify_merkle(txid: &Txid, proof: &MerkleProof, root: &[u8; 32], txcount: Option<u32>) -> bool;
pub fn verify_header_proof(raw: &[u8], height: u32, branch: &[[u8; 32]], cp: &HeaderCheckpointRoot) -> bool;
impl Rpc {
    pub fn new(stream: BoxStream, poisoned: Arc<AtomicBool>) -> Self;
    pub async fn call(&mut self, method: &str, params: Value) -> Result<Value, EngineError>;
    pub async fn batch<T>(&mut self, calls: &[(&str, Value)],
        decode: impl FnMut(usize, Value) -> Result<T, EngineError>) -> Result<Vec<T>, EngineError>;
    // error reply -> Server(Rejected(classify_reject(msg))) without poisoning; every other early return, including a decode Err, poisons
}
pub(crate) fn classify_reject(msg: &str) -> RejectKind;
impl ElectrumClient {
    pub fn is_poisoned(&self) -> bool;
    pub async fn header_with_proof(&self, height: u32, cp_height: u32) -> Result<(Vec<u8>, Vec<[u8; 32]>), EngineError>;
    pub async fn transaction_known(&self, txid: &Txid) -> Result<bool, EngineError>;
    pub async fn relay_fee(&self) -> Result<Option<FeeRate>, EngineError>;
    pub async fn estimate_fee(&self, blocks: u32) -> Result<Option<FeeRate>, EngineError>;
}

// wallet.rs
impl WrenWallet {
    pub(crate) fn create(db_path: &Path, network: Network, keys: &KeySource) -> Result<Self, EngineError>;
    pub fn open(db_path: &Path, network: Network) -> Result<Self, EngineError>;
    pub fn exists(db_path: &Path) -> bool;
    pub fn fingerprint(&self) -> Fingerprint;
    pub fn flag(&self, f: MetaFlag) -> Result<bool, EngineError>;
    pub fn set_flag(&mut self, f: MetaFlag, v: bool) -> Result<(), EngineError>;
    pub fn sweep_txid(&self) -> Result<Option<Txid>, EngineError>;
    pub fn set_sweep_txid(&mut self, t: Txid) -> Result<(), EngineError>;
    pub fn balance(&self, params: &ChainParams) -> WrenBalance;
    pub fn next_receive_address(&mut self) -> Result<Address, EngineError>;   // MAX_UNUSED_RECEIVE = 10
    pub fn next_claim_address(&mut self) -> Result<Address, EngineError>;     // always fresh; refuses if unused run > STOP_GAP - 1
}

// payment.rs
pub struct PendingPayments; // id -> (Psbt, ReviewKind, ReviewSummary); kind(), unsigned_txid(), summary()
pub fn bump_fee(w: &mut WrenWallet, params: &ChainParams, tip: u32, pending: &mut PendingPayments, txid: Txid, fee_rate: FeeRate) -> Result<ReviewSummary, EngineError>;
pub(crate) fn sign_pending(keys: &KeySource, psbt: &Psbt) -> Result<Transaction, EngineError>;
pub(crate) fn sign_and_record(w: &mut WrenWallet, keys: &KeySource, pending: &mut PendingPayments, review_id: u64, now: u64) -> Result<Transaction, EngineError>;
pub(crate) async fn broadcast_recorded(client: &ElectrumClient, tx: &Transaction) -> BroadcastOutcome;
pub async fn sign_and_broadcast(w: &mut WrenWallet, keys: &KeySource, pending: &mut PendingPayments, review_id: u64,
    client: &ElectrumClient, now: u64) -> Result<BroadcastOutcome, EngineError>;  // = sign_and_record + broadcast_recorded
pub(crate) fn map_create_tx(e: CreateTxError) -> EngineError;

// claim
pub enum NotClaimedReason { Unconfirmed, Uneconomic, TooLargeForQr, Immature, Unverified }
pub struct NotClaimed { pub script_type: ClaimScriptType, pub reason: NotClaimedReason, pub coins: usize, pub amount: Amount }
pub struct FoundSummary { pub total: Amount, pub coins: usize, pub by_type: Vec<(ClaimScriptType, Amount, usize)>,
    pub not_claimable: Vec<NotClaimed>, pub gap_used: u32, pub types_scanned: Vec<ClaimScriptType> }
pub struct ClaimSession { pub(crate) network: Network, pub(crate) wallets: Vec<(ClaimScriptType, Wallet)>,
    pub(crate) plan: Option<ClaimPlan>, pub(crate) own_destinations: Vec<Address>, pub(crate) sent: Vec<bool> }
pub struct ClaimTx { pub script_type: ClaimScriptType, pub destination: Address, pub psbt: Psbt, pub inputs: usize,
    pub amount: Amount, pub fee: Amount, pub qr_frames: Vec<String> }
pub struct ClaimPlan { pub txs: Vec<ClaimTx>, pub total_in: Amount, pub total_fee: Amount, pub total_out: Amount,
    pub not_claimed: Vec<NotClaimed>, pub abnormal_fee: bool }
pub enum ClaimDestination { OwnWallet, Elsewhere(Address) }
impl ClaimSession {
    pub async fn scan(&mut self, client: &ElectrumClient, chain: &HeaderChain, search_further: bool, now: u64) -> Result<FoundSummary, EngineError>;
    pub fn build(&mut self, params: &ChainParams, tip: u32, dest: ClaimDestination,
        next_own: &mut dyn FnMut() -> Result<Address, EngineError>, fee_rate: FeeRate) -> Result<&ClaimPlan, EngineError>;
    pub(crate) fn build_with_limits(&mut self, params: &ChainParams, tip: u32, dest: ClaimDestination,
        next_own: &mut dyn FnMut() -> Result<Address, EngineError>, fee_rate: FeeRate,
        max_inputs: usize, max_bytes: usize) -> Result<&ClaimPlan, EngineError>;  // refuses once any sent[i]
    pub async fn finish(&mut self, signed: Vec<ScannedPayload>, client: &ElectrumClient) -> Result<Vec<BroadcastOutcome>, EngineError>;
}
// ClaimSession.client and connect_isolated are removed. SkippedCoin, SkipReason and ClaimPlan.skipped are removed.

// testutil.rs (MockWorld; MockWorld::new initialises every field)
pub struct MockWorld {
    pub chain: TestChain, pub history: HashMap<String, Vec<(Txid, i64)>>, pub txs: HashMap<Txid, Transaction>,
    pub tx_override: HashMap<Txid, Transaction>, pub merkle: HashMap<Txid, (u32, u32, Vec<[u8; 32]>)>,
    pub broadcasts: Vec<Transaction>,
    pub relay_fee: f64, pub fee_estimate: f64, pub broadcast_mode: BroadcastMode,      // B: 0.00001, 0.00002, Accept
    pub pre_anchor: Vec<Vec<u8>>, pub pre_anchor_served: HashMap<u32, Vec<u8>>, pub cp_supported: bool, // A: cp_supported = true
    pub log: Vec<(u32, String, Value)>,                                                // C
}
pub enum BroadcastMode { Accept, Reject(String), RecordThenFail, LieOk }
pub fn serve(world: Arc<Mutex<MockWorld>>) -> BoxStream;                        // = serve_tagged(world, 0)
pub fn serve_tagged(world: Arc<Mutex<MockWorld>>, tag: u32) -> BoxStream;       // carries A's cp arm and B's fee/broadcast arms
pub fn serve_stalling(world: Arc<Mutex<MockWorld>>, answer_first_n: usize) -> BoxStream;
pub fn abandon_keys() -> KeySource;                                              // Task 12a; used by pre-Task-20 wallet tests
// mine, mine_bits, TEST_NOW (Task 6); header_tree, TestChain::params_with_pre_anchor (Task 13); Funded.keys (Task 14)
```

### Conflict resolutions (numbered; binding)

#### A. `EngineError` and error text

1. **Task 1 B vs C/E/D, variants declared in three places.** B rewrites the enum, C adds `Cancelled`/`CertificateChanged`, E adds `DangerousSignature`, D (12a) adds seven more.
   - **Resolution:** declare each variant exactly once. Task 1 holds the base set plus the B, C and E variants; Task 12a appends D's seven (final enum below).
   - Give `Cancelled` and `AuthCancelled` different messages; both are currently "cancelled".
2. **`Server("...".into())` left in Area A (Task 11) and Area C (Task 9), which no longer compiles under `Server(ServerError)`.**
   - "server is behind the chain" → `ServerError::Behind`; "server returned no headers" → `NoHeaders` (B already supersedes these two A sites).
   - "response too large" → `TooLarge`.
   - "server sent too many unsolicited messages" → `TooLarge`.
   - "server speaks an unsupported Electrum protocol version" → new `ServerError::UnsupportedProtocol`.
   - "address history too large" → new `ServerError::HistoryTooLarge`. C's test asserts `Err(Server(ServerError::HistoryTooLarge))`.
   - Unparsable version replies → `Malformed`.
3. **`bad()` helper: B makes it argument-free, but A and C still call `bad("tip")`, `bad("merkle")`, `bad("version")`.**
   - **Resolution:** `fn bad() -> EngineError { EngineError::Server(ServerError::Malformed) }` everywhere in `electrum/`. The `xpub_import` `bad()` (Task 16) is a separate helper and is unaffected.
4. **`Wallet(&'static str)` (B, Task 1) vs `.into()`/`to_string()` callers elsewhere.**
   - Callers: A Task 11 `save_chain` (`Wallet(e.to_string())`); D Task 12a `open_sealed` and `KeySource::from_seed` (5 sites); D Task 20 `open_seed` and `unlock` (3 sites).
   - **Resolution:** drop `.into()` everywhere. `save_chain` uses `|_| EngineError::Wallet("could not save the header chain")`.
5. **`Network(String)` used inconsistently.** C writes both `Network("connection lost")` (a `&str`) and `Network("Tor ...".into())`. Separately, `net(e)` puts io/Arti/rustls text into errors; that text can include host:port and digits, which breaks the doc promise and the "no raw error text" decision.
   - **Resolution:** change it to `Network(&'static str)`.
   - `fn net<E>(_: E) -> EngineError { EngineError::Network("could not reach the server") }`.
   - Timeouts → `Network("the server did not answer in time")`.
   - Poisoned → `Network("connection lost")`.
   - Plus `Network("Tor did not start within 2 minutes")` and `Network("this build does not include Tor")`.
6. **D's `KeyVault`.** The brief mentions `NoKeyVault`; nothing in the file uses that name. Keep `KeyVault`.

#### B. Electrum client, errors and poisoning

7. **Task 13 A `pre_anchor_header` matches `Err(EngineError::Server(_)) => None`.** Under B and C, Malformed, TooLarge and InvalidTransaction all poison the client. A malformed proof is also a "present but wrong" proof and must fail.
   - **Resolution:** match only `Err(EngineError::Server(ServerError::Rejected(_))) => None`, which is the JSON-RPC error reply such as electrs' one-argument `blockchain.block.header`. Every other error propagates.
8. **`Rpc::batch`: B edits the error branch, C replaces the whole body.**
   - **Resolution:** use C's algorithm. Step 7 becomes `Err(Server(ServerError::Rejected(classify_reject(msg))))`, without poisoning.
   - State explicitly that an `Err` from the `decode` closure also poisons, because unread replies desynchronise the stream.

#### C. Header sync and chain parameters

9. **`sync_headers` arity.** C Task 20 `open_claim_client` calls `sync_headers(&client, &mut probe)`, the old 2-argument form.
   - **Resolution:** `sync_headers(&client, &mut probe, &[], now())`.
10. **`ChainParams` maturity (B's `coinbase_relay_maturity` vs E/M9).** Task 22 sets Knots `-testcoinbasematuritylong=210:210:330` (long maturity 120). B's `spendable::immature_coinbase` hard-codes 6,480, and regtest uses relay maturity 100 with a window of `u32::MAX`, so Wren would call a coin mature at 100 while Knots needs 120. The M9 boundary test (`mine(118)` immature, `+1` mature) fails.
    - **Resolution:** replace `coinbase_relay_maturity` with one field `coinbase_maturity_long: u32` (6_480 mainnet, 100 regtest). It is used for both the consensus long window and the relay rule, because Knots relay uses `CoinbaseMaturityLong`.
    - Add the builder `with_long_maturity(start, end, blocks)`.
    - Task 22 `engine()` uses `ChainParams::regtest(SPLIT).with_long_maturity(210, 330, 120)`. Confirm the three flag fields against Knots `chainparams.cpp` before pinning.
    - Rename the B test to `long_maturity_is_6480_on_mainnet_and_100_on_regtest`.
11. **`HeaderChain: Clone` is defined in both Task 6 A and Task 6 C.** This is a duplicate, not a conflict; keep A's line and delete C's block.
12. **Owner-data timing.** The product decision says the constants are needed "before Task 6 is implemented", but Task 5 Step 0 already needs `mainnet-retargets.txt`.
    - **Resolution:** "before Task 5" everywhere (header and Owner-data heading already say so). Fix the decision bullet and the review "Data still needed before Task 6" line.

#### D. Sync, broadcast and the Task 13/14 shape

13. **Task 13 has three different return shapes for `fetch_confirmed_update`/`sync_wallet`.** A: `VerifiedUpdate`, args unchanged, sync returns `WalletSyncReport`. B: new args, `FetchResult`, sync returns `()`. D: sync returns `SyncReport`.
    - **Resolution:** use B's arguments and a merged `FetchResult` (see the reconciliation section), with `sync_wallet -> WalletSyncReport`. Drop D's `SyncReport`.
    - Compute `pre_split_history` from the reported heights (D's rule; A's is equivalent).
    - Duplicate tests: drop A's `report_flags_pre_split_history`; keep D's two.
    - The Task 19 B retry also uses `client.transaction_known(&txid).await?`. After a broadcast timeout, C's poisoned client fails at once and the `?` loses the per-transaction outcomes and the `sent` flags. **Resolution:** call `transaction_known(..).await.unwrap_or(false)`; on any error keep `Unknown(txid)`.
14. **Unconfirmed and unverified amounts for the claim report (B vs E vs A).**
    - B: the claim scan never fetches unconfirmed transactions and counts them in `unconfirmed_ignored`.
    - E: removes `unconfirmed_ignored` and lists `NotClaimed{Unconfirmed, coins, amount}`, which needs the amount.
    - A: skips fetching unverified pre-split transactions, but E wants `Unverified` entries with amounts.
    - **Resolution:** `fetch_confirmed_update` still fetches (txid-checked) both kinds but never inserts them. It returns them in `FetchResult.held_back`, and `scan` builds `NotClaimed` entries from the outputs that pay the scanned scripts.
    - Remove `unconfirmed_ignored` and `unconfirmed_seen`.
    - Rewrite B's `claim_scan_ignores_server_invented_unconfirmed_spend` to assert `found.coins == 1` and that `not_claimable` has no `Unconfirmed` entry (the fake spend pays a foreign script).
15. **Engine `sign_and_broadcast` return type.** D Task 20 returns `Result<Txid>` and uses `out_is_definite_rejection`; B returns `BroadcastOutcome`, and D's own Task 14 says to keep B's type. D also holds the state lock across `client.broadcast`, which can take up to 90 s. That contradicts C's rule that `set_server`/`set_tor`/`disconnect` never wait on the state lock.
    - **Resolution:** return `Result<BroadcastOutcome>`.
    - Split the payment function into `payment::sign_and_record(..) -> Result<Transaction>` (under the lock: sign, `apply_unconfirmed_txs`, persist) and `payment::broadcast_recorded(client, &tx) -> BroadcastOutcome` (lock released, `Arc` client cloned).
    - Keep `payment::sign_and_broadcast` as the composition for the Task 14 tests.
    - `claim_finish` likewise does `st.claim.take()`, releases the lock, runs `finish`, then re-locks and puts the slot back only if not every outcome is `Sent`. Drop it if `lock()` or a new `claim_start` happened in between.
16. **BTC-seed one-send-all (D) vs rejected or lost broadcasts (B).** B keeps a `Rejected` or `Unknown` transaction recorded with its inputs locked, and Speed up is the only recovery. D refuses `bump_fee` while `BtcHistoryDetected` is set, so a rejected or stuck sweep can never move.
    - **Resolution:** set `BtcSweepUsed = true` and store `btc_sweep_txid` in `wren_meta` (new `WrenWallet::{sweep_txid, set_sweep_txid}`) under the lock right after `sign_and_record` succeeds. This fails safe.
    - `btc_seed_gate` also allows `ReviewKind::Bump` when the bumped txid equals `sweep_txid`, and updates `sweep_txid` to the replacement after signing.
    - D's test: `bump_fee` of a non-sweep transaction gives `SeedHoldsBtc`; bump of the sweep is allowed.
17. **`payment::review_summary(w, &pending, id)` (D Task 20) is defined nowhere.**
    - **Resolution:** `PendingPayments` stores `(Psbt, ReviewKind, ReviewSummary)`, with `pub(crate) fn summary(&self, id) -> Option<&ReviewSummary>`.
    - `SendApproval::from` maps `to_self` and `abnormal_fee` unconditionally; B adds them in Task 14, so drop D's "when those fields exist" hedge.
    - `map_create_tx` (Task 14) must be `pub(crate)`; Task 18 calls it.

#### E. Claim (Tasks 17-20)

18. **Fresh destination per claim transaction (E, product decision) vs `MAX_UNUSED_RECEIVE` (B).** E's closure `|| wallet.next_receive_address()` returns the oldest unused address once 10 are revealed, so several claim transactions would share one address.
    - **Resolution:** new `WrenWallet::next_claim_address(&mut self) -> Result<Address, EngineError>`. It always calls `reveal_next_address(External)` and persists.
    - It refuses with `Claim("this claim needs too many new addresses; claim fewer coins at once")` if the unused run would exceed `STOP_GAP - 1` (19), so restore still finds every payment.
    - The `claim_build` closure calls it; `own_destinations` gives reuse across rebuilds.
19. **`ClaimSession` client ownership: three designs.** C: `ClaimSlot.client` in the engine, `scan`/`finish` take `&ElectrumClient`. E: `ClaimSession.client` plus `finish(signed)` with no client and `connect_isolated()`. B: `finish(&[..], client)`.
    - **Resolution:** keep C's `ClaimSlot{id, session, client: Option<Arc<ElectrumClient>>}` and `open_claim_client`.
    - Delete `ClaimSession.client` and `connect_isolated`.
    - `finish(&mut self, signed: Vec<ScannedPayload>, client: &ElectrumClient) -> Result<Vec<BroadcastOutcome>>`, using E's `Vec` (needed for zeroize) and B's return type.
20. **Restore order in Task 22.** Area D restores at height 121 (`restore_wallet` → `restore_check` → `finish_setup` after `mine(101)`, now `mine(121)` per E). Area A's rules cannot verify a chain below activation (200), so `restore_check` and `sync_to_tip` always fail. The original suite also needs the engine wallet to exist before the split to get `wren_addr0`.
    - **Resolution:** derive `wren_addr0` in the test from `master(WREN_WORDS)` at m/84'/0'/0'/0/0 (the test already derives `key0` there) and fund it with `fund(...)` before the split.
    - After `mine(SPLIT + 5 - height())`: `e.debug_allow_pre_split_history().await; sync_to_tip(&e).await; e.restore_wallet(WREN_WORDS.into(), "".into()).await?; e.restore_check().await?; e.finish_setup().await?; sync_to_tip(&e).await;`.
    - The allow flag must come before `restore_check`, which honours it; D placed it after.
    - A's Task 22 block: delete `early.restore_wallet(WREN_WORDS, "")`. It is not needed and uses the old signature.
    - Mechanical updates in Task 22 (no area covers them):
      - `.confirmed` → `.spendable`;
      - `let txid = e.sign_and_broadcast(..)?` → `let BroadcastOutcome::Sent(txid) = .. else { panic!() }`, likewise for `slow`/`fast`;
      - `claim_finish(..)` → `Vec<BroadcastOutcome>` instead of `txids[0]`;
      - E Step 6 asserts `Ok(BroadcastOutcome::Sent(_))`.
21. **Task 18 B vs E `ClaimPlan` and build shape.**
    - **`build` signature:** merge B's `(params, tip, &dest, fee_rate)` and E's `(ClaimDestination, next_own, fee_rate)` into one (see the reconciliation section). Name the internal variant `build_with_limits` (E).
    - **Skip reporting:** B's `SkippedCoin`/`SkipReason`/`ClaimPlan.skipped` vs E's `NotClaimed`/`not_claimed`. Use E's types only; immature coins become `NotClaimedReason::Immature`, aggregated per (type, reason).
    - **Uneconomic threshold:** B's descriptor-derived `input_fee` (164 wu + `max_weight_to_satisfy` + 1 wu for taproot, rounded up) vs E's fixed 272/364/592/231 WU table. Keep B's `input_fee`.
    - **Build errors:** `build_one`/`sweep` return `Result<Psbt, CreateTxError>`. `CoinSelection(_) | OutputBelowDustLimit(_)` → that chunk is `Uneconomic`; anything else → `map_create_tx`.
    - **Fee check:** the first statement is `check_fee_rate(fee_rate)?`. Remove E's separate "MAX_FEE_RATE next to the floor" line.
    - **Empty-plan error:** keep E's reason-dependent wording (not B's single message), and keep B's `abnormal_fee`.
    - **Duplicate test name:** both areas have `uneconomic_coins_are_skipped_and_reported`. Keep E's; rename B's P2PKH-500 case to `uneconomic_p2pkh_coin_is_reported_per_type` and assert `not_claimed == [NotClaimed{P2pkh, Uneconomic, 1, 500 sat}]`.
    - **B×E gap:** after any `sent[i]` is true, `build` refuses with `Claim("part of this claim was sent; finish or cancel it")`. Otherwise a rebuild re-spends the same inputs and resets `sent`.
22. **Task 20 `claim_build`: B's `default_destination` and `.build(params, tip, &dest, ..)` vs E's `OwnWallet` and `own_destinations`.** Use E's design with #18 and the merged signature.
    - `ClaimReview` merges B's `abnormal_fee` with E's `not_claimed` and per-transaction `destination`; drop B's `skipped`.
    - Duplicate: drop B's `claim_build_reuses_destination_across_rebuilds`; keep E's `claim_build_reuses_own_destinations_on_rebuild`.
23. **Claim Tor rule: E vs C.** E checks `self.cfg.tor` and exempts all of regtest. C uses the current `st.tor` and exempts only regtest with a loopback `Tcp` server (`claim_direct_allowed`).
    - **Resolution:** C's rule, with E's `CLAIM_NEEDS_TOR` constant replacing C's literal.
    - Check order in `claim_scan`/`claim_finish`: `touch` (`Locked`), then the Tor rule, then `verified` (`NotVerified`), then the session id.
24. **Tor test duplicates that cannot run as written.** C `claim_requires_tor_on_mainnet` and E `claim_scan_refuses_without_tor_on_mainnet` both need "a restored wallet" on mainnet params. Under D, restore needs `restore_check` on a verified connection, and A's mainnet checkpoints, activation and min-work cannot be met by a MockWorld.
    - **Resolution:** one test, `claim_requires_tor_unless_regtest_loopback`. Regtest params with `server: Tls{host:"electrum.example.com", port:50002, pinned_spki_sha256:None}` and `tor:false`, then connect via `connect_with_stream`, `setup_restored`, `claim_start`.
    - `claim_scan` gives `Claim(CLAIM_NEEDS_TOR)`, and the test dialer records no `Purpose::Claim` dial.
    - Also merge E `claim_uses_its_own_connection` into C's `claim_scan_uses_its_own_session` and `claim_finish_broadcasts_on_the_claim_session`. Keep E's `claim_cancel_and_finish_drop_the_claim_connection`.

#### F. Engine lifecycle and locking (Task 20)

25. **`lock()` (D) vs C's no-wait rules.** `lock()` takes the state lock, which `claim_scan`/`sync` can hold for minutes, so "lock on app background" would hang.
    - **Resolution:** `lock()` first does `abort_claim.send_modify(|g| *g += 1)` and `abort_net.send_modify(..)`, then locks and clears.
    - Add `restore_check` to the `cancellable(abort_net, ..)` list; it does network sync under the lock.
26. **`new` / `with_vault` / the test dialer.** D: `new` needs the installed vault; C: `new` runs the `RoutePolicy` check and has a `#[cfg(test)] test_dialer` field with no setter; A: `new` sets aside a corrupt `headers.bin`.
    - **Resolution:** `new` does all three: vault, then policy, then header store.
    - Add `#[cfg(test)] pub(crate) fn with_test_dialer(self, f) -> Self`.
    - Every engine test helper uses `WrenEngine::with_vault(cfg{server: Tcp{"127.0.0.1",1}}, TestVault::new())`.
27. **Global `OnceLock` vault.** D's `install_key_vault_succeeds_once` (Task 12a, inline) installs the process-global vault in the same lib test binary as D's `new_without_installed_vault_fails` (Task 20, inline), so the result depends on test order.
    - **Resolution:** move `install_key_vault_succeeds_once` to its own integration binary `tests/vault_install.rs`.
28. **`ConnectionStatus`: A adds `tip_time`/`stale_tip`; C removes `tor`, adds `route` and an optional `tor_progress`.** C's spec change makes Tor progress mandatory on Home.
    - **Resolution:** use the struct in the reconciliation section and make `tor_progress` required.
    - Compute `stale_tip` at read time in `connection_status()` from the snapshot's `tip_time`: `now().saturating_sub(t) > STALE_TIP_SECS`. Saturating, because a tip may be up to 2 h in the future. If computed in `publish()`, the snapshot goes stale.

#### G. Ordering and duplicates

29. **Task 1 E `check.sh` uses `--features test-hooks`, but D defines `[features] test-hooks = []` in Task 12a.** Cargo errors on the unknown feature from Task 1 on.
    - **Resolution:** define the feature in Task 1 Step 8's `Cargo.toml`. Task 12a only uses it.
30. **Task 16 A and E both rewrite L4565-4567 and both add path tests.**
    - **Resolution:** use E's code (`comps.is_empty() || odd`, `chunks_exact`, `try_from`), plus A's `try_from` for the parent fingerprint (L4570) and master fingerprint (L4586).
    - Drop A's `ur_path_with_odd_length_is_refused` and `ur_path_index_beyond_u32_is_refused` (covered by E's `ur_odd_length_or_oversized_component_refused`); keep A's `ur_fingerprint_beyond_u32_is_refused`.
31. **Tests written before D's Task 12 change break.** Tests in Tasks 12 (B), 13 (B), 14 and 17-19 build a `WrenWallet` from words with the old `create(.., words, passphrase)`.
    - **Resolution:** add `testutil::abandon_keys() -> KeySource` (Task 12a) and call `WrenWallet::create(&p, net, &abandon_keys())`.
32. **Task 22a uses crate-private items from an integration test.** `tests/seedsigner_fixtures.rs` names `SigClass` (pub(crate)) and `verify_and_finalize`.
    - **Resolution:** assert `Err(EngineError::DangerousSignature)` instead of `SigClass::Dangerous`.
    - Re-export `read_signed_psbt`, `verify_and_finalize` and `ClaimTx::from_fixture` under `#[cfg(feature = "test-hooks")]`, and run that test with `--features test-hooks`.

#### H. Spec text

33. **Spec §5 Home is replaced by both A and C.** Merged: "A status line showing the route actually in use (Tor or direct), the server connection, Tor start-up progress, and a warning when the server's latest block is more than 2 hours old."
    - Spec §4.1 Scanning is replaced by both C and E: use E's text, with C's "The scan and the broadcast run on their own Tor circuit".
    - §3.1 should also list `setServer`, `setTor`, `disconnect` and `probeCertificate`.
34. **Product decision "Seed words cross JS only at Create, Restore, Reveal" vs D's `confirm_backup(answers: Vec<String>)`.** That call sends 3 seed words across the bridge at the backup check.
    - **Resolution (owner to confirm):** keep the API and amend the decision and spec §3.2 to "...and three of them typed back at the backup check". The alternative is to make backup confirmation native.

#### I. Labels and counts

35. **Mislabelled areas.**
    - E Task 18 "Area D's MAX_FEE_RATE" → Area B.
    - E Task 17 `Immature /* Area D */, Unverified /* Area B */` → Immature is Area B, Unverified is Area A.
    - E Task 22 "WrenBalance.immature (Area D) and Area D destination refusal" → Area B.
    - The final "Merge notes" line "Unverified ... from Areas D and B" → Area A.
    - **Merged test counts:** Task 6: 19; Task 9: 22; Task 12: 6; Task 13: 14; Task 16: about 23; Task 18: 13; Task 20: 37.

**Intra-area issues noticed (not cross-area)**
- **D `wallet_presence`:** an orphan `wallet.sqlite` with no sealed blob reports `None`, but `finish_setup` then fails on create every time. `finish_setup` should rename the orphan to `wallet.sqlite.orphan` (never delete it).
- **A `verify_merkle`:** `proof.merkle.len() as u32` should be `u32::try_from(..).ok() != Some(depth)`, per A's own no-`as` rule.

---

## Global sections (constraints, review focus, file structure, coverage map, research)

### Area A — headers, sync, chain selection

**Global Constraints:** append these bullets.
- Header acceptance matches Knots `ContextualCheckBlockHeader` for everything a header alone can show:
  - link to the previous header;
  - v2 at and after 961,640;
  - header height field; reserved flags;
  - version ≥ 4 with the v2 flag masked;
  - exact difficulty (`fixed_bits` on regtest);
  - proof of work;
  - time later than the median of the previous 11, using pinned pre-anchor times;
  - at most 2 h in the future (network path only);
  - checkpoints.
- Chain selection: a server's branch replaces ours only after it fully verifies and has strictly more cumulative work above the fork point. A tie keeps ours. Reorgs deeper than 144 are refused. A rejected branch leaves our chain unchanged.
- A connection is `verified` only when all three hold:
  - the server's header at 961,640 is v2 and matches the checkpoint;
  - our verified tip has reached it;
  - our chain's cumulative work is at least Knots `nMinimumChainWork`.
- Confirmations below 961,632 count only with a `cp_height` header proof against the pinned root at 961,631. Without one they are "unverified": never in the balance and never claimable.
- Every integer from a server is range-checked (`try_from`, `checked_*`). There is no `as` narrowing of server data.

**Task 6, wrong-chain guard paragraph** (replaces L1182):
> **The wrong-chain guard lives here and in Task 11.** A legacy-chain server fails in one of three places, whichever comes first:
> 1. its header at 961,632 does not hash to the pinned anchor;
> 2. on every connect, `probe_activation` (Task 11) finds its header at 961,640 is not a v2 header matching the pinned checkpoint;
> 3. while extending, its header at 961,640 fails the same checks.
>
> All three return `EngineError::WrongChain`. A server whose tip is below activation, or whose chain has less work than Knots' `nMinimumChainWork`, is refused with `EngineError::Chain`. Nothing downstream can send without a `HeaderChain` that passed these checks (enforced in Tasks 14 and 20).

**Review Focus:**
- Replace item 2 with:
  > 2. **A reorg across the verified tip while the app was closed, or a server offering a different branch.** Expected: header sync finds the fork point (up to 144 blocks), verifies the server's branch on a copy, and adopts it only if it has strictly more work. A weaker, equal-work or invalid branch is refused and our chain is unchanged. Deeper than 144 is refused. Tests in Task 11 (`sync_handles_reorg_within_window`, `sync_refuses_reorg_deeper_than_window`, `weaker_branch_is_refused`, `equal_work_branch_is_refused`, `invalid_candidate_leaves_chain_intact`).
- Append item 6:
  > 6. **A server inventing a pre-split confirmation.** Expected: below the anchor, a confirmation needs a `cp_height` header proof against the pinned root. A wrong proof fails the sync. No proof leaves the coin unverified (not in balance, not claimable). Tests in Task 13 (`fabricated_pre_anchor_header_is_not_confirmed`, `pre_anchor_payment_without_cp_support_is_unverified`).
- Append item 7:
  > 7. **A server below activation, or a legacy-chain server against a saved chain.** Expected: not verified, or `WrongChain`, with the chain unchanged. Tests in Task 11 (`server_below_activation_is_not_verified`, `legacy_server_with_saved_chain_is_wrong_chain`) and Task 22.

**File Structure:** add these lines.
```
        testutil.rs                      #[cfg(test)] header mining (Task 6), synthetic chains + mock Electrum server (Tasks 11, 13)
      tests/
        vectors/mainnet-retargets.txt    real fork retargets: height prev_bits first_time prev_time expected_bits (owner node)
        fixtures/mainnet-961622-961631.hex  the 10 headers below the anchor (time context; they link into the anchor)
        fixtures/cp-proof-900000.json    recorded blockchain.block.header(900000, 961631) reply from the default server
    scripts/header_merkle_root.py        Electrum header-merkle root at a cp_height, recomputed from a node's block hashes
```

**Coverage map:**
- Replace the row `| §4 wrong-chain guard | 6, 11, 20, 22 |` with `| §4 wrong-chain guard (activation probe on every connect, most-work selection, minimum chain work) | 5, 6, 11, 20, 22 |`.
- Add `| §3.1 merkle proofs incl. pre-split (cp_height) and v2 tx count | 9, 13 |`.
- Add `| §5 status line: stale-tip warning | 20 (UI in sub-plan 2) |`.

**Research doc** `research/2026-09-28-fork-consensus-facts.md`: append to "## Block header":
- **Contextual header rules Wren mirrors** (`validation.cpp:4706-4778`):
  - `nBits == GetNextWorkRequired`;
  - time > median of the previous 11 (`:4737`);
  - time ≤ now + 2 h (`MAX_FUTURE_BLOCK_TIME`, `chain.h:29`; `:4753`; effective time);
  - masked `nVersion ≥ 4` once BIP34/66/65 are active (`:4757-4762`). `nVersion` is stored without the v2 flag (`primitives/block.h`, `SER_READ … v & ~VERSION_HEADER_V2_FLAG`).
- **Regtest:**
  - powLimit `7fff…`, `fPowAllowMinDifficultyBlocks`, `fPowNoRetargeting` (`chainparams.cpp:623-628`). So every regtest header has bits `0x207fffff`.
  - `Blake2bTargetShift` defaults to 20 (`consensus/params.h:141`); only mainnet sets 22 (`chainparams.cpp:134`).
  - BIP34/65/66 are active from height 1.
- **Chain work:**
  - `GetBlockProof = ~t/(t+1)+1` (`chain.cpp:131-144`).
  - Mainnet `nMinimumChainWork = 0x…13e00277374c9f9eeadc70200` (`chainparams.cpp:149`; `:150` is `defaultAssumeValid`).
- **Electrum `cp_height`** (protocol 1.4+):
  - supported by upstream Fulcrum and privkeyio `v2.1.2-blake-3` (`Servers.cpp:1299-1320`; leaves = `BTC::HeaderHash(raw)`, `Storage.cpp:5177-5183`; branch and root are sent in display order);
  - **not** supported by romanz electrs (`electrum.rs:242/631/657`, one-argument `blockchain.block.header`);
  - fork electrs builds not checked.
  - History up to 961,631 is shared, so the root at `cp_height 961,631` is identical on BTC Electrum servers.
- **Pinned values** (owner-captured; table with value, date and the two sources): `PRE_ANCHOR_HEADER_ROOT`, `PRE_ANCHOR_CHAIN_WORK`, `KNOTS_CHAINWORK_961632`, `ANCHOR_PREV_TIMES`, `mainnet-retargets.txt` sha256.

### Area B — server trust in everyday use

- **Global Constraints, replace L29** (`- Fee floor: 1 sat/vB (min relay fee 1000 sat/kvB).`) with:
  - Fee rates: floor 1 sat/vB, hard cap `MAX_FEE_RATE` = 1,000 sat/vB in payment, bump and claim. Server estimates and relayfee above `MAX_ESTIMATE_RATE` (500 sat/vB) are ignored: the floor is used and `FeeEstimates.unreliable` is set. A review sets `abnormal_fee` when the fee is ≥ 10% of the amount or the rate is > 200 sat/vB.
- **Global Constraints, add:**
  - The server is untrusted for unconfirmed data.
    - An unconfirmed spend of one of the wallet's coins enters the graph only with a valid `0x21` P2WPKH signature from the wallet key.
    - Unconfirmed coins from foreign parents are never spendable and are reported as `incoming_pending`.
    - Only the wallet's own unconfirmed txs can be bumped or replaced, and a replacement re-spends the same inputs.
  - No server-supplied text ever reaches an `EngineError` or the UI. Server failures map to fixed `ServerError` / `RejectKind` variants.
  - Every signed payment is recorded and persisted before it is broadcast. A broadcast reports `Sent`, `Rejected(kind)` or `Unknown`, and Wren rebroadcasts its own pending txs on sync.
  - Payment destinations are limited to P2PKH, P2SH, P2WPKH, P2WSH and P2TR, with a scriptPubKey of at most 34 bytes.
- **Review Focus, replace #4** (L42):
  - **Coinbase outputs under consensus and relay maturity.** Consensus: a coinbase created at height ≥ 973,440 needs 6,480 confirmations while the spend height is < 979,920, and 100 otherwise. Knots relay policy needs 6,480 for every coinbase, at any height (`validation.cpp:1021-1023`, commit `c60c36d704`).
  - Expected: a coin is spent only when it is mature under both, in payments, fee bumps and claims alike.
  - Tests: Task 14 `relay_policy_coinbase_is_not_spendable` and `bump_fee_never_adds_an_immature_coinbase`; Task 18 `claim_skips_relay_immature_coinbase`.
- **Review Focus, add #6:**
  - **A server that invents unconfirmed transactions.** Expected: a fake spend of a wallet coin is refused at sync, *Speed up* refuses any tx the wallet key did not sign, and fake incoming payments are never spendable.
  - Tests: Task 13 `server_invented_spend_of_our_coin_is_refused`; Task 14 `bump_fee_refuses_a_tx_our_key_did_not_sign` and `fake_incoming_unconfirmed_is_not_spendable`.
- **Review Focus, add #7:**
  - **A broadcast reply that is lost or faked.** Expected: the tx stays recorded with its inputs locked, its status is reported as `Unknown`, it is rebroadcast on sync, and a claim retry re-sends only the unsent txs.
  - Tests: Task 14 `lost_reply_keeps_payment_pending_and_inputs_locked`; Task 13 `own_unconfirmed_missing_from_server_is_rebroadcast`; Task 19 `claim_finish_retry_skips_sent_and_reports_per_tx_status`.
- **Research `2026-09-28-fork-consensus-facts.md` L85-87, replace with:**
  - Long coinbase maturity, consensus: a coinbase created at ≥ 973,440 needs 6,480 confirmations while the spend height is < 979,920, and 100 otherwise (`src/consensus/tx_verify.cpp`, `src/validation.cpp:3012-3034`).
  - Relay policy: Knots mempool admission and reorg filtering need `CoinbaseMaturityLong` (6,480 on mainnet) for every coinbase, at any height (`src/validation.cpp:443`, `:1021-1023`; commit `c60c36d704` "Policy: Use long maturity for mempool"). Knots' wallet does the same (`src/node/interfaces.cpp:338`).
  - Wren treats a coinbase as spendable only when both rules are met. This affects users who receive coinbase outputs directly, and claims of post-split coinbases.
- **File Structure, add:** `        spendable.rs                     immature_coinbase, untrusted_unconfirmed, is_own_tx (payments, bumps, claims)`
- **Coverage map:**
  - Replace the `§5 send / send all / fee tiers / speed up / activity` row with: `| §5 send / send all / fee tiers (capped) / speed up (own txs only) / activity (confirmations) | 12, 13, 14, 20, 22 |`
  - Add `| §5 Receive: fresh address, bounded by MAX_UNUSED_RECEIVE | 12 |`
  - Add `| §5 Home balance: spendable vs incoming vs immature | 12, 20 |`
  - Add `| Server trust: invented spends, fee caps, broadcast outcomes, fixed error text | 1, 9, 13, 14, 19, 20 |`
  - Add `| Destinations limited to relayable, non-anyone-can-spend types | 15, 20 |`

### Area C — network, Tor, TLS, Electrum

- **Findings:** chrisguida H3, Tor/.onion; ca-ruz #6, #9; chrisguida M14 (Electrum part), Electrum version.
- **Supersedes:** In Global Constraints, the line starting "Electrum protocol: negotiate the range `["1.4", "1.8"]`;". Append "; the server's reply must be inside that range (`parse_version`), or the connection is refused."
- **Changes:**
  - Add these Global Constraints bullets:
    - "Network routes: plaintext Electrum only to v3 `.onion` hosts over Tor, or to loopback on regtest. Clearnet servers use TLS: web PKI, or a pinned SPKI SHA-256. Without Tor, a `.onion` host is refused before any name resolution. Every socket (Electrum, certificate probe, claim session) is opened through a `transport::Dialer`, never `TcpStream::connect` directly."
    - "Claim always runs over Tor on its own isolated Tor circuit and Electrum session (scan and broadcast). It never uses the wallet's session. The only exception is regtest with a loopback server (tests)."
    - "No network wait is unbounded. Each Electrum request has a 90 s absolute deadline, each method has a reply size limit, and each batch has a byte budget. Tor bootstrap has a 120 s deadline. The engine's state lock is never the reason `connection_status`, `set_server`, `set_tor`, `disconnect` or `claim_cancel` waits."
  - Review Focus: add item 6, "A hostile or broken Electrum server: endless notifications, silence, oversized replies, out-of-range protocol version. Expected: the request fails at its deadline or limit, the connection is dropped and marked unverified, and cancel or status return at once. Tests in Task 9 (`notifications_do_not_extend_a_request_past_its_deadline`, `history_item_cap_is_an_error_not_a_truncation`, `refuses_protocol_outside_requested_range`) and Task 20 (`status_and_cancel_do_not_wait_for_a_stuck_scan`)."

### Area D — key custody, engine API, PIN

- **Global Constraint L31.** Replace it with:
  > - Private key material (seed, `Mnemonic`, `Xpriv`, `SecretKey`) exists only inside `custody.rs` and `sign.rs`, and only for the duration of one engine call (setup, unlock check, sign, reveal, change PIN). Every copy the engine owns is wiped on success and error paths (`Zeroizing`, `Wiped`/`non_secure_erase`, the bip39 `zeroize` feature), and `clippy.toml` forbids these types elsewhere. This is best effort: Rust moves and copies, rust-bitcoin's BIP32 internals and the bridge's string copies can leave bytes the engine cannot reach.
  > - The engine holds no private key between calls. The seed is stored only as a sealed blob in the native key vault. Every signature needs a fresh `KeyVault::authorize` (spec §4).
  > - Seed words cross the public API only in `create_wallet` (shown once), `restore_wallet` (typed once) and `reveal_words` (Wren PIN). Wren never stores a phrase with pre-split history (spec §4).
- **L3184.** Replace with the Task 12 "Keys:" paragraph above.
- **Coverage map L6727.** Replace with these rows:
  ```
  | §3.1/§3.2/§4 seed custody: sealed seed, key-vault contract, no key held between calls, PIN-only reveal, PIN key helpers | 12a, 12, 20 |
  | §4 app lock in the engine: unlock, lock, 5-minute idle lock, per-send authorization, backup-check gate | 20 |
  | §4 BTC-seed check: restore refused and nothing stored; later detection allows one send-all | 13, 20, 22 |
  | App-side only (sub-plan 2): native key vault (Secure Enclave/Keystore keys, Wren PIN pad, wrong-PIN counter and waits, biometric item, sealed-seed storage, earlier-install prompt), native send-confirm sheet, backup confirm-3-words screens, screenshot blocking and app-switcher cover, keyboard settings, OTA updates off, screens, visual design, SeedSigner real-device test, SeedSigner guide | — |
  ```
- **New dependencies to pin** (`cargo add`, locked by `Cargo.lock`, and covered by `cargo audit`):
  - `chacha20poly1305@0.10`
  - `argon2@0.5`
  - `hkdf@0.12`
  - `sha2@0.10`
  - `p256@0.13` (feature `hash2curve`)
  - `bip39@2.2.2` (feature `zeroize`; feature-only, unified with bdk_wallet's copy)
  - dev: `tokio` feature `test-util`
  - The `test-hooks` feature moves to Task 12a.
- **Review Focus addition (optional item 6):**
  > 6. **JS calls a send without the user.** Expected: refused unless the native vault authorizes it. Test: `send_without_vault_authorization_broadcasts_nothing` (Task 20).

### Area E — Claim, links, QR, tests, CI

- **L47 (Gate A).** Replace "and the two test-reference deviations in Task 22 (Knots instead of Shrike for byte-exact cross-signing; Knots `walletprocesspsbt` as the automated SeedSigner stand-in)" with: "the test-reference deviations in Task 22: Knots instead of Shrike for byte-exact cross-signing; Knots `walletprocesspsbt` as the fork-firmware signer; Core 29.1 as the standard signer and legacy-chain oracle; real SeedSigner firmware via fixtures and the headless parser in Task 22a."
- **Review Focus #3.** Replace with: "**A signed PSBT whose unsigned transaction differs from the one Wren built** (swapped output, added input, changed fee, an input really removed from a 2-input claim). Expected: refused with a specific message before broadcast; nothing sent. Tests in Task 19 (`changed_transaction_is_refused_for_each_kind_of_change`, `nothing_is_broadcast_unless_every_transaction_verifies`)."
- **Review Focus #5.** Append: "Arbitrary UTF-8 (emoji, bidi characters) is an error, never a panic (`non_ascii_input_is_an_error_not_a_panic`)."
- **New Review Focus #6:** "**A signer that returns a standard or BTC-valid signature** (0x01, 64-byte Taproot, or 0x21 over a legacy/BIP143 digest). Expected: `DangerousSignature`, nothing sent, payload zeroized. Tests in Task 19 and Task 22 (Core-signed standard claims for all four types)."
- **New Review Focus #7:** "**A hostile animated QR** (absurd fragment count, endless frames). Expected: refused before decoding. Tests in Task 16."
- **File Structure.**
  - L61: change to `fmt + clippy (incl. test-hooks) + test + audit; CI job "check"`.
  - L95: change to `regtest.rs                       integration suite (feature test-hooks, #[ignore]; run by run.sh / CI job "integration")`.
  - L99: change to `run.sh                             fresh world + headless SeedSigner + regtest suite; fails if 0 tests ran`.
  - Add `.github/workflows/engine.yml` (required jobs `check` and `integration`, arm64).
  - Add `tests/fixtures/seedsigner/` (real-firmware exports and signed PSBTs, SOURCES.md).
  - Add `tests/seedsigner_fixtures.rs` and `integration/seedsigner_sign.py`.
- **Coverage map.**
  - Change "§8 vectors, local test chain, replay proof, wrong-chain proof" to `1, 7, 22 (CI job "integration")`.
  - Add `| §8 run on every build (replay proof, claim tests) | 1 (check.sh), 22 Step 6 (engine.yml, required checks) |`.
  - Add `| §8 SeedSigner signer compatibility | 16 (N1 fix), 22a |`.
  - Add `| §4.1 dangerous signature refusal | 7, 19, 22 |`.
  - Add `| §4.1 claim over Tor, separate session | 20, 21 |`.
  - In the App-side row, change "SeedSigner real-device test" to "SeedSigner device/emulator run at Gate B (22a Step 7)".

## Task 1

### Area B — server trust in everyday use

- **Findings:** M11, Low "Error strings", M10 (outcome type)
- **Supersedes:** in `error.rs`, `    Server(String),` and `    Wallet(String),` together with their `#[error]` lines.
- **New failing tests first:**
  - `errors_display_fixed_text_only` (inline in `error.rs`): `Server(ServerError::Rejected(RejectKind::Other))`, `Wallet("x")` and `InsufficientFunds{..}` must display text with no digits and no server text.
- **Changes:**
  ```rust
  #[derive(Debug, Error)]
  pub enum EngineError {
      // Header, Chain, WrongChain, Signing, Network, Input, Claim, NotVerified unchanged
      #[error("server error: {0}")]
      Server(ServerError),
      #[error("wallet error: {0}")]
      Wallet(&'static str),
      #[error("not enough XBT for this payment and its fee")]
      InsufficientFunds { available: Amount, needed: Amount }, // fields for the UI, never in Display
  }
  #[derive(Debug, Clone, Copy, PartialEq, Eq, Error)]
  pub enum ServerError {
      #[error("the server sent data Wren could not read")] Malformed,
      #[error("the server sent too much data")] TooLarge,
      #[error("the server is behind the chain")] Behind,
      #[error("the server returned no headers")] NoHeaders,
      #[error("the server sent an invalid transaction")] InvalidTransaction,
      #[error("{0}")] Rejected(RejectKind),
  }
  #[derive(Debug, Clone, Copy, PartialEq, Eq, Error)]
  pub enum RejectKind {
      #[error("the fee is too low")] FeeTooLow,
      #[error("it conflicts with another transaction")] Conflict,
      #[error("a coin it spends is missing or already spent")] InputsMissingOrSpent,
      #[error("a mined coin is not mature yet")] PrematureCoinbase,
      #[error("it is not final yet")] NonFinal,
      #[error("an output is too small")] Dust,
      #[error("the fee is too high")] FeeTooHigh,
      #[error("it is already confirmed")] AlreadyConfirmed,
      #[error("the server refused the request")] Other,
  }
  ```
  - Update the doc comment to: "Variants never carry server-supplied text, and their messages contain no keys, addresses, amounts or txids."
- **Interfaces changed:** `ServerError` and `RejectKind` (used by Tasks 9, 11, 14, 19, 20), `Wallet(&'static str)`, and `InsufficientFunds`.

### Area C — network, Tor, TLS, Electrum

- **Findings:** ca-ruz #9; chrisguida Low "Pinned certificates".
- **Supersedes:** in `src/error.rs`, the last variant `NotVerified,`. The two variants below go after it.
- **New failing tests first:** none. Variants are exercised in Tasks 10 and 20.
- **Changes:** add
```rust
    #[error("cancelled")]
    Cancelled,
    #[error("the server's certificate key changed")]
    CertificateChanged { spki_sha256: [u8; 32], cert_sha256: [u8; 32] },
```
  Both carry no keys, addresses or amounts, so the `error.rs` promise holds.
- **Interfaces changed:** `EngineError::Cancelled`, `EngineError::CertificateChanged { spki_sha256, cert_sha256 }`.

### Area E — Claim, links, QR, tests, CI

- **Findings:** ca-ruz #12, chrisguida M3 (new error variant)
- **Supersedes:**
  - `error.rs` enum (L204-225), which ends at `NotVerified,`.
  - `check.sh` (L285-292), from `cargo clippy --all-targets -- -D warnings` to `cargo test --workspace`.
- **New failing tests first:** none. The gate itself is the test.
- **Changes:**
  1. Add to `EngineError`:
     ```rust
     #[error("the signer made a normal signature; nothing was sent")]
     DangerousSignature,
     ```
  2. In `check.sh`, after the existing clippy and test lines:
     ```bash
     cargo clippy --all-targets --features test-hooks -- -D warnings
     cargo test --workspace --features test-hooks --no-run   # regtest.rs must compile on every build
     ```
- **Interfaces changed:** `EngineError::DangerousSignature` (unit variant).

## Task 5

### Area A — headers, sync, chain selection

- **Findings:**
  - ca-ruz #1 (adds the 256-bit work arithmetic)
  - chrisguida M7 (minimum chain work)
  - chrisguida Low "Regtest difficulty" (shift and limit become parameters; more retarget vectors)
- **Supersedes:**
  - L841 `pub fn next_bits(height: u32, prev_bits: u32, period_first_time: i64, prev_time: i64, blake2b_height: u32, no_retarget: bool) -> u32;` and its implementation at L1087.
  - L1080 `fn apply_blake2b_shift(bits: u32) -> u32 {`
  - L1104 `pub fn expected_bits_for(h_height: u32, prev: &ForkHeader, period_first: Option<&ForkHeader>, blake2b_height: u32, no_retarget: bool) -> Option<u32> {`
  - Test call sites L886, L888, L895, L899 (`next_bits(…, 961_640, false)`).
  - Step 4 `Expected: PASS (5 tests).` (L1118)
- **New step before Step 1:**
  - **Step 0:** stop unless `core/wren-engine/tests/vectors/mainnet-retargets.txt` exists, filled by the owner (see Owner data placeholders, item 4).
- **New failing tests first:**
  - `block_proof_matches_knots`: `block_proof(0x1d00ffff) == U256([0x1_0001_0001,0,0,0])` (the genesis chainwork) and `block_proof(0x207fffff) == U256([2,0,0,0])`.
  - `block_proof_of_invalid_compact_is_zero`: compacts `0`, `0x04923456` and `0xff123456` all return `U256::ZERO`.
  - `div_matches_div_u64`: for x in {pow_limit, regtest_pow_limit, 0x1234…} and d in {1, 3, 0xffff, u64::MAX}, `x.div(&U256([d,0,0,0])) == x.div_u64(d)`; also `pow_limit().div(&pow_limit()) == U256::ONE`.
  - `knots_pow_tests_vectors`: Knots `src/test/pow_tests.cpp:17-82` as `next_bits(h, bits, first, last, 961_640, 22, pow_limit())`:
    - `(32256, 0x1d00ffff, 1261130161, 1262152739) → 0x1d00d86a`
    - `(2016, 0x1d00ffff, 1231006505, 1233061996) → 0x1d00ffff`
    - `(68544, 0x1c05a3f4, 1279008237, 1279297671) → 0x1c0168fd`
    - `(46368, 0x1c387f6f, 1263163443, 1269211443) → 0x1d00e1fd`
  - `real_retargets_match_knots`: reads each `height prev_bits first_time prev_time expected_bits` line (bits in hex) from `tests/vectors/mainnet-retargets.txt` and asserts `next_bits(height, prev_bits, first, prev, 961_640, 22, pow_limit()) == expected`.
  - Expected count goes from 5 to 10.
- **Changes:**
  1. Add the following to `U256`:
     ```rust
     pub const ONE: U256 = U256([1, 0, 0, 0]);
     pub fn not(&self) -> U256 { U256([!self.0[0], !self.0[1], !self.0[2], !self.0[3]]) }
     pub fn checked_add(&self, o: &U256) -> Option<U256> {
         let (mut r, mut c) = ([0u64; 4], 0u128);
         for i in 0..4 { let v = self.0[i] as u128 + o.0[i] as u128 + c; r[i] = v as u64; c = v >> 64; }
         (c == 0).then_some(U256(r))
     }
     pub fn saturating_add(&self, o: &U256) -> U256 { self.checked_add(o).unwrap_or(U256([u64::MAX; 4])) }
     pub fn wrapping_sub(&self, o: &U256) -> U256 {
         let (mut r, mut borrow) = ([0u64; 4], false);
         for i in 0..4 {
             let (a, b1) = self.0[i].overflowing_sub(o.0[i]);
             let (a, b2) = a.overflowing_sub(borrow as u64);
             r[i] = a; borrow = b1 || b2;
         }
         U256(r)
     }
     fn bit(&self, i: u32) -> bool { (self.0[(i / 64) as usize] >> (i % 64)) & 1 == 1 }
     /// Full 256-bit restoring division. `d` must be non-zero.
     pub fn div(&self, d: &U256) -> U256 {
         assert!(!d.is_zero());
         let (mut q, mut r) = (U256::ZERO, U256::ZERO);
         for i in (0..256u32).rev() {
             let top = r.0[3] >> 63;            // bit shifted out: remainder is then >= d
             r = r.shl(1);
             r.0[0] |= self.bit(i) as u64;
             if top == 1 || r >= *d { r = r.wrapping_sub(d); q.0[(i / 64) as usize] |= 1u64 << (i % 64); }
         }
         q
     }
     ```
  2. Add the port of Knots `GetBlockProof` (`chain.cpp:131-144`):
     ```rust
     pub fn block_proof(bits: u32) -> U256 {
         let (t, neg, ovf) = set_compact(bits);
         if neg || ovf || t.is_zero() { return U256::ZERO; }
         match t.checked_add(&U256::ONE) {
             Some(d) => t.not().div(&d).saturating_add(&U256::ONE),
             None => U256::ZERO,
         }
     }
     ```
  3. Make the shift and pow limit parameters. Remove the dead `no_retarget` argument; regtest never calls `next_bits` (it uses `ChainParams.fixed_bits`, Task 6).
     ```rust
     fn apply_blake2b_shift(bits: u32, shift: u32, limit: U256) -> u32 // body: `limit` replaces pow_limit(), `shift` replaces BLAKE2B_TARGET_SHIFT
     pub fn next_bits(height: u32, prev_bits: u32, period_first_time: i64, prev_time: i64,
                      blake2b_height: u32, shift: u32, limit: U256) -> u32   // retarget clamp uses .min(limit)
     pub fn expected_bits_for(h_height: u32, prev: &ForkHeader, period_first: Option<&ForkHeader>,
                              blake2b_height: u32, shift: u32, limit: U256) -> Option<u32>
     ```
     Keep `BLAKE2B_TARGET_SHIFT = 22` only as the mainnet value used by `ChainParams::mainnet()`. Update the existing tests to pass `22, pow_limit()`.
- **Interfaces changed:**
  - New: `U256::{ONE, not, checked_add, saturating_add, wrapping_sub, div}` and `pow::block_proof`.
  - New signatures: `next_bits`, `expected_bits_for`.
  - The `no_retarget` parameter is removed.

## Task 6

### Area A — headers, sync, chain selection

- **Findings:**
  - ca-ruz #1 (atomic `extend`, work accounting)
  - ca-ruz #3 (`reached_activation`)
  - ca-ruz #4 (MTP context, future-time bound)
  - chrisguida M7 (minimum chain work)
  - chrisguida Lows: "nVersion floor", "Regtest difficulty", "Pre-anchor headers" (pinned root lives in params)
- **Supersedes:**
  - The params field `pub enforce_difficulty: bool,     // false on regtest (min-difficulty rules)`, both in the interface (L1152) and in Step 1 (L1205). Also `enforce_difficulty: true,` (L1233) and `enforce_difficulty: false,` (L1252).
  - The interface line `pub fn extend(&mut self, start_height: u32, raws: &[Vec<u8>]) -> Result<(), EngineError>;` (L1175). Its semantics change: atomic, and no clock.
  - The paragraph starting `**The wrong-chain guard lives here.**` (L1182). Replacement text is under Global sections.
  - In `check_next`, the difficulty block starting `if p.enforce_difficulty {` (L1560) and the MTP line `let mut times: Vec<u32> = self.headers[n.saturating_sub(MTP_WINDOW)..]…` (L1576).
  - `pub struct HeaderChain {` (no `Clone` today).
  - The `extend` body (L1526-1540).
  - `Expected: PASS (8 tests).` (L1639).
- **New step before Step 1:**
  - **Step 0:** stop unless every `<OWNER:…>` placeholder below is filled and cross-checked.
- **New failing tests first:**
  - `outdated_version_is_rejected`: real header 961,641 with `version = 0x8000_0003` gives `Err(Chain(m))` where `m` contains "version".
  - `mtp_uses_pre_anchor_context`: regtest params anchored at a mined header with time `t0`, plus `with_anchor_prev_times(vec![t0+5000; 10])`. A successor at `t0+1` is rejected; one at `t0+5001` is accepted. The current code accepts both.
  - `future_time_bound_applies_only_with_clock`: `extend_at(…, now)` accepts a header at `now+7200` and rejects one at `now+7201` ("future"). Plain `extend` accepts both.
  - `regtest_bits_must_equal_fixed_bits`: a successor mined with `bits 0x207f_fffe` is rejected; one with `0x207f_ffff` is accepted.
  - `failed_extend_leaves_chain_unchanged`: extending with `[good, good, broken-link]` returns `Err`, and the tip height and hash are unchanged.
  - `chain_work_sums_block_proofs`: a regtest chain of N headers (anchor included) has `chain_work() == 2N`, and `work_above(h) == 2·(tip−h)`.
  - `meets_min_work_gate`: with `with_min_chain_work(U256([50,0,0,0]))`, 20 headers fail the gate and 30 headers pass.
  - `reached_activation_requires_last_checkpoint`: with mainnet activation vectors extended to 961,639 the result is false; extended to 961,640 it is true.
  - `anchor_prev_times_are_authentic`: in `tests/fixtures/mainnet-961622-961631.hex`, each header meets its own SHA256d proof of work, each links to the next, and the last links into the anchor's `prev_hash`. Their `time()` values equal `ChainParams::mainnet().anchor_prev_times`.
  - `mainnet_anchor_chain_work_matches_knots`: after `set_anchor(raws[0])`, `chain_work() == KNOTS_CHAINWORK_961632`. That constant is owner-filled in the test module.
  - Expected count goes from 8 to 18.
- **Changes:**
  1. **`testutil.rs` moves into this task.** Create `core/wren-engine/src/testutil.rs` here, with `#[cfg(test)] pub(crate) mod testutil;` in `lib.rs`.
     - Move `pub fn mine(…)` from Task 11 Step 1 into it.
     - Add `pub fn mine_bits(prev, height, blake2b_height, time, salt, root, bits: u32) -> ForkHeader`. `mine` calls it with `0x207f_ffff`.
     - Add `pub const TEST_NOW: u64 = 1_800_000_000;`.
     - Task 11 Step 1 then *appends* the rest of `testutil.rs` instead of creating the file.
  2. **`params.rs`:**
     ```rust
     #[derive(Clone, Debug, PartialEq, Eq)]
     pub struct HeaderCheckpointRoot { pub cp_height: u32, pub root: [u8; 32] } // Electrum header-merkle root, internal order
     pub struct ChainParams {
         pub network: Network, pub split_height: u32, pub blake2b_height: u32,
         pub anchor: Checkpoint, pub checkpoints: Vec<Checkpoint>, pub pow_limit: U256,
         pub fixed_bits: Option<u32>,      // Some(0x207f_ffff) on regtest (Knots: min-difficulty + no-retarget); None = full retarget rules
         pub blake2b_shift: u32,           // 22 mainnet (chainparams.cpp:134); 20 regtest (consensus/params.h:141)
         pub anchor_prev_times: Vec<u32>,  // effective times of the <=10 headers below the anchor, oldest first
         pub pre_anchor_chain_work: U256,  // Knots chainwork at anchor-1; ZERO when anchored at genesis or synthetic
         pub min_chain_work: U256,         // Knots nMinimumChainWork; ZERO on regtest
         pub pre_anchor_root: Option<HeaderCheckpointRoot>, // proves headers below the anchor (Task 13)
         pub long_maturity_start: u32, pub long_maturity_end: u32,
     }
     ```
     - In `mainnet()`:
       ```rust
       fixed_bits: None,
       blake2b_shift: 22,
       anchor_prev_times: vec![/* <OWNER:ANCHOR_PREV_TIMES> 10 values, heights 961,622..961,631 */],
       pre_anchor_chain_work: U256::from_le_bytes(&internal_from_display("<OWNER:PRE_ANCHOR_CHAIN_WORK>")),
       min_chain_work: U256::from_le_bytes(&internal_from_display("00000000000000000000000000000000000000013e00277374c9f9eeadc70200")), // Knots chainparams.cpp:149
       pre_anchor_root: Some(HeaderCheckpointRoot { cp_height: 961_631, root: internal_from_display("<OWNER:PRE_ANCHOR_HEADER_ROOT>") }),
       ```
     - In `regtest()`: `fixed_bits: Some(0x207f_ffff)`, `blake2b_shift: 20`, `anchor_prev_times: vec![]`, `pre_anchor_chain_work: U256::ZERO`, `min_chain_work: U256::ZERO`, `pre_anchor_root: None`.
     - Add builders for tests and the regtest harness: `with_anchor_prev_times(Vec<u32>)`, `with_min_chain_work(U256)`, `with_pre_anchor_root(HeaderCheckpointRoot)`.
  3. **Step 2 (fixtures):** add exports of `tests/fixtures/mainnet-961622-961631.hex` (`export_headers.py … 961622 961631 …`) and `tests/fixtures/cp-proof-900000.json`. Create `core/scripts/header_merkle_root.py`. The script and its cross-checks are under Owner data placeholders. Record the sources in `tests/fixtures/README.md`.
  4. **`headerchain.rs`:**
     - `#[derive(Clone)] pub struct HeaderChain`
     - `pub const MAX_FUTURE_BLOCK_TIME: u64 = 2 * 60 * 60; // Knots chain.h:29, validation.cpp:4753`
     - Make extension atomic, with an optional clock:
       ```rust
       pub fn extend(&mut self, start: u32, raws: &[Vec<u8>]) -> Result<(), EngineError> { self.extend_inner(start, raws, None) } // stored or test headers: no clock
       pub fn extend_at(&mut self, start: u32, raws: &[Vec<u8>], now: u64) -> Result<(), EngineError> { self.extend_inner(start, raws, Some(now)) }
       fn extend_inner(&mut self, start: u32, raws: &[Vec<u8>], now: Option<u64>) -> Result<(), EngineError> {
           let len0 = self.headers.len();
           let r = self.push_all(start, raws, now); // the old extend body, passing `now` to check_next
           if r.is_err() { self.headers.truncate(len0); self.hashes.truncate(len0); }
           r
       }
       ```
  5. **`check_next(&self, h, hash, height, now: Option<u64>)`.** The order is: link → v2-for-height → reserved flags and height field (all unchanged) → version floor → difficulty → PoW (unchanged) → MTP → future → checkpoint (unchanged). New and changed parts:
     ```rust
     if (h.version & !crate::header::VERSION_HEADER_V2_FLAG) < 4 {           // Knots validation.cpp:4757-4762 (nVersion is stored without the v2 flag)
         return Err(EngineError::Chain("outdated block version"));
     }
     match p.fixed_bits {
         Some(b) => if h.bits != b { return Err(EngineError::Chain("unexpected difficulty")); },
         None => { /* existing retarget block, calling expected_bits_for(height, prev, period_first, p.blake2b_height, p.blake2b_shift, p.pow_limit) */ }
     }
     // PoW check unchanged
     let n = self.headers.len();
     let mut times: Vec<u32> = p.anchor_prev_times.iter().copied()
         .chain(self.headers[n.saturating_sub(MTP_WINDOW)..].iter().map(|x| x.time())).collect();
     let mut times = times.split_off(times.len().saturating_sub(MTP_WINDOW));
     times.sort_unstable();
     if h.time() <= times[times.len() / 2] { return Err(EngineError::Chain("time not after median time past")); }
     if let Some(now) = now {
         if h.time() as u64 > now.saturating_add(MAX_FUTURE_BLOCK_TIME) { return Err(EngineError::Chain("header time too far in the future")); }
     }
     // checkpoint check unchanged
     ```
  6. **Work and activation:**
     ```rust
     pub fn work_above(&self, height: u32) -> U256 {   // sum of block_proof for held headers with height > `height`
         let skip = (height.saturating_sub(self.base_height()) as usize + 1).min(self.headers.len());
         self.headers[skip..].iter().fold(U256::ZERO, |w, h| w.saturating_add(&block_proof(h.bits)))
     }
     pub fn chain_work(&self) -> U256 {                // Knots nChainWork of our tip
         self.headers.iter().fold(self.params.pre_anchor_chain_work, |w, h| w.saturating_add(&block_proof(h.bits)))
     }
     pub fn meets_min_work(&self) -> bool { self.chain_work() >= self.params.min_chain_work }
     pub fn reached_activation(&self) -> bool {
         let need = self.params.checkpoints.iter().map(|c| c.height).max().unwrap_or(0).max(self.params.blake2b_height);
         self.tip_height().is_some_and(|t| t >= need)
     }
     ```
     If `height < base`, `work_above` would count the anchor; callers only pass `agree >= base`.
- **Interfaces changed:**
  - New: `ChainParams.{fixed_bits, blake2b_shift, anchor_prev_times, pre_anchor_chain_work, min_chain_work, pre_anchor_root}` and the builders above.
  - Removed: `ChainParams.enforce_difficulty`.
  - New type: `params::HeaderCheckpointRoot`.
  - New on `HeaderChain`: `Clone`, `extend_at`, `work_above`, `chain_work`, `meets_min_work`, `reached_activation`, and `pub const MAX_FUTURE_BLOCK_TIME`.
  - `testutil::{mine, mine_bits, TEST_NOW}` now originate in Task 6.

### Area B — server trust in everyday use

- **Findings:** M8 + ca-ruz #7
- **Supersedes:** in both `pub struct ChainParams {` blocks (L1145, L1198), add the new field. In `mainnet()` and `regtest(...)`, add it after `long_maturity_end:`.
- **New failing tests first:**
  - `relay_maturity_is_6480_on_mainnet_and_100_on_regtest`.
- **Changes:**
  - Add `pub coinbase_relay_maturity: u32, // Knots mempool: every coinbase needs CoinbaseMaturityLong (validation.cpp:1021-1023)`.
  - Values: mainnet `6_480`, regtest `100`.
- **Interfaces changed:** the new `ChainParams` field, read by `spendable::immature_coinbase` (Task 12) and set by Task 22 if the M9 fix sets `-testcoinbasematuritylong`.

### Area C — network, Tor, TLS, Electrum

- **Findings:** Claim on its own session (needs a separately verified chain copy).
- **Changes:** `HeaderChain` derives `Clone`.
- **Interfaces changed:** `HeaderChain: Clone`.

## Task 7

### Area E — Claim, links, QR, tests, CI

- **Findings:** chrisguida M3 (variant 2: byte 0x21 over a legacy or BIP143 digest)
- **Supersedes:** none (additive).
- **New failing tests first:**
  - `legacy_raw_matches_rust_bitcoin_for_0x01`: equals `SighashCache::legacy_signature_hash(i, spk, 0x01)`.
  - `bip143_raw_matches_rust_bitcoin_for_0x01`: equals `p2wpkh_signature_hash(.., EcdsaSighashType::All)`.
  - `bip143_raw_0x21_differs_from_0x01`: the digests differ, and only the hash-type field of the preimage differs.
  - `raw_digests_refuse_non_all_types`: `0x22`, `0x81` and `0xa1` return `None`.
- **Changes:** add to `sighash.rs`. These digests exist only to recognise signatures that would be valid on BTC, and are never used to sign.
  ```rust
  /// BTC-rule digests with a raw hash-type byte. Only base type ALL without ANYONECANPAY.
  pub fn legacy_sighash_raw(tx: &Transaction, i: usize, script_code: &Script, hash_type: u32) -> Option<[u8; 32]> {
      if hash_type & 0x9f != 0x01 { return None; }
      SighashCache::new(tx).legacy_signature_hash(i, script_code, hash_type).ok().map(|h| h.to_byte_array())
  }
  pub fn bip143_sighash_raw(tx: &Transaction, i: usize, script_code: &Script, value: Amount, hash_type: u32) -> Option<[u8; 32]> {
      if hash_type & 0x9f != 0x01 || i >= tx.input.len() { return None; }
      // version | sha256d(prevouts) | sha256d(sequences) | outpoint | script_code (compact-size) | value | sequence
      // | sha256d(outputs) | locktime | hash_type (u32 LE); return sha256d(preimage)
  }
  ```
- **Interfaces changed:** adds `legacy_sighash_raw` and `bip143_sighash_raw`.

## Task 9

### Area A — headers, sync, chain selection

- **Findings:**
  - chrisguida Low "Merkle proofs ignore the v2 transaction count"
  - ca-ruz test defect ("merkle test expects rejection for the duplicated final leaf")
  - ca-ruz #2 and chrisguida Low "Pre-anchor headers" (proof primitives)
  - chrisguida Low "Integer handling" (`tip` cast)
- **Supersedes:**
  - L2088 and L2404 `pub fn verify_merkle(txid: &Txid, proof: &MerkleProof, merkle_root_internal: &[u8; 32]) -> bool`
  - L2221-2223 `assert!(verify_merkle(&c, &proof_c, &root));` / `let wrong_pos = MerkleProof { pos: 3, ..proof_c.clone() };` / `assert!(!verify_merkle(&c, &wrong_pos, &root));`
  - L2454 `let h = r.get("height").and_then(Value::as_u64).ok_or_else(|| bad("tip"))? as u32;`
  - L2510-2511 `block_height: … as u32,` and `pos: … as u32,`
  - `Expected: PASS (6 tests).` (L2543)
- **New failing tests first:**
  - `merkle_verification` (rewritten): the tree is [a,b,c] and `proof_c = {pos 2, [c, ab]}`.
    ```rust
    assert!(verify_merkle(&c, &proof_c, &root, None));
    assert!(verify_merkle(&c, &proof_c, &root, Some(3)));
    // v1: pos 3 is the duplicated tail and folds to the same root, so it cannot be told apart
    assert!(verify_merkle(&c, &MerkleProof { pos: 3, ..proof_c.clone() }, &root, None));
    // v2: the header commits txcount = 3, so pos 3 is out of range
    assert!(!verify_merkle(&c, &MerkleProof { pos: 3, ..proof_c.clone() }, &root, Some(3)));
    assert!(!verify_merkle(&c, &proof_c, &root, Some(5)));                          // 5 leaves need depth 3
    assert!(!verify_merkle(&c, &MerkleProof { pos: 0, ..proof_c.clone() }, &root, None)); // genuinely wrong position
    ```
  - `merkle_depth_values`: n = 1, 2, 3, 4, 5, 961,632 give 0, 1, 2, 2, 3, 20.
  - `header_proof_verifies_recorded_mainnet_proof`: `tests/fixtures/cp-proof-900000.json` verifies against `ChainParams::mainnet().pre_anchor_root`.
  - `header_proof_rejects_tampering`: each of these returns false: one flipped header bit; one flipped branch bit; `height+1`; the last branch entry dropped; a 164-byte header; `height > cp_height`.
  - `tip_rejects_height_beyond_u32`: a mock answering `blockchain.headers.subscribe` with `{"height": 4294967301, "hex": …}` gives `Err`.
  - `merkle_proof_rejects_values_beyond_u32`: a mock answering `get_merkle` with `block_height` 2^32+110 gives `Err`.
  - `header_with_proof_reverses_display_order`: a mock reply with known hex gives the expected internal-order branch.
  - Expected count goes from 6 to 12.
- **Changes:**
  - Merkle helpers:
    ```rust
    pub fn merkle_depth(n: u32) -> u32 { if n <= 1 { 0 } else { 32 - (n - 1).leading_zeros() } }
    fn merkle_fold(leaf: [u8; 32], mut pos: u32, branch: &[[u8; 32]]) -> Option<[u8; 32]> { /* existing loop */ (pos == 0).then_some(h) }
    pub fn verify_merkle(txid: &Txid, proof: &MerkleProof, root: &[u8; 32], txcount: Option<u32>) -> bool {
        if let Some(n) = txcount {
            if n == 0 || proof.pos >= n || proof.merkle.len() as u32 != merkle_depth(n) { return false; }
        }
        merkle_fold(txid.to_byte_array(), proof.pos, &proof.merkle) == Some(*root)
    }
    /// Electrum blockchain.block.header(height, cp_height): returns (raw header, branch in internal order).
    /// The server's own "root" is ignored; the caller checks against the pinned root.
    pub async fn header_with_proof(&self, height: u32, cp_height: u32) -> Result<(Vec<u8>, Vec<[u8; 32]>), EngineError>;
    pub fn verify_header_proof(raw: &[u8], height: u32, branch: &[[u8; 32]], cp: &HeaderCheckpointRoot) -> bool {
        raw.len() == 80 && height <= cp.cp_height
            && branch.len() as u32 == merkle_depth(cp.cp_height + 1)
            && merkle_fold(sha256d(raw), height, branch) == Some(cp.root)
    }
    ```
    `header_with_proof` reverses each `branch` hex entry. Fulcrum sends both branch and root in display order (`Servers.cpp:1274-1281`).
  - Narrowing: use `u32::try_from(...).map_err(|_| bad("tip"))?` in `tip`, and the same with `bad("merkle")` for `block_height` and `pos`.
- **Interfaces changed:**
  - `verify_merkle` gains `txcount: Option<u32>`.
  - New: `merkle_depth`, `verify_header_proof`, and `ElectrumClient::header_with_proof`.

### Area B — server trust in everyday use

- **Findings:** M11, H4 (parsing), M10 (`transaction_known`)
- **Supersedes:**
  - L2304-2307, starting `if let Some(e) = v.get("error").filter(|e| !e.is_null()) {`
  - L2330 `return Err(EngineError::Server("response too large".into()));`
  - L2373-2375 `fn bad(what: &str) -> EngineError {`
  - L2495 `return Err(EngineError::Server("server sent a transaction that does not match its id".into()));`
  - L2422-2426 `fn fee_from_btc_per_kvb(btc: f64) -> FeeRate {`
  - L2517-2526 `estimate_fee` / `relay_fee`
- **New failing tests first:**
  - `server_error_text_never_reaches_display`: the mock returns error `"Update required: download https://evil.example/wren.apk"`. `e.to_string()` must contain none of `http`, `evil` or `download`.
  - `reject_reasons_are_classified`: `"min relay fee not met"` → `FeeTooLow`, `"bad-txns-premature-spend-of-coinbase"` → `PrematureCoinbase`, `"Transaction outputs already in utxo set"` → `AlreadyConfirmed`, anything else → `Other`.
  - `bad_fee_values_are_none`: negative, zero, NaN or non-number from `estimatefee`/`relayfee` gives `Ok(None)` or `Malformed`, never a panic.
- **Changes:**
  - `rpc.rs`: on a JSON-RPC error, `return Err(EngineError::Server(ServerError::Rejected(classify_reject(msg))));`. Add:
    ```rust
    pub(crate) fn classify_reject(msg: &str) -> RejectKind {
        let m = msg.to_ascii_lowercase();
        for (needle, k) in [("min relay fee not met", RejectKind::FeeTooLow), ("insufficient fee", RejectKind::FeeTooLow),
            ("mempool min fee not met", RejectKind::FeeTooLow), ("txn-mempool-conflict", RejectKind::Conflict),
            ("missingorspent", RejectKind::InputsMissingOrSpent), ("premature-spend-of-coinbase", RejectKind::PrematureCoinbase),
            ("non-final", RejectKind::NonFinal), ("non-bip68-final", RejectKind::NonFinal), ("dust", RejectKind::Dust),
            ("max-fee-exceeded", RejectKind::FeeTooHigh), ("absurdly-high-fee", RejectKind::FeeTooHigh),
            ("already in utxo set", RejectKind::AlreadyConfirmed), ("already in block chain", RejectKind::AlreadyConfirmed)] {
            if m.contains(needle) { return k; }
        }
        RejectKind::Other
    }
    ```
  - `"response too large"` becomes `ServerError::TooLarge`. `bad(_)` becomes `EngineError::Server(ServerError::Malformed)` (keep the argument-free helper `bad()`). A txid mismatch becomes `ServerError::InvalidTransaction`.
  - Fee parsing: `fn fee_from_btc_per_kvb(btc: f64) -> Option<FeeRate>` returns `None` unless `btc.is_finite() && btc > 0.0`. It no longer floors; the floor and cap move to Task 14. `estimate_fee` and `relay_fee` both return `Result<Option<FeeRate>, EngineError>`. A non-number is `Malformed`.
  - New method:
    ```rust
    pub async fn transaction_known(&self, txid: &Txid) -> Result<bool, EngineError> {
        match self.transactions(&[*txid]).await { Ok(_) => Ok(true), Err(EngineError::Server(ServerError::Rejected(_))) => Ok(false), Err(e) => Err(e) }
    }
    ```
- **Interfaces changed:** `relay_fee() -> Result<Option<FeeRate>, _>`, the new `transaction_known`, and `classify_reject` (crate-visible).

### Area C — network, Tor, TLS, Electrum

- **Findings:** ca-ruz #9 (Electrum part); chrisguida M14 (Electrum part); chrisguida Low "Electrum version".
- **Supersedes:**
  - `const MAX_LINE: usize = 16 * 1024 * 1024;`
  - `const TIMEOUT: Duration = Duration::from_secs(60);`
  - `pub async fn batch(&mut self, calls: Vec<(&str, Value)>) -> Result<Vec<Value>, EngineError> {` (the whole body)
  - `async fn read_line(&mut self) -> Result<Vec<u8>, EngineError> {`. It never size-checks the final segment.
  - `let protocol = r.get(1).and_then(Value::as_str).ok_or_else(|| bad("version"))?.to_string();`
  - `for chunk in calls.chunks(50) {` (`ElectrumClient::batch`)
  - The bodies of `histories`, `transactions` and `merkle_proofs`. They now decode typed values as each reply arrives.
  - Step 6 "Expected: PASS (6 tests)." becomes "PASS (13 tests)."
- **New failing tests first:**
  - Step 1: add `pub enum Hostile { NotifyForever { every: Duration }, Silent, HugeNotifications { size: usize } }` and `pub fn spawn_hostile(mode: Hostile) -> BoxStream` to `mock.rs`. Both answer `server.version` with `["x","1.4"]` and then misbehave.
  - Add the dev-dependency `cargo add --dev tokio@1 --features test-util,macros,rt-multi-thread`.
  - `refuses_protocol_outside_requested_range`: each of `["x","1.3"]`, `["x","1.9"]`, `["x","2.0"]`, `["x"]`, `"1.4"` and `["x","one"]` gives `Err(Server)`. `["x","1.4.2"]` connects.
  - `notifications_do_not_extend_a_request_past_its_deadline` (`start_paused`): with `NotifyForever { every: 30 s }`, `relay_fee()` returns `Err(Network)` within `REQUEST_DEADLINE + 1 s`. An outer 3600 s timeout must not be what fires.
  - `silent_server_fails_at_the_deadline_and_poisons_the_client` (`start_paused`): `Silent` gives `Err(Network)`. Then `is_poisoned()` is true, and the next call returns `Err(Network)` with zero elapsed time.
  - `unsolicited_bytes_are_budgeted`: `HugeNotifications { size: 1 MiB }` gives `Err(Server)`.
  - `history_reply_over_the_method_limit_is_refused`: 60,000 history items (more than 4 MiB) give `Err(Server)`.
  - `history_item_cap_is_an_error_not_a_truncation`: 20,001 small items (under 4 MiB) give `Err(Server("address history too large"))`.
  - `batch_byte_budget_bounds_transaction_fetches`: 50 requests for one tx with a 512 KiB output script (about 1 MiB of hex each) give `Err(Server)`, not memory growth.
- **Changes:**
  - `rpc.rs`:
```rust
pub const REQUEST_DEADLINE: Duration = Duration::from_secs(90);
const UNSOLICITED_BUDGET: usize = 1 << 20;   // per batch; protocol 1.7 servers may send server.ping noise
const BATCH_BUDGET: usize = 32 << 20;        // raw bytes read per batch
pub(crate) fn reply_limit(method: &str) -> usize {
    match method {
        "blockchain.transaction.get" => 16 << 20,          // ~4 MB consensus tx -> 8 MB hex, padded
        "blockchain.scripthash.get_history" => 4 << 20,
        "blockchain.block.headers" => 2 << 20,             // 2016 x 164 B as hex ~ 661 kB
        _ => 64 << 10,
    }
}
pub struct Rpc { reader: BufReader<ReadHalf<BoxStream>>, writer: WriteHalf<BoxStream>, next_id: u64, poisoned: Arc<AtomicBool> }
impl Rpc {
    pub fn new(stream: BoxStream, poisoned: Arc<AtomicBool>) -> Self;
    pub async fn call(&mut self, method: &str, params: Value) -> Result<Value, EngineError>; // batch with identity decode
    pub async fn batch<T>(&mut self, calls: &[(&str, Value)],
        decode: impl FnMut(usize, Value) -> Result<T, EngineError>) -> Result<Vec<T>, EngineError>;
}
```
  - `batch` algorithm:
    1. If `poisoned`, return `Err(Network("connection lost"))`.
    2. Set `deadline = Instant::now() + REQUEST_DEADLINE`. Every write and read uses `tokio::time::timeout_at(deadline, …)`.
    3. Set `cap = max(reply_limit(m))` over the calls, and track `budget = BATCH_BUDGET` and `unsolicited = UNSOLICITED_BUDGET`.
    4. Each line is read with `read_line(cap.min(budget))`, then `budget -= len`.
    5. A line whose id is missing, out of range or a duplicate is charged to `unsolicited`; once that is exhausted, fail with `Server("server sent too many unsolicited messages")`.
    6. An in-range reply longer than `reply_limit(calls[idx].0)` fails with `Server("response too large")`.
    7. An `error` reply returns `Err(Server(..))` without poisoning.
    8. Each `result` goes straight to `decode(idx, result)`, so its `Value` is dropped before the next read.
    9. Every other early return (I/O error, timeout, closed connection, limit or budget exceeded, malformed JSON) first sets `poisoned = true`.
  - `read_line(max)` checks `buf.len() + pos > max` on the final segment as well as on intermediate chunks.
  - `mod.rs`:
    - `ElectrumClient { rpc: Mutex<Rpc>, protocol: String, poisoned: Arc<AtomicBool> }`.
    - `pub fn is_poisoned(&self) -> bool` reads the flag without locking.
    - `ElectrumClient::batch<T>(calls, decode)` keeps chunks of 50 and passes absolute indices to `decode`.
    - `histories` checks `arr.len() > MAX_HISTORY_ITEMS_PER_SCRIPT` before mapping, and returns `Err(Server("address history too large"))`. It never truncates.
    - `transactions` does hex decode, `deserialize` and the txid check inside `decode`.
    - `merkle_proofs` builds `MerkleProof` inside `decode`.
```rust
pub const MAX_HISTORY_ITEMS_PER_SCRIPT: usize = 20_000;
fn parse_version(s: &str) -> Option<(u32, u32, u32)> {
    let mut it = s.split('.').map(|p| p.parse::<u32>().ok());
    let (a, b) = (it.next()??, it.next()??);
    let c = match it.next() { None => 0, Some(x) => x? };
    it.next().is_none().then_some((a, b, c))
}
// connect():
let arr = r.as_array().filter(|a| a.len() == 2).ok_or_else(|| bad("version"))?;
let protocol = arr[1].as_str().ok_or_else(|| bad("version"))?;
let v = parse_version(protocol).ok_or_else(|| bad("version"))?;
if v < (1, 4, 0) || v > (1, 8, u32::MAX) {
    return Err(EngineError::Server("server speaks an unsupported Electrum protocol version".into()));
}
```
- **Interfaces changed:**
  - `ElectrumClient::is_poisoned()`.
  - `rpc::REQUEST_DEADLINE`.
  - `MAX_HISTORY_ITEMS_PER_SCRIPT`.
  - `Rpc::new(stream, poisoned)`.
  - `Rpc::batch` is now generic with a decode closure. It is internal; only `ElectrumClient` calls it.
  - The public typed methods of `ElectrumClient` keep their signatures.

## Task 10

### Area C — network, Tor, TLS, Electrum

- **Findings:** chrisguida H3; ca-ruz #6; chrisguida Low "Pinned certificates"; chrisguida Low Tor (i) `.onion` without Tor.
- **Supersedes:**
  - The `ServerAddress` enum, both in Interfaces and in Step 4 (`Tls { host: String, port: u16, pinned_sha256: Option<[u8; 32]> },` and `Onion { host: String, port: u16, tls: bool },`).
  - `pub async fn connect(addr: &ServerAddress) -> Result<BoxStream, EngineError>;`, which is removed.
  - `pub async fn fetch_certificate_fingerprint(host: &str, port: u16) -> Result<[u8; 32], EngineError>;`.
  - `struct PinnedCert {` (and `if sha256(end_entity.as_ref()) == self.pin {`).
  - `*self.seen.lock().unwrap() = Some(sha256(end_entity.as_ref()));`.
  - `async fn tls_connect(host: &str, port: u16, config: ClientConfig)`.
  - Test `async fn wrong_pin_is_refused() {`, which is replaced.
  - `tls_echo_server` now returns `(u16, CertInfo)`.
  - Step 5 "PASS (4 tests)" becomes "PASS (12 tests)".
  - Task 21's `connect_via` and `tls_over`/`tls_config` move here.
- **New failing tests first:**
  - `plaintext_only_to_onion_or_regtest_loopback`. On mainnet, `Tcp` to `electrum.example.com` is refused on both routes, and so is `Tcp` to `192.168.1.10`. `Onion` with a valid 56-character v3 host over `Route::Tor` is OK. On regtest, `Tcp` to `127.0.0.1` is OK and `Tcp` to `10.0.0.5` is refused.
  - `onion_hosts_need_tor_in_every_variant`: `Tcp`, `Tls` and `Onion` with a `.onion` host over `Route::Direct` give `Err(Input)`.
  - `tls_to_onion_requires_a_pin`: `Tls { host: "<v3>.onion", pinned_spki_sha256: None }` gives `Err(Input)`.
  - `refused_address_never_dials`: after a refused `connect_via`, `RecordingDialer::calls()` is empty.
  - `direct_dialer_refuses_onion_before_resolving`: `DirectDialer.dial("abc.onion", 50002)` gives `Err(Input)`.
  - `fingerprint_capture_uses_the_given_dialer`: probing `"abc.onion"` through a `RecordingDialer` aimed at the echo server returns the server's `CertInfo`. The calls are exactly `[("abc.onion", port)]`.
  - `spki_pin_survives_reissue_with_same_key`: two rcgen certs from one `KeyPair` (serials 1 and 2) have different `cert_sha256` and the same `spki_sha256`. A pin from the first connects to the second.
  - `new_key_reports_certificate_changed_with_presented_fingerprint`: pin `[0; 32]` gives `Err(CertificateChanged { spki_sha256, cert_sha256 })` equal to the server's `CertInfo`.
  - `display_fingerprint_matches_openssl_format`: the result is 95 characters, uppercase hex pairs joined by `:`.
  - The existing tests `pinned_self_signed_cert_connects`, `self_signed_without_pin_is_refused_by_web_pki` and `fingerprint_capture_matches_server_cert` switch to `connect_via(…, &DirectDialer, &RoutePolicy { network: Network::Regtest })`.
- **Changes:**
  - Step 1 also runs `cargo tree -i rustls-webpki` to find the version rustls uses, then `cargo add rustls-webpki@<that exact version> --no-default-features --features alloc`. No new crypto crate.
  - Implementation:
```rust
#[derive(Clone, Debug, PartialEq, Eq)]
pub enum ServerAddress {
    Tcp { host: String, port: u16 },                                      // regtest loopback only
    Tls { host: String, port: u16, pinned_spki_sha256: Option<[u8; 32]> }, // None = web PKI; .onion host needs a pin
    Onion { host: String, port: u16 },                                    // plaintext Electrum inside Tor (onion is end-to-end)
}
#[derive(Clone, Copy, Debug, PartialEq, Eq)] pub enum Route { Direct, Tor }
pub type DialFuture<'a> = Pin<Box<dyn Future<Output = Result<BoxStream, EngineError>> + Send + 'a>>;
pub trait Dialer: Send + Sync {
    fn route(&self) -> Route;
    fn dial<'a>(&'a self, host: &'a str, port: u16) -> DialFuture<'a>;
}
pub const DIAL_DEADLINE: Duration = Duration::from_secs(30);
pub const HANDSHAKE_DEADLINE: Duration = Duration::from_secs(30);
pub fn is_onion(h: &str) -> bool { h.len() > 6 && h[h.len() - 6..].eq_ignore_ascii_case(".onion") }
fn is_v3_onion(h: &str) -> bool {
    is_onion(h) && h.len() == 62 && h[..56].bytes().all(|b| b.is_ascii_lowercase() || (b'2'..=b'7').contains(&b))
}
pub struct DirectDialer;
impl Dialer for DirectDialer {
    fn route(&self) -> Route { Route::Direct }
    fn dial<'a>(&'a self, host: &'a str, port: u16) -> DialFuture<'a> {
        Box::pin(async move {
            if is_onion(host) { return Err(EngineError::Input("onion servers need Tor")); } // before getaddrinfo
            let tcp = tokio::time::timeout(DIAL_DEADLINE, TcpStream::connect((host, port))).await.map_err(net)?.map_err(net)?;
            Ok(Box::new(tcp) as BoxStream)
        })
    }
}
pub struct RoutePolicy { pub network: Network }
impl RoutePolicy {
    pub fn check(&self, addr: &ServerAddress, route: Route) -> Result<(), EngineError> {
        let loopback = |h: &str| h == "localhost" || h.parse::<IpAddr>().map(|ip| ip.is_loopback()).unwrap_or(false);
        let tor = route == Route::Tor;
        match addr {
            ServerAddress::Tcp { host, .. } if self.network == Network::Regtest && loopback(host) => Ok(()),
            ServerAddress::Tcp { .. } => Err(EngineError::Input("unencrypted servers are only allowed as onion addresses; use SSL")),
            ServerAddress::Onion { host, .. } if !is_v3_onion(host) => Err(EngineError::Input("not a valid onion address")),
            ServerAddress::Onion { .. } if !tor => Err(EngineError::Input("onion servers need Tor")),
            ServerAddress::Tls { host, pinned_spki_sha256, .. } if is_onion(host) => {
                if !tor { Err(EngineError::Input("onion servers need Tor")) }
                else if pinned_spki_sha256.is_none() { Err(EngineError::Input("an onion SSL server needs its certificate confirmed first")) }
                else { Ok(()) }
            }
            _ => Ok(()),
        }
    }
}
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub struct CertInfo { pub spki_sha256: [u8; 32], pub cert_sha256: [u8; 32] }
fn cert_info(der: &CertificateDer<'_>) -> Result<CertInfo, rustls::Error> {
    let ee = webpki::EndEntityCert::try_from(der).map_err(|_| rustls::Error::General("unreadable server certificate".into()))?;
    Ok(CertInfo { spki_sha256: sha256(ee.subject_public_key_info().as_ref()), cert_sha256: sha256(der.as_ref()) })
}
pub fn display_fingerprint(h: &[u8; 32]) -> String {
    h.iter().map(|b| format!("{b:02X}")).collect::<Vec<_>>().join(":")
}
// PinnedSpki { pin, presented: Mutex<Option<CertInfo>>, provider }:
//   verify_server_cert: info = cert_info(end_entity)?; equal pin -> Ok; else store info in `presented`, Err(General).
//   Handshake signature checks unchanged (they prove the server holds the pinned key).
// CaptureCert stores Some(cert_info(end_entity)?) and aborts as today.
async fn tls_over(stream: BoxStream, host: &str, pin: Option<[u8; 32]>) -> Result<BoxStream, EngineError>;
//   Builds config: PinnedSpki or webpki roots. Runs under HANDSHAKE_DEADLINE.
//   On handshake error with PinnedSpki.presented == Some(i) -> Err(CertificateChanged { spki_sha256: i.spki_sha256, cert_sha256: i.cert_sha256 }).
pub async fn connect_via(addr: &ServerAddress, dialer: &dyn Dialer, policy: &RoutePolicy) -> Result<BoxStream, EngineError> {
    policy.check(addr, dialer.route())?;
    match addr {
        ServerAddress::Tcp { host, port } | ServerAddress::Onion { host, port } => dialer.dial(host, *port).await,
        ServerAddress::Tls { host, port, pinned_spki_sha256 } => tls_over(dialer.dial(host, *port).await?, host, *pinned_spki_sha256).await,
    }
}
pub async fn fetch_certificate_fingerprint(host: &str, port: u16, dialer: &dyn Dialer) -> Result<CertInfo, EngineError>;
//   dialer.dial, then tls_over-with-CaptureCert under HANDSHAKE_DEADLINE (fails by design), then return the captured CertInfo.
```
  - Test support, as `#[cfg(test)] pub(crate)` items in `transport.rs`:
    - `RecordingDialer { route, target_port: Option<u16>, calls: Mutex<Vec<(String, u16)>> }` records each call, then connects to `127.0.0.1:target_port`.
    - `FnDialer { route, f: Arc<dyn Fn(&str, u16) -> BoxStream + Send + Sync> }` is for engine tests.
- **Interfaces changed:**
  - `ServerAddress` (renamed field `pinned_spki_sha256`; `Onion` has no `tls`).
  - `Route`, `Dialer`, `DialFuture`, `DirectDialer`, `RoutePolicy`, `CertInfo`, `display_fingerprint`.
  - `connect_via(addr, &dyn Dialer, &RoutePolicy)`.
  - `fetch_certificate_fingerprint(host, port, &dyn Dialer) -> CertInfo` (now `pub(crate)`; the app goes through `WrenEngine::probe_certificate`).
  - `transport::connect` is removed.

## Task 11

### Area A — headers, sync, chain selection

- **Findings:**
  - ca-ruz #1
  - ca-ruz #3
  - ca-ruz #4 (the clock reaches extension)
  - chrisguida M7 (min-work gate; `tip_time`)
  - chrisguida Lows: "Wrong-chain detection", "Header store", "Integer handling"
- **Supersedes:**
  - The interface `pub async fn sync_headers(client: &ElectrumClient, chain: &mut HeaderChain) -> Result<HeaderSyncOutcome, EngineError>;` (L2815) and the implementation starting L3084.
  - `pub struct HeaderSyncOutcome { pub tip_height: u32, pub reorg_from: Option<u32> }`
  - In the implementation:
    - L3091: `if our_tip > server_tip + MAX_SERVER_LAG {`
    - L3109: `chain.truncate_above(agree);` (truncation before verification)
    - L3119: `chain.extend(next, &raws)?;`
  - `save_chain` body: `std::fs::write(&tmp, chain.to_bytes())…`
  - Step 1 "Create … `testutil.rs`". It becomes "Modify", because `mine` moved to Task 6.
  - Every `sync_headers(&client(&world).await, &mut chain)` in the tests (L3001-3045). These become `sync_headers(&client(&world).await, &mut chain, &[], TEST_NOW)`.
  - `Expected: PASS (5 tests).` (L3144)
- **New failing tests first:**
  - `weaker_branch_is_refused`: `a = TestChain::new(100,30,105,1)` is synced, then the world serves `a.fork_at(120,25,2)`. Result is `Err(Chain(_))`, and the tip stays at 129 with an unchanged hash.
  - `equal_work_branch_is_refused`: the world serves `a.fork_at(120,30,2)`. Result is `Err`, chain unchanged.
  - `invalid_candidate_leaves_chain_intact`: `b = a.fork_at(120,40,2)` with `b.headers[25].prev_hash[0] ^= 1`. Result is `Err`, chain unchanged. Break the link rather than the nonce: the regtest target passes about half of all hashes.
  - `server_below_activation_is_not_verified`: a fresh chain against `TestChain::new(100,4,105,1)` gives `Err(Chain(_))`, and `chain.is_empty()` afterwards.
  - `legacy_server_with_saved_chain_is_wrong_chain`: `a = TestChain::new(100,300,105,1)` is synced, then the world serves `TestChain::new(100,300,1000,1)`. That chain shares heights 100-104 and has a v1 header at 105. Result is `Err(WrongChain)`, and the tip stays at 399.
  - `too_little_work_is_refused`: `params.with_min_chain_work(U256([50,0,0,0]))`. 20 headers give `Err` and leave the chain empty; 30 headers give `Ok`.
  - `reset_store_conflicting_with_wallet_history_is_refused`: a fresh chain with `known = [(150, a@150), (399, a@399)]`, while the world serves `a.fork_at(120, 450, 2)`. Result is `Err(Chain(_))`, chain empty.
  - `reset_store_shallow_reorg_is_accepted`: the same `known`, while the world serves `a.fork_at(390, 420, 2)`. Result is `Ok`, tip 419.
  - `save_leaves_no_tmp_file`: after `save_chain`, `headers.tmp` does not exist.
  - `outcome_reports_tip_time`: `out.tip_time == a.headers.last().unwrap().time()`.
  - Expected count goes from 5 to 15.
- **Changes:**
  ```rust
  pub struct HeaderSyncOutcome { pub tip_height: u32, pub tip_time: u32, pub reorg_from: Option<u32> }

  /// `known`: blocks the wallet recorded (BDK LocalChain checkpoints, height + internal hash), checked only when
  /// the header store was empty (fresh or reset). `now`: unix seconds, for the 2 h future bound.
  /// On any error the chain is left exactly as it was (or empty if it started empty).
  pub async fn sync_headers(client: &ElectrumClient, chain: &mut HeaderChain, known: &[(u32, [u8; 32])], now: u64)
      -> Result<HeaderSyncOutcome, EngineError> {
      let started_empty = chain.is_empty();
      let tip0 = chain.tip_height();
      let r = sync_headers_inner(client, chain, known, now, started_empty).await;
      if r.is_err() {
          match tip0 { None => *chain = HeaderChain::new(chain.params().clone()), Some(t) => chain.truncate_above(t) }
      }
      r
  }

  async fn sync_headers_inner(client: &ElectrumClient, chain: &mut HeaderChain, known: &[(u32, [u8; 32])], now: u64,
                              started_empty: bool) -> Result<HeaderSyncOutcome, EngineError> {
      if started_empty { chain.set_anchor(&client.header(chain.base_height()).await?)?; }
      let (server_tip, _) = client.tip().await?;
      probe_activation(client, chain, server_tip).await?;               // before any walk-back
      let our_tip = chain.tip_height().expect("anchored");
      if our_tip.saturating_sub(MAX_SERVER_LAG) > server_tip {
          return Err(EngineError::Server("server is behind the chain".into()));
      }
      let mut reorg_from = None;
      let check = our_tip.min(server_tip);
      if check > chain.base_height() && server_hash(client, check).await? != chain.hash_at(check).unwrap() {
          let agree = find_agreement(client, chain, check).await?;       // the existing walk-back loop, <= MAX_REORG_DEPTH
          let mut cand = chain.clone();
          cand.truncate_above(agree);
          fetch_and_extend(client, &mut cand, server_tip, now).await?;
          if cand.work_above(agree) <= chain.work_above(agree) {          // most work wins; a tie keeps ours (Knots first-seen)
              return Err(EngineError::Chain("server's branch has no more work than ours"));
          }
          *chain = cand;                                                  // committed only after verification and the work check
          reorg_from = Some(agree + 1);
      } else {
          fetch_and_extend(client, chain, server_tip, now).await?;
      }
      if !chain.reached_activation() { return Err(EngineError::Chain("chain did not reach BLAKE2b activation")); }
      if !chain.meets_min_work() { return Err(EngineError::Chain("server chain has too little work")); }
      if started_empty { check_known(chain, known)?; }
      let tip = chain.tip_height().unwrap();
      Ok(HeaderSyncOutcome { tip_height: tip, tip_time: chain.header_at(tip).unwrap().time(), reorg_from })
  }

  async fn probe_activation(client: &ElectrumClient, chain: &HeaderChain, server_tip: u32) -> Result<(), EngineError> {
      let p = chain.params();
      if server_tip < p.blake2b_height {
          return Err(EngineError::Chain("server has not reached BLAKE2b activation"));
      }
      let h = ForkHeader::parse(&client.header(p.blake2b_height).await?)?;
      let hash = h.block_hash();
      let cp_bad = p.checkpoints.iter().any(|c| c.height == p.blake2b_height && c.hash != hash);
      let ours_bad = chain.hash_at(p.blake2b_height).is_some_and(|x| x != hash);
      if !h.is_v2() || cp_bad || ours_bad { return Err(EngineError::WrongChain); }
      Ok(())
  }

  async fn fetch_and_extend(client: &ElectrumClient, chain: &mut HeaderChain, server_tip: u32, now: u64) -> Result<(), EngineError> {
      let mut next = chain.tip_height().expect("anchored").checked_add(1).ok_or(EngineError::Chain("height overflow"))?;
      while next <= server_tip {
          let raws = client.headers(next, (server_tip - next + 1).min(2016)).await?;
          if raws.is_empty() { return Err(EngineError::Server("server returned no headers".into())); }
          chain.extend_at(next, &raws, now)?;
          next = next.checked_add(raws.len() as u32).ok_or(EngineError::Chain("height overflow"))?;
      }
      Ok(())
  }

  /// After a fresh or reset store: a reorg may only touch the last MAX_REORG_DEPTH blocks below the wallet's
  /// highest recorded block, and the server may not be more than MAX_SERVER_LAG behind it.
  fn check_known(chain: &HeaderChain, known: &[(u32, [u8; 32])]) -> Result<(), EngineError> {
      let base = chain.base_height();
      let Some(top) = known.iter().map(|(h, _)| *h).filter(|h| *h > base).max() else { return Ok(()) };
      let tip = chain.tip_height().expect("anchored");
      let conflict = tip.saturating_add(MAX_SERVER_LAG) < top
          || known.iter().any(|(h, hash)| *h > base && h.saturating_add(MAX_REORG_DEPTH) <= top && chain.hash_at(*h) != Some(*hash));
      if conflict { Err(EngineError::Chain("server chain conflicts with this wallet's history")) } else { Ok(()) }
  }

  pub fn save_chain(chain: &HeaderChain, path: &Path) -> Result<(), EngineError> {
      use std::io::Write;
      let io = |e: std::io::Error| EngineError::Wallet(e.to_string());
      let tmp = path.with_extension("tmp");
      let mut f = std::fs::File::create(&tmp).map_err(io)?;
      f.write_all(&chain.to_bytes()).map_err(io)?;
      f.sync_all().map_err(io)?;
      drop(f);
      std::fs::rename(&tmp, path).map_err(io)?;
      #[cfg(unix)]
      if let Some(dir) = path.parent() { std::fs::File::open(dir).and_then(|d| d.sync_all()).map_err(io)?; }
      Ok(())
  }
  ```
  - `EngineError::Wallet(e.to_string())` follows whatever the error-text area decides. This area only needs durability.
  - Limitation, to state in the plan: `check_known` can only be as precise as the wallet's checkpoint density. BDK keeps one checkpoint per sync tip.
  - **Deferred, add to backlog:** a rolling verified checkpoint to bound the RAM use and the re-verify on launch.
- **Interfaces changed:**
  - `sync_headers(client, chain, known: &[(u32, [u8; 32])], now: u64)`
  - `HeaderSyncOutcome.tip_time`
- **Mechanical call-site updates in other tasks:** pass `&[], TEST_NOW` in tests and the real values in the engine.
  - Task 13 L3470; Task 14 L3734; Task 17 L4703; Task 18 L4969
  - Task 20 L5913 and L5941 (see Task 20)

### Area B — server trust in everyday use

- **Findings:** M11 (string variants), test infrastructure for H4, M10 and M11
- **Supersedes:**
  - L3092 `return Err(EngineError::Server("server is behind the chain".into()));` becomes `ServerError::Behind`.
  - L3117 `return Err(EngineError::Server("server returned no headers".into()));` becomes `ServerError::NoHeaders`.
  - In MockWorld, L2917 `pub struct MockWorld {`, L2928 `Arc::new(Mutex::new(MockWorld { chain, ...`, and L2969-2976 (the `estimatefee` / `relayfee` / `broadcast` arms).
- **New failing tests first:** none of its own. The fields are exercised by the Task 14, 19 and 20 tests.
- **Changes:**
  - Add fields `pub relay_fee: f64, pub fee_estimate: f64, pub broadcast_mode: BroadcastMode`, defaulting to `0.00001`, `0.00002` and `BroadcastMode::Accept`.
  - Add `pub enum BroadcastMode { Accept, Reject(String), RecordThenFail, LieOk }`.
  - `serve` behaviour:
    - `estimatefee` returns `w.fee_estimate`; `relayfee` returns `w.relay_fee`.
    - `broadcast` with `Accept`: push the tx and return its txid.
    - `Reject(s)`: return `Err(s)` without recording.
    - `RecordThenFail`: push the tx, then `Err("connection reset by peer")`.
    - `LieOk`: return the txid without pushing the tx.
    - In all modes, after `Accept` or `RecordThenFail`, insert the tx into `w.txs` so `transaction.get` finds it.
- **Interfaces changed:** the MockWorld fields and `BroadcastMode` (test-only), used by Tasks 14, 19 and 20.

### Area C — network, Tor, TLS, Electrum

- **Findings:** ca-ruz #9 and the Claim session decision. These are test helpers needed by Task 20.
- **Changes:** in `testutil.rs`:
  - Add `pub fn serve_tagged(world, tag: u32) -> BoxStream`. It is `serve` plus a push of `(tag, method, params)` to a new field `MockWorld.log: Vec<(u32, String, Value)>`, and a push of `(tag, "closed", Null)` when the stream ends.
  - Add `pub fn serve_stalling(world, answer_first_n: usize) -> BoxStream`. It answers the first N requests, then never answers again, while sending a `blockchain.headers.subscribe` notification every 30 s.
  - `serve(world)` stays, as `serve_tagged(world, 0)`.
- **Interfaces changed:** `serve_tagged`, `serve_stalling`, `MockWorld.log` (tests only).

## Task 12a — Custody (NEW, before Task 12)

### Area D — key custody, engine API, PIN

- **Findings:** chrisguida H2, ca-ruz #5, chrisguida M12 (PIN key construction), ca-ruz "Unconfirmed risks" (KDF/hardware protocol)
- **Supersedes:** none (new task). It takes over `ACCOUNT_PATH` and `generate_words` from Task 12.
- **Files:**
  - Create `core/wren-engine/src/custody.rs` and `core/clippy.toml`.
  - Create `core/wren-engine/tests/vectors/h2c_p256.json`.
  - Modify `src/error.rs`, `src/lib.rs` (`pub mod custody;`), `tests/vectors/SOURCES.md` and `Cargo.toml`.
- **New failing tests first** (inline in `custody.rs`):
  - `generated_seed_has_12_valid_words`: `SeedSecret::generate().words()` returns 12 words, and `from_words` accepts them.
  - `from_words_rejects_invalid_and_24_word_phrases`: moved from Task 12.
  - `words_round_trip`: `from_words(ABANDON).words().join(" ") == ABANDON`.
  - `seal_then_open_round_trips`: entropy and passphrase come back identical.
  - `open_with_wrong_key_is_wrong_pin`: a different `WrappingKey` gives `EngineError::WrongPin`.
  - `open_with_other_network_fails`: a blob sealed for `Bitcoin` will not open as `Regtest`.
  - `tampered_or_unknown_version_blob_is_refused`: flipping one ciphertext byte gives `WrongPin`; `blob[0] = 2` gives a "newer Wren" error.
  - `sealed_blob_contains_no_entropy_or_passphrase`: the blob doesn't contain the entropy bytes or the UTF-8 passphrase.
  - `passphrase_changes_fingerprint`: `KeySource` fingerprints differ for ABANDON with `""` and with `"tree"`.
  - `with_key_for_signs_bip84_vector_key`: a PSBT input with `bip32_derivation` for `m/84'/0'/0'/0/0`; the closure's key has the P2WPKH address `bc1qcr8te4kr609gcawutmrza0j4xv80jy8z306fyu`.
  - `with_key_for_skips_foreign_fingerprint_and_paths_outside_the_account`: gives `Signing("no key in this wallet for an input")`.
  - `with_key_for_propagates_closure_error`: the closure's `Err` is returned unchanged.
  - `stretch_pin_is_deterministic_and_bound_to_salt_and_params`: same inputs give the same output; changing the salt or `t` changes it; `m_kib < 19_456` is refused.
  - `hash_to_p256_matches_rfc9380_vectors`: the `msg=""` and `msg="abc"` vectors from RFC 9380 Appendix J.1.1 (`P256_XMD:SHA-256_SSWU_RO_`, DST `QUUX-V01-CS02-with-P256_XMD:SHA-256_SSWU_RO_`), read from `tests/vectors/h2c_p256.json`.
  - `wrapping_key_depends_on_both_inputs`: changing `a` or `h` changes the key.
  - `install_key_vault_succeeds_once`: the second call returns `Err(EngineError::KeyVault)`.
- **Changes:**
  1. Dependencies (from `core/wren-engine`):
     - `cargo add chacha20poly1305@0.10 argon2@0.5 hkdf@0.12 sha2@0.10`
     - `cargo add p256@0.13 --features hash2curve`
     - `cargo add bip39@2.2.2 --features zeroize`. This is feature-only: it unifies with bdk_wallet's copy. Code still uses `bdk_wallet::keys::bip39` paths.
     - `cargo add --dev tokio@1 --features test-util`
     - Add to `Cargo.toml`: `[features] test-hooks = []`. This moves here from Task 20 Step 1.
  2. Vectors: copy the two RFC 9380 J.1.1 vectors (msg, P.x, P.y) into `tests/vectors/h2c_p256.json`, and record the source URL and sha256 in `SOURCES.md`.
  3. `core/clippy.toml`:
     ```toml
     disallowed-types = [
       { path = "bdk_wallet::bitcoin::bip32::Xpriv", reason = "secrets live only in custody.rs/sign.rs" },
       { path = "bdk_wallet::bitcoin::secp256k1::SecretKey", reason = "secrets live only in custody.rs/sign.rs" },
       { path = "bdk_wallet::bitcoin::PrivateKey", reason = "secrets live only in custody.rs/sign.rs" },
       { path = "bdk_wallet::keys::bip39::Mnemonic", reason = "secrets live only in custody.rs" },
     ]
     ```
     Allow these types only in `custody.rs`, `sign.rs`, `#[cfg(test)]` modules and `tests/regtest.rs`, via `#![allow(clippy::disallowed_types)]`.
  4. Add to `EngineError` (`error.rs`). These carry no secrets:
     ```rust
     #[error("Wren is locked")] Locked,
     #[error("wrong Wren PIN")] WrongPin,
     #[error("cancelled")] AuthCancelled,
     #[error("too many wrong PINs; try again later")] LockedOut { retry_after_secs: u64 },
     #[error("confirm your recovery phrase before sending")] BackupNotConfirmed,
     #[error("this recovery phrase was used for bitcoin before the split")] SeedHoldsBtc,
     #[error("secure storage is not available")] KeyVault,
     ```
  5. `custody.rs`:
     ```rust
     //! Seed custody. With sign.rs, the only module allowed to hold seed bytes, Mnemonic, Xpriv or SecretKey.
     //! Best effort: every copy we own is wiped on success and error paths; Rust moves/copies,
     //! rust-bitcoin's BIP32 internals and bridge string copies can leave bytes we cannot reach.
     #![allow(clippy::disallowed_types)]
     use crate::error::EngineError;
     use bdk_wallet::bitcoin::bip32::{ChainCode, ChildNumber, DerivationPath, Fingerprint, Xpriv, Xpub};
     use bdk_wallet::bitcoin::secp256k1::rand::{rngs::OsRng, RngCore};
     use bdk_wallet::bitcoin::secp256k1::{All, PublicKey, Secp256k1, SecretKey};
     use bdk_wallet::bitcoin::{psbt, Amount, Network};
     use bdk_wallet::keys::bip39::{Language, Mnemonic};
     use chacha20poly1305::aead::{Aead, KeyInit, Payload};
     use chacha20poly1305::{Key, XChaCha20Poly1305, XNonce};
     use std::str::FromStr;
     use std::sync::{Arc, OnceLock};
     use zeroize::Zeroizing;

     pub const ACCOUNT_PATH: &str = "m/84'/0'/0'";
     pub const SEALED_SEED_VERSION: u8 = 1;
     const SEAL_AAD: &[u8] = b"wren/sealed-seed";

     // ---------- values that wipe themselves ----------
     pub(crate) trait Wipe { fn wipe(&mut self); }
     impl Wipe for Xpriv { fn wipe(&mut self) { self.private_key.non_secure_erase(); self.chain_code = ChainCode::from([0u8; 32]); } }
     impl Wipe for SecretKey { fn wipe(&mut self) { self.non_secure_erase(); } }
     pub(crate) struct Wiped<T: Wipe>(pub(crate) T);
     impl<T: Wipe> Drop for Wiped<T> { fn drop(&mut self) { self.0.wipe(); } }
     impl<T: Wipe> std::ops::Deref for Wiped<T> { type Target = T; fn deref(&self) -> &T { &self.0 } }

     // ---------- the seed ----------
     /// BIP39 entropy + passphrase. Never the words string.
     pub(crate) struct SeedSecret { entropy: Zeroizing<[u8; 16]>, passphrase: Zeroizing<String> }
     impl SeedSecret {
         pub(crate) fn generate() -> Self {
             let mut entropy = Zeroizing::new([0u8; 16]);
             OsRng.fill_bytes(&mut *entropy);
             SeedSecret { entropy, passphrase: Zeroizing::new(String::new()) }
         }
         pub(crate) fn from_words(words: Zeroizing<String>, passphrase: Zeroizing<String>) -> Result<Self, EngineError> {
             let m = Mnemonic::parse_in_normalized(Language::English, &words)
                 .map_err(|_| EngineError::Input("recovery phrase is not valid"))?; // ZeroizeOnDrop (bip39 "zeroize")
             if m.word_count() != 12 { return Err(EngineError::Input("recovery phrase must be 12 words")); }
             let full = Zeroizing::new(m.to_entropy_array().0);
             let mut entropy = Zeroizing::new([0u8; 16]);
             entropy.copy_from_slice(&full[..16]);
             Ok(SeedSecret { entropy, passphrase })
         }
         pub(crate) fn words(&self) -> Zeroizing<Vec<String>> {
             let m = Mnemonic::from_entropy(&*self.entropy).expect("16 bytes is valid BIP39 entropy");
             Zeroizing::new(m.words().map(String::from).collect())
         }
     }

     // ---------- sealing ----------
     /// Opens the sealed seed. Produced only by a native KeyVault (or TestVault). Never built from TypeScript.
     pub struct WrappingKey(Zeroizing<[u8; 32]>);
     impl WrappingKey { pub fn from_bytes(b: Zeroizing<[u8; 32]>) -> Self { WrappingKey(b) } }
     impl std::fmt::Debug for WrappingKey { fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result { f.write_str("WrappingKey(..)") } }

     fn aad(network: Network) -> Vec<u8> {
         let mut a = SEAL_AAD.to_vec();
         a.push(SEALED_SEED_VERSION);
         a.extend_from_slice(&network.magic().to_bytes());
         a
     }
     /// version(1) || nonce(24) || XChaCha20-Poly1305(entropy(16) || passphrase utf8)
     pub(crate) fn seal(seed: &SeedSecret, key: &WrappingKey, network: Network) -> Vec<u8> {
         let mut plain = Zeroizing::new(Vec::with_capacity(16 + seed.passphrase.len()));
         plain.extend_from_slice(&*seed.entropy);
         plain.extend_from_slice(seed.passphrase.as_bytes());
         let mut nonce = [0u8; 24];
         OsRng.fill_bytes(&mut nonce);
         let ct = XChaCha20Poly1305::new(Key::from_slice(&key.0[..]))
             .encrypt(XNonce::from_slice(&nonce), Payload { msg: &plain, aad: &aad(network) })
             .expect("encryption of a short message cannot fail");
         let mut out = vec![SEALED_SEED_VERSION];
         out.extend_from_slice(&nonce);
         out.extend_from_slice(&ct);
         out
     }
     pub(crate) fn open_sealed(blob: &[u8], key: &WrappingKey, network: Network) -> Result<SeedSecret, EngineError> {
         if blob.first() != Some(&SEALED_SEED_VERSION) {
             return Err(EngineError::Wallet("the saved wallet is damaged or from a newer Wren".into()));
         }
         if blob.len() < 1 + 24 + 16 + 16 { return Err(EngineError::Wallet("the saved wallet is damaged".into())); }
         let plain = Zeroizing::new(
             XChaCha20Poly1305::new(Key::from_slice(&key.0[..]))
                 .decrypt(XNonce::from_slice(&blob[1..25]), Payload { msg: &blob[25..], aad: &aad(network) })
                 .map_err(|_| EngineError::WrongPin)?,
         );
         let mut entropy = Zeroizing::new([0u8; 16]);
         entropy.copy_from_slice(&plain[..16]);
         let passphrase = Zeroizing::new(String::from_utf8(plain[16..].to_vec())
             .map_err(|_| EngineError::Wallet("the saved wallet is damaged".into()))?);
         Ok(SeedSecret { entropy, passphrase })
     }

     // ---------- PIN key helpers (called by the native vault; same code on iOS and Android) ----------
     #[derive(Clone, Copy, Debug, PartialEq, Eq)]
     pub struct PinKdfParams { pub m_kib: u32, pub t: u32, pub p: u32 }
     /// Starting point. Sub-plan 2 calibrates to about 1 s per guess on the slowest supported phone
     /// and stores the chosen params with the salt.
     pub const PIN_KDF_DEFAULT: PinKdfParams = PinKdfParams { m_kib: 65_536, t: 3, p: 1 };
     pub const PIN_POINT_DST: &[u8] = b"WREN-V01-CS01-with-P256_XMD:SHA-256_SSWU_RO_";

     /// A = Argon2id(PIN, salt).
     pub fn stretch_pin(pin: &[u8], salt: &[u8; 16], k: PinKdfParams) -> Result<Zeroizing<[u8; 32]>, EngineError> {
         if k.m_kib < 19_456 || k.t < 2 { return Err(EngineError::Input("PIN settings are too weak")); }
         let params = argon2::Params::new(k.m_kib, k.t, k.p, Some(32)).map_err(|_| EngineError::Input("bad PIN settings"))?;
         let mut out = Zeroizing::new([0u8; 32]);
         argon2::Argon2::new(argon2::Algorithm::Argon2id, argon2::Version::V0x13, params)
             .hash_password_into(pin, salt, &mut *out)
             .map_err(|_| EngineError::Input("bad PIN settings"))?;
         Ok(out)
     }
     /// P = hash_to_curve(A) on P-256 (RFC 9380). iOS: H = ECDH(Secure Enclave key, P).
     /// Never use A·G as the peer point: it is computable from the public key.
     pub fn pin_point(a: &[u8; 32]) -> Result<[u8; 65], EngineError> { hash_to_p256(a, PIN_POINT_DST) }
     fn hash_to_p256(msg: &[u8], dst: &[u8]) -> Result<[u8; 65], EngineError> {
         use p256::elliptic_curve::hash2curve::{ExpandMsgXmd, GroupDigest};
         use p256::elliptic_curve::sec1::ToEncodedPoint;
         let p = p256::NistP256::hash_from_bytes::<ExpandMsgXmd<sha2::Sha256>>(&[msg], &[dst])
             .map_err(|_| EngineError::Input("hash to curve failed"))?;
         let enc = p.to_affine().to_encoded_point(false);
         let mut out = [0u8; 65];
         out.copy_from_slice(enc.as_bytes());
         Ok(out)
     }
     /// W = HKDF-SHA256(A || H, info "wren/seal/v1"). H: iOS ECDH x-coordinate, Android Keystore HMAC-SHA256(A).
     pub fn derive_wrapping_key(a: &[u8; 32], h: &[u8; 32]) -> WrappingKey {
         let mut ikm = Zeroizing::new([0u8; 64]);
         ikm[..32].copy_from_slice(a);
         ikm[32..].copy_from_slice(h);
         let mut okm = Zeroizing::new([0u8; 32]);
         hkdf::Hkdf::<sha2::Sha256>::new(None, &*ikm).expand(b"wren/seal/v1", &mut *okm).expect("32 bytes is a valid HKDF length");
         WrappingKey(okm)
     }

     // ---------- the native key vault ----------
     /// What the native confirm sheet shows. Built by the engine from its own ReviewSummary.
     #[derive(Clone, Debug, PartialEq)]
     pub struct SendApproval { pub review_id: u64, pub recipient: String, pub amount: Amount, pub fee: Amount,
         pub fee_rate_sat_vb: u64, pub to_self: bool, pub abnormal_fee: bool }
     #[derive(Clone, Debug, PartialEq)]
     pub enum AuthPurpose {
         Unlock,               // app open; biometrics allowed
         Send(SendApproval),   // every send and Speed up; biometrics allowed; native sheet shows the approval
         ConfirmBackup,        // finishing a skipped backup check later; biometrics allowed
         RevealWords,          // Wren PIN only
         ChangePin,            // Wren PIN only
     }
     #[derive(Debug)]
     pub enum VaultError { Cancelled, LockedOut { retry_after_secs: u64 }, Unavailable }
     impl From<VaultError> for EngineError {
         fn from(e: VaultError) -> Self {
             match e {
                 VaultError::Cancelled => EngineError::AuthCancelled,
                 VaultError::LockedOut { retry_after_secs } => EngineError::LockedOut { retry_after_secs },
                 VaultError::Unavailable => EngineError::KeyVault,
             }
         }
     }
     /// Implemented in Swift/Kotlin (sub-plan 2), never in TypeScript. Methods may block (PIN pad,
     /// biometric prompt); the engine calls them via spawn_blocking and never while holding its state mutex.
     pub trait KeyVault: Send + Sync {
         /// Authenticate for `purpose` (native Wren PIN pad, or biometrics where allowed), then return W.
         /// The wrong-PIN counter is incremented and written to storage BEFORE W is derived.
         fn authorize(&self, purpose: &AuthPurpose) -> Result<WrappingKey, VaultError>;
         /// Native PIN setup (entered twice). Returns the new W.
         fn enroll(&self) -> Result<WrappingKey, VaultError>;
         /// The engine reports whether W opened the seal; true resets the counter.
         fn report(&self, opened: bool);
         fn store_sealed(&self, blob: &[u8]) -> Result<(), VaultError>; // device-only, atomic replace
         fn load_sealed(&self) -> Result<Option<Vec<u8>>, VaultError>;
         fn delete_sealed(&self) -> Result<(), VaultError>;
     }
     static VAULT: OnceLock<Arc<dyn KeyVault>> = OnceLock::new();
     /// Called once from native app start, before JS runs. NOT exported to TypeScript.
     pub fn install_key_vault(v: Arc<dyn KeyVault>) -> Result<(), EngineError> {
         VAULT.set(v).map_err(|_| EngineError::KeyVault)
     }
     pub(crate) fn installed_vault() -> Option<Arc<dyn KeyVault>> { VAULT.get().cloned() }

     // ---------- per-operation signer ----------
     pub(crate) struct KeySource { account: Wiped<Xpriv>, fp: Fingerprint, secp: Secp256k1<All> }
     impl KeySource {
         pub(crate) fn from_seed(seed: &SeedSecret, network: Network) -> Result<Self, EngineError> {
             let m = Mnemonic::from_entropy(&*seed.entropy).map_err(|_| EngineError::Wallet("the saved wallet is damaged".into()))?;
             let mut bip39_seed = Zeroizing::new([0u8; 64]);
             *bip39_seed = m.to_seed(seed.passphrase.as_str());
             drop(m);
             let secp = Secp256k1::new();
             let master = Wiped(Xpriv::new_master(network, &*bip39_seed).map_err(|_| EngineError::Wallet("key derivation failed".into()))?);
             let fp = master.fingerprint(&secp);
             let account = Wiped(master.derive_priv(&secp, &DerivationPath::from_str(ACCOUNT_PATH).unwrap())
                 .map_err(|_| EngineError::Wallet("key derivation failed".into()))?);
             Ok(KeySource { account, fp, secp }) // `master` wiped here
         }
         pub(crate) fn fingerprint(&self) -> Fingerprint { self.fp }
         /// Public descriptors only.
         pub(crate) fn descriptors(&self) -> (String, String) {
             let xpub = Xpub::from_priv(&self.secp, &self.account);
             let fp = self.fp;
             (format!("wpkh([{fp}/84'/0'/0']{xpub}/0/*)"), format!("wpkh([{fp}/84'/0'/0']{xpub}/1/*)"))
         }
         /// Derives the key for `input`, lends it to `f`, and wipes every copy we own on every path.
         pub(crate) fn with_key_for<R>(&self, input: &psbt::Input,
             f: impl FnOnce(&SecretKey) -> Result<R, EngineError>) -> Result<R, EngineError> {
             let acct = DerivationPath::from_str(ACCOUNT_PATH).unwrap();
             let acct: &[ChildNumber] = acct.as_ref();
             for (pk, (origin_fp, path)) in &input.bip32_derivation {
                 let full: &[ChildNumber] = path.as_ref();
                 if *origin_fp != self.fp || full.len() != 5 || full[..3] != *acct { continue; }
                 let child = Wiped(self.account.derive_priv(&self.secp, &DerivationPath::from(&full[3..]))
                     .map_err(|_| EngineError::Signing("key derivation failed"))?);
                 if PublicKey::from_secret_key(&self.secp, &child.private_key) != *pk { continue; }
                 let sk = Wiped(child.private_key);
                 return f(&sk);
             }
             Err(EngineError::Signing("no key in this wallet for an input"))
         }
     }
     ```
  6. Test vault, compiled only with `#[cfg(any(test, feature = "test-hooks"))] pub mod test_vault`:
     - `pub enum Answer { Correct, Wrong, Cancel }`
     - `pub struct TestVault { key: [u8; 32], pub sealed: Mutex<Option<Vec<u8>>>, pub purposes: Mutex<Vec<AuthPurpose>>, pub reports: Mutex<Vec<bool>>, script: Mutex<VecDeque<Answer>> }`
     - `TestVault::new() -> Arc<Self>` uses a random key.
     - `answer_next(Answer)` queues the next answer. `authorize` records the purpose and pops the queue (default `Correct`); `Wrong` returns a different key.
     - `enroll` returns the key. The storage methods use `sealed`.
  7. Run `cargo test -p wren-engine custody::`. Expected: PASS (16 tests).
  8. Commit:
     ```bash
     git add core/clippy.toml core/wren-engine
     git commit -m "feat(engine): seed custody — sealed seed, native key-vault contract, per-operation signer"
     ```
- **Interfaces changed:**
  - New `custody` module: `ACCOUNT_PATH`, `WrappingKey`, `SendApproval`, `AuthPurpose`, `VaultError`, `KeyVault`, `install_key_vault`, `PinKdfParams`, `PIN_KDF_DEFAULT`, `stretch_pin`, `pin_point`, `derive_wrapping_key`, and crate-internal `SeedSecret`, `seal`, `open_sealed`, `KeySource`, `Wiped`.
  - New `EngineError` variants (listed in step 4).

## Task 12

### Area B — server trust in everyday use

- **Findings:** M6, M15, M8 (shared filter), Low "Error strings"
- **Supersedes:**
  - L3261 `fn werr<E: std::fmt::Display>(e: E) -> EngineError {`
  - L3301 `pub struct TxSummary {` (struct plus constructor at L3359)
  - L3341 `pub fn next_receive_address(&mut self) -> Result<Address, EngineError> {`
  - L3347 `pub fn balance(&self) -> Balance {`
  - L3322 `return Err(EngineError::Wallet("a wallet already exists on this device".into()));`
  - L3337 `.ok_or(EngineError::Wallet("no wallet on this device".into()))?;`
  - Interfaces lines L3169-3176
- **New failing tests first:**
  - `receive_addresses_never_outrun_the_restore_gap`: 30 calls to `next_receive_address()` with no payments. The max index (`derivation_of_spk`) must be `< MAX_UNUSED_RECEIVE`, and calls 11-30 return the index-0 address.
  - `wallet_errors_carry_no_amounts_or_ids`: a bad-database error's `to_string()` has no digits.
- **Changes:**
  1. Remove `werr`. Every `map_err(werr)` becomes `map_err(|_| EngineError::Wallet("<fixed text>"))`, for example "could not open the wallet database" and "could not derive wallet keys". Drop the `.into()` calls.
  2. Create `src/spendable.rs` and add `pub(crate) mod spendable;` to `lib.rs`:
     ```rust
     //! Which coins may be spent. Shared by payments (Task 14), fee bumps, and claims (Task 18).
     /// A tx is "own" if it spends at least one of this wallet's outputs. After Task 13 that implies our key signed it.
     pub(crate) fn is_own_tx(w: &Wallet, tx: &Transaction) -> bool {
         tx.input.iter().any(|i| w.tx_graph().get_txout(i.previous_output).is_some_and(|o| w.is_mine(o.script_pubkey.clone())))
     }
     /// Coinbase outputs not yet mature under BOTH consensus (long window) and Knots relay policy
     /// (every coinbase needs CoinbaseMaturityLong in the mempool, validation.cpp:1021-1023).
     pub(crate) fn immature_coinbase(w: &Wallet, params: &ChainParams, tip: u32) -> Vec<OutPoint> {
         let spend_height = tip + 1;
         w.list_unspent().filter_map(|u| {
             let tx = w.tx_graph().get_tx(u.outpoint.txid)?;
             if !tx.is_coinbase() { return None; }
             let ChainPosition::Confirmed { anchor, .. } = &u.chain_position else { return Some(u.outpoint) };
             let h = anchor.block_id.height;
             let consensus = if h >= params.long_maturity_start && spend_height < params.long_maturity_end { 6_480 } else { 100 };
             (spend_height.saturating_sub(h) < consensus.max(params.coinbase_relay_maturity)).then_some(u.outpoint)
         }).collect()
     }
     /// Unconfirmed coins whose parent is not our own tx: never spendable, never in `spendable`.
     pub(crate) fn untrusted_unconfirmed(w: &Wallet) -> Vec<OutPoint> {
         w.list_unspent().filter(|u| !u.chain_position.is_confirmed())
             .filter(|u| w.tx_graph().get_tx(u.outpoint.txid).map_or(true, |t| !is_own_tx(w, &t)))
             .map(|u| u.outpoint).collect()
     }
     ```
  3. Add `pub struct WrenBalance { pub spendable: Amount, pub incoming_pending: Amount, pub immature: Amount }` and `pub fn balance(&self, params: &ChainParams) -> WrenBalance`. With `tip = self.wallet.latest_checkpoint().height()`, each `list_unspent` coin is counted as:
     - in `immature_coinbase(..)` → `immature`;
     - in `untrusted_unconfirmed(..)` → `incoming_pending`;
     - otherwise → `spendable`.
  4. `TxSummary` gains `pub confirmations: u32`: `tip - h + 1`, or 0 when unconfirmed.
  5. Receive gap guard:
     ```rust
     pub const MAX_UNUSED_RECEIVE: u32 = 10; // well below STOP_GAP (20), so a restore always finds every payment
     fn last_used_index(&self, k: KeychainKind) -> Option<u32> { // canonical txs only, so fakes do not count
         self.wallet.transactions().flat_map(|t| t.tx_node.tx.output.iter()
             .filter_map(|o| self.wallet.derivation_of_spk(o.script_pubkey.clone())).collect::<Vec<_>>())
             .filter(|(kk, _)| *kk == k).map(|(_, i)| i).max()
     }
     pub fn next_receive_address(&mut self) -> Result<Address, EngineError> {
         let k = KeychainKind::External;
         let first_unused = self.last_used_index(k).map_or(0, |u| u + 1);
         let run = self.wallet.derivation_index(k).map_or(0, |r| (r + 1).saturating_sub(first_unused));
         let a = if run >= MAX_UNUSED_RECEIVE { self.wallet.peek_address(k, first_unused).address }
                 else { self.wallet.reveal_next_address(k).address };
         self.persist()?;
         Ok(a)
     }
     ```
- **Interfaces changed:** `WrenBalance`, `balance(&ChainParams)`, `TxSummary.confirmations`, `MAX_UNUSED_RECEIVE`, and the crate-visible `spendable::{is_own_tx, immature_coinbase, untrusted_unconfirmed}` (used by Tasks 13, 14, 18 and 20). Add `spendable.rs` to File Structure.

### Area D — key custody, engine API, PIN

- **Findings:** H2, ca-ruz #5
- **Supersedes:**
  - `pub const ACCOUNT_PATH: &str = "m/84'/0'/0'";` (now in custody; re-export it)
  - `pub fn generate_words() -> Result<Vec<String>, EngineError>;`
  - `pub fn create(db_path: &Path, network: Network, words: &str, passphrase: &str)…`
  - `pub fn open(db_path: &Path, network: Network, words: &str, passphrase: &str)…`
  - `pub(crate) fn secret_key_for(&self, input: &psbt::Input) -> Result<SecretKey, EngineError>;`
  - `Keys: the mnemonic is parsed, turned into a seed (held in Zeroizing)…` (L3184)
  - The implementation blocks `struct SigningKeys {`, `impl Drop for SigningKeys {` and `fn keys_and_descriptors(`.
  - The tests `reopen_with_same_words_and_refuse_wrong_passphrase`, `generated_words_are_12_valid_bip39_words` and `rejects_invalid_and_24_word_phrases` (the last two move to 12a).
- **New failing tests first:**
  - `bip84_official_vector_first_address`: `WrenWallet::create(&p, Bitcoin, &KeySource::from_seed(&SeedSecret::from_words(ABANDON…)…))`; the first address is `bc1qcr8te4kr609gcawutmrza0j4xv80jy8z306fyu`.
  - `open_is_watch_only_and_keeps_fingerprint`: `open(&p, Bitcoin)` with no words; `fingerprint()` equals the `KeySource` fingerprint; the first address is the same.
  - `create_refuses_to_overwrite`: unchanged intent.
  - `meta_flags_default_false_and_persist`: set `BackupConfirmed`, reopen, read it back as true.
- **Changes:**
  1. Replace the "Keys:" paragraph (L3184) with:
     > Keys: `WrenWallet` is watch-only. It is built from `KeySource::descriptors()` (Task 12a) and never holds a private key. The SQLite file holds public descriptors with key origin `[fingerprint/84'/0'/0']`, plus a `wren_meta` table. Signing keys exist only inside a `KeySource` for the duration of one engine call.
  2. New interface:
     ```rust
     pub use crate::custody::ACCOUNT_PATH;
     #[derive(Clone, Copy, Debug, PartialEq, Eq)]
     pub enum MetaFlag { BackupConfirmed, BtcHistoryDetected, BtcSweepAcknowledged, BtcSweepUsed }
     impl WrenWallet {
         pub(crate) fn create(db_path: &Path, network: Network, keys: &KeySource) -> Result<Self, EngineError>; // fails if a wallet exists; stores fingerprint in wren_meta
         pub fn open(db_path: &Path, network: Network) -> Result<Self, EngineError>;                         // stored descriptors; check_network
         pub fn exists(db_path: &Path) -> bool;
         pub fn fingerprint(&self) -> Fingerprint;
         pub fn flag(&self, f: MetaFlag) -> Result<bool, EngineError>;
         pub fn set_flag(&mut self, f: MetaFlag, v: bool) -> Result<(), EngineError>;
         // next_receive_address, balance, transactions, persist, bdk, bdk_mut: unchanged
     }
     ```
  3. Implementation notes:
     - `create`: `let (ext, int) = keys.descriptors();`, then the existing create code. After that: `CREATE TABLE IF NOT EXISTS wren_meta (key TEXT PRIMARY KEY, value TEXT NOT NULL)` and insert `fingerprint`.
     - `open`: `Wallet::load().check_network(network).load_wallet(&mut db)` with no `.descriptor(..)`. Read the fingerprint from `wren_meta`.
     - Flags are stored as `"1"`/`"0"`; a missing flag reads as `false`.
     - Delete `SigningKeys`, `keys_and_descriptors`, `generate_words` and `secret_key_for`, and the `keys` field.
  4. Expected test count: PASS (4 tests).
- **Interfaces changed:**
  - `WrenWallet::create` and `WrenWallet::open` signatures.
  - Added `exists`, `fingerprint`, `flag`, `set_flag` and `MetaFlag`.
  - Removed `secret_key_for` and `generate_words`.

## Task 13

### Area A — headers, sync, chain selection

- **Findings:**
  - ca-ruz #2
  - chrisguida Lows: "Pre-anchor headers", "Merkle proofs ignore the v2 transaction count", "Integer handling" (history heights)
  - Decision-file hook: pre-split history detection
- **Supersedes:**
  - The whole **Trust note** paragraph (L3433). The replacement is below.
  - `pub(crate) async fn fetch_confirmed_update(…) -> Result<Update, EngineError>` and `pub async fn sync_wallet(…) -> Result<(), EngineError>`.
  - L3578: `let confirmed: Vec<(Txid, u32)> = found.iter().filter(|(_, h)| **h > 0 && (**h as u32) <= tip)…`
  - L3585: the `else` arm of `let (hash, header) = if *height >= base {` (the unproven `client.header(*height)`).
  - L3595: `if !verify_merkle(txid, &proof, &header.merkle_root) {`
  - L3603: `if *h <= 0 || (*h as u32) > tip {`
  - The transaction fetch `for tx in client.transactions(&missing).await? {`. It moves after the proofs so unverified transactions are left out.
  - Test setup L3470 (`sync_headers` arguments).
  - `Expected: PASS (4 tests).` (L3644)
- **New failing tests first:**
  - `pre_anchor_payment_with_valid_proof_is_confirmed`: 100 synthetic pre-anchor headers. Height 50 has `merkle_root = txid` of a payment to the wallet, and the params use `tc.params_with_pre_anchor(&pre)`. After sync, `confirmed == amount` and the confirmation height is 50.
  - `fabricated_pre_anchor_header_is_not_confirmed`: the honest tree's header at 50 has an unrelated root. The server serves a fabricated header at 50 (`merkle_root = txid`) together with the honest branch. Result is `Err(Chain(_))` and `balance().total() == 0`.
  - `pre_anchor_payment_without_cp_support_is_unverified`: the world has `cp_supported = false`. Sync returns `Ok`. `confirmed == 0`, the transaction is not in `transactions()`, and `report.unverified_pre_split == [(txid, 50)]`.
  - `v2_proof_deeper_than_txcount_is_rejected`: block 110's root is `sha256d(txid ‖ sib)`, and the proof is `(110, 0, [sib])` while the header commits `txcount 1`. Result is `Err(Chain(_))`. The old code accepts it.
  - `report_flags_pre_split_history`: any history item with `0 < height < split_height` sets `report.pre_split_history`, whether proven or not.
  - Expected count goes from 4 to 9.
- **Changes:**
  1. **testutil (append):**
     ```rust
     pub fn header_tree(leaves: &[[u8; 32]], pos: u32) -> ([u8; 32], Vec<[u8; 32]>); // Bitcoin/Electrum pairing, odd level duplicates last
     impl TestChain { pub fn params_with_pre_anchor(&self, pre: &[Vec<u8>]) -> ChainParams; } // root over sha256d(pre[0..anchor]), cp_height = anchor - 1
     // MockWorld: pub pre_anchor: Vec<Vec<u8>> (raw headers, index = height), pub pre_anchor_served: HashMap<u32, Vec<u8>>, pub cp_supported: bool (default true)
     // serve "blockchain.block.header":
     //   h < anchor with no cp param -> pre_anchor[h]
     //   with p[1] > 0 -> if !cp_supported Err("unsupported"), else
     //   {"header": served-or-honest hex, "branch": header_tree branch display-hex, "root": display-hex}
     ```
  2. **Return types:**
     ```rust
     pub struct WalletSyncReport { pub pre_split_history: bool, pub unverified_pre_split: Vec<(Txid, u32)> }
     pub(crate) struct VerifiedUpdate { pub update: Update, pub unverified_pre_split: Vec<(Txid, u32)> }
     pub async fn sync_wallet(client: &ElectrumClient, chain: &HeaderChain, wallet: &mut Wallet, now: u64) -> Result<WalletSyncReport, EngineError>;
     pub(crate) async fn fetch_confirmed_update(/* unchanged args */) -> Result<VerifiedUpdate, EngineError>;
     ```
  3. **Heights:** convert history heights with `u32::try_from(*h)`. An out-of-range positive height returns `Err(EngineError::Chain("history height out of range"))`.
  4. **Proof loop:** take the header from the verified chain at or above the anchor; below it, use the proof helper:
     ```rust
     let header = if *height >= base { chain.header_at(*height).unwrap().clone() } else {
         match pre_anchor_header(client, chain.params(), *height, &mut proven).await? {
             Some(h) => h,
             None => { unverified.push((*txid, *height)); continue; }
         }
     };
     if !verify_merkle(txid, &proof, &header.merkle_root, header.v2.as_ref().map(|v| v.txcount as u32)) {
         return Err(EngineError::Chain("merkle proof does not match block"));
     }
     ```
     The helper:
     ```rust
     /// Ok(None) = this server cannot prove the height (no pinned root, or a JSON-RPC error reply such as electrs'
     /// one-argument blockchain.block.header) -> "unverified". A proof that is present but wrong fails the sync.
     async fn pre_anchor_header(client: &ElectrumClient, p: &ChainParams, height: u32,
                                cache: &mut HashMap<u32, Option<ForkHeader>>) -> Result<Option<ForkHeader>, EngineError> {
         if let Some(h) = cache.get(&height) { return Ok(h.clone()); }
         let Some(cp) = &p.pre_anchor_root else { cache.insert(height, None); return Ok(None) };
         let got = match client.header_with_proof(height, cp.cp_height).await {
             Ok((raw, branch)) => {
                 if !verify_header_proof(&raw, height, &branch, cp) {
                     return Err(EngineError::Chain("pre-split header proof does not match"));
                 }
                 Some(ForkHeader::parse(&raw)?)
             }
             Err(EngineError::Server(_)) => None,
             Err(e) => return Err(e),
         };
         cache.insert(height, got.clone());
         Ok(got)
     }
     ```
     - The `Server` error arm must match however the error-handling area finally types "JSON-RPC error reply".
     - The transaction fetch, `seen_ats` and the LocalChain `blocks` all skip transactions listed in `unverified`.
     - `pre_split_history = found.values().any(|h| *h > 0 && *h < p.split_height as i64)`.
  5. **Replacement Trust note:**
     > **Trust note (Gate A, amended):** confirmations at or above the anchor (961,632) are proven against headers the engine verified itself, and v2 merkle proofs must match the transaction count the header commits to. A confirmation **below** the anchor counts only if the server proves that block's header with `blockchain.block.header(height, 961631)` against the header-merkle root pinned in `ChainParams` (the same root on both chains, because history to 961,631 is shared). A wrong proof fails the sync. A server that cannot give the proof (no `cp_height` support, e.g. electrs) gets those transactions reported as **unverified** (`WalletSyncReport.unverified_pre_split`): they are left out of balance and history and are not claimable. Any history below the split also sets `pre_split_history`, which the restore policy uses to block a BTC seed.
- **Interfaces changed:**
  - `sync_wallet` returns `WalletSyncReport`.
  - `fetch_confirmed_update` returns `VerifiedUpdate`. Task 17 L4831 uses `.update` and passes `unverified_pre_split` to the claim report as the reason "this server can't prove this pre-split coin". That reason belongs to the claim area.
  - New testutil helpers: `header_tree`, `params_with_pre_anchor`, and the `MockWorld` fields above.

### Area B — server trust in everyday use

- **Findings:** H1, M6 (eviction), M10 (rebroadcast)
- **Supersedes:**
  - Interfaces L3429-3430 `pub(crate) async fn fetch_confirmed_update(client: &ElectrumClient, chain: &HeaderChain, genesis: BlockId,`
  - Impl L3558 (same signature)
  - The seen-at loop L3602-3606 `for (txid, h) in found {`
  - Test asserts L3479 `w.balance().confirmed`, L3495 `w.balance().untrusted_pending`, L3504 `w.balance().total()`
  - L3637 `wallet.apply_update(update).map_err(...)` (now followed by a rebroadcast step)
- **New failing tests first** (in `wallet_sync_tests`, using the existing `setup()`; its coin is `funding:0` at External/3):
  - `server_invented_spend_of_our_coin_is_refused`: the server lists an unconfirmed RBF tx spending `funding:0` to a foreign script with witness `[[0x30,0x21],[2;33]]`. `sync_wallet` returns `Err(Server(InvalidTransaction))`, the graph has no such tx, and `spendable` is still 50,000.
  - `own_signed_spend_reported_by_server_is_admitted`: sign the same spend with `sign_p2wpkh_unified`, using the key for m/84'/0'/0'/0/3 derived inline from ABANDON. It is admitted, and the coin no longer appears in `list_unspent`.
  - `foreign_unconfirmed_dropped_by_server_is_evicted`: sync once with an input-less unconfirmed payment, remove it from history, sync again. Then `incoming_pending == 0`.
  - `own_unconfirmed_missing_from_server_is_rebroadcast`: apply an own signed tx locally that is absent from server history. After `sync_wallet`, `world.broadcasts` contains the same bytes.
  - Update the existing asserts to `w.balance(&tc.params()).spendable`, `.incoming_pending`, and `spendable + incoming_pending + immature == 0`.
- **Changes:**
  - New signatures:
    ```rust
    pub(crate) struct FetchResult { pub update: Update, pub unconfirmed_seen: usize, pub own_missing: Vec<Arc<Transaction>> }
    pub(crate) async fn fetch_confirmed_update(client: &ElectrumClient, chain: &HeaderChain, wallet: &Wallet,
        found: &HashMap<Txid, i64>, admit_unconfirmed: bool, now: u64) -> Result<FetchResult, EngineError>;
    pub async fn sync_wallet(client: &ElectrumClient, chain: &HeaderChain, wallet: &mut Wallet, now: u64) -> Result<(), EngineError>; // unchanged
    ```
    `genesis`, `existing` and `have` are now derived inside from `wallet`.
  - Unconfirmed items (h ≤ 0 or h > tip):
    - if `!admit_unconfirmed`: neither fetched nor inserted; only counted in `unconfirmed_seen`;
    - otherwise each fetched one that spends a wallet output must pass this check before any `seen_at` is set:
    ```rust
    /// Only our key can spend our coins. An unconfirmed spend of one of them is admitted only if every such
    /// input carries a valid SIGHASH_ALL|UNIFIED P2WPKH signature from our key.
    pub(crate) fn verify_our_p2wpkh_inputs(tx: &Transaction, spent: &[TxOut], ours: &dyn Fn(&Script) -> bool) -> Result<(), EngineError> {
        let secp = Secp256k1::verification_only();
        let bad = || EngineError::Server(ServerError::InvalidTransaction);
        for (i, txin) in tx.input.iter().enumerate() {
            if !ours(&spent[i].script_pubkey) { continue; }
            let w: Vec<&[u8]> = txin.witness.iter().collect();
            if !txin.script_sig.is_empty() || w.len() != 2 || w[0].last() != Some(&(SIGHASH_ALL_UNIFIED as u8)) { return Err(bad()); }
            let pk = PublicKey::from_slice(w[1]).map_err(|_| bad())?;
            if spent[i].script_pubkey != ScriptBuf::new_p2wpkh(&CompressedPublicKey(pk).wpubkey_hash()) { return Err(bad()); }
            verify_ecdsa_unified(&secp, tx, i, spent, ScriptType::WitnessV0, &p2wpkh_script_code(&pk), w[0], &pk).map_err(|_| bad())?;
        }
        Ok(())
    }
    ```
  - `spent` comes from `resolve_prevouts(client, tx, known)`. `known` looks up `wallet.tx_graph().get_txout` and the txs in this update. Parents that are still missing are fetched with `client.transactions` (already txid-checked). A prevout that cannot be resolved means `InvalidTransaction`. UNIFIED commits to every input, so all prevouts are needed.
  - Eviction and rebroadcast: for each canonical unconfirmed tx in `wallet` whose txid is not in `found`:
    - `spendable::is_own_tx` → push to `own_missing`;
    - otherwise → `tx_update.evicted_ats.insert((txid, now))`.
  - `sync_wallet`, after `apply_update`: `for tx in own_missing { let _ = client.broadcast(&tx).await; }`. Same bytes, once per sync, and the result is ignored. Never evict own txs.
  - Trust note (L3433): append "Unconfirmed spends of the wallet's own coins are admitted only with a valid 0x21 signature by the wallet's key. Server-reported unconfirmed payments from foreign parents are shown as incoming and are never spendable."
- **Interfaces changed:** `fetch_confirmed_update` signature and `FetchResult` (**overlap: Task 17, Area D**), and the crate-visible `verify_our_p2wpkh_inputs` (used by Task 14).

### Area D — key custody, engine API, PIN

- **Findings:** M13
- **Supersedes:** `pub async fn sync_wallet(client: &ElectrumClient, chain: &HeaderChain, wallet: &mut Wallet, now: u64) -> Result<(), EngineError>;` (interface and implementation).
- **New failing tests first:**
  - `sync_reports_pre_split_history_from_reported_heights`: a MockWorld history item for a wallet script at height `params.split_height - 1` gives `report.pre_split_history == true`. This holds even when the merkle proof is missing or unverified.
  - `post_split_history_is_not_flagged`: items only at or above `split_height` (and unconfirmed `0`/`-1`) give `false`.
- **Changes:**
  - `pub struct SyncReport { pub pre_split_history: bool }`
  - `sync_wallet(..) -> Result<SyncReport, EngineError>`.
  - After both `scan_keychain` calls, compute `pre_split_history = found.values().any(|&h| h > 0 && (h as u64) < chain.params().split_height as u64)` before fetching proofs. It is deliberately based on the server-reported heights, so the check fails safe (blocks). Return it with the existing update.
- **Interfaces changed:** `sync_wallet` return type; new `SyncReport`.

## Task 14

### Area B — server trust in everyday use

- **Findings:** H1 (bump guard), H4, M6, M8 + ca-ruz #7 (RBF), M10, Low "Self-send", Low "Error strings"
- **Supersedes:**
  - Interfaces L3666-3678: `FeeEstimates`, `ReviewSummary`, `bump_fee`, `sign_and_broadcast`
  - L3898 `fn floor() -> FeeRate {` (keep it, add caps below it)
  - L3902-3908 `pub async fn fee_estimates(`
  - L3910-3923 `/// Coinbase outputs still immature ...` (moved to `spendable.rs`)
  - L3925-3947 `fn summarize(`
  - L3950-3953 `if fee_rate < floor() {` / `let exclude = immature_coinbase(w, params, tip);`
  - L3972-3980 `pub fn bump_fee(`
  - L4003-4010 `pub async fn sign_and_broadcast(`
  - Every `EngineError::Wallet(e.to_string())`
  - In tests: L3779 and L3823 `let txid = sign_and_broadcast(...).await.unwrap();`, L3824 `bump_fee(&mut f.wallet, &mut pending, txid, rate(5))`
- **New failing tests first:**
  - `absurd_server_fee_rates_fall_back_to_floor`: `relay_fee = 1.0`, `fee_estimate = 1.0`. All tiers `== floor()` and `unreliable == true`.
  - `builders_refuse_rates_above_cap`: `build_payment` and `bump_fee` at 1,001 sat/vB return `Err(Input("fee rate is too high"))`.
  - `review_flags_abnormal_fee`: send-all of 2,000 sats at 20 sat/vB gives `abnormal_fee` (fee ≥ 10%). 50,000 at 201 sat/vB also gives `abnormal_fee`.
  - `fake_incoming_unconfirmed_is_not_spendable`: a confirmed 10,000 coin plus an input-less unconfirmed 90,000 coin on External/5. `Exact(50_000)` fails with `InsufficientFunds`; `SendAll` spends only 10,000.
  - `own_unconfirmed_change_is_spendable`: pay 20,000 from 50,000 and apply it. A second `Exact(10_000)` uses the change.
  - `relay_policy_coinbase_is_not_spendable`: a coinbase at 110 with the long window off and `coinbase_relay_maturity = 6_480` is refused. With 100 it is allowed.
  - `bump_fee_never_adds_an_immature_coinbase`: coins are a plain 20,000 and a relay-immature 200,000 coinbase. Pay 15,000, then bump to 100 sat/vB. Expect `Err(InsufficientFunds)`, and the coinbase is never an input.
  - `bump_fee_refuses_a_tx_our_key_did_not_sign`: `apply_unconfirmed_txs` a garbage-witness spend of our coin. `bump_fee` returns `Err(Signing(_))`.
  - `lost_reply_keeps_payment_pending_and_inputs_locked` (`RecordThenFail`): outcome `Unknown(txid)`, the txid is in `transactions()` unconfirmed, and its inputs are not in `list_unspent`.
  - `explicit_rejection_is_reported_and_tx_kept` (`Reject("min relay fee not met")`): `Rejected{reason: FeeTooLow}`, the tx stays recorded, and `bump_fee(txid, ..)` re-spends the same inputs.
  - `self_send_review_shows_amount_and_recipient`: pay 20,000 to `peek_address(External, 5)`. Expect `amount == 20_000`, `recipient` is that address, and `to_self == true`.
  - Update the existing tests:
    - destructure `BroadcastOutcome::Sent(txid)`;
    - call `bump_fee(&mut f.wallet, &f.params, 349, &mut pending, txid, rate(5))`;
    - the insufficient-funds case expects `EngineError::InsufficientFunds { .. }`.
- **Changes:**
  ```rust
  pub const MAX_ESTIMATE_RATE: FeeRate = FeeRate::from_sat_per_vb_unchecked(500);
  pub const MAX_FEE_RATE: FeeRate = FeeRate::from_sat_per_vb_unchecked(1_000); // hard cap: payment, bump, claim
  pub const ABNORMAL_FEE_PERCENT: u64 = 10;
  pub const ABNORMAL_RATE: FeeRate = FeeRate::from_sat_per_vb_unchecked(200);
  pub struct FeeEstimates { pub slow: FeeRate, pub normal: FeeRate, pub fast: FeeRate, pub unreliable: bool }

  fn sane(r: Option<FeeRate>) -> Option<FeeRate> { r.map(|r| r.max(floor())).filter(|r| *r <= MAX_ESTIMATE_RATE) }
  pub fn check_fee_rate(r: FeeRate) -> Result<(), EngineError> {
      if r < floor() { return Err(EngineError::Input("fee rate is below 1 sat/vB")); }
      if r > MAX_FEE_RATE { return Err(EngineError::Input("fee rate is too high")); }
      Ok(())
  }
  pub async fn fee_estimates(client: &ElectrumClient) -> Result<FeeEstimates, EngineError> {
      let raw = [client.relay_fee().await?, client.estimate_fee(12).await?, client.estimate_fee(6).await?, client.estimate_fee(2).await?];
      let unreliable = raw.iter().any(|r| r.is_some() && sane(*r).is_none());
      let min = sane(raw[0]).unwrap_or(floor());
      let slow = sane(raw[1]).unwrap_or(min).max(min);
      let normal = sane(raw[2]).unwrap_or(slow).max(slow);
      let fast = sane(raw[3]).unwrap_or(normal).max(normal);
      Ok(FeeEstimates { slow, normal, fast, unreliable })
  }
  fn map_create_tx(e: CreateTxError) -> EngineError {
      match e { CreateTxError::CoinSelection(f) => EngineError::InsufficientFunds { available: f.available, needed: f.needed },
                CreateTxError::OutputBelowDustLimit(_) => EngineError::Input("amount is too small to send"),
                _ => EngineError::Wallet("could not build this transaction") }
  }
  fn map_bump(e: BuildFeeBumpError) -> EngineError {
      match e { BuildFeeBumpError::TransactionConfirmed(_) => EngineError::Input("this payment is already confirmed"),
                _ => EngineError::Wallet("this payment cannot be sped up") }
  }
  ```
  - `ReviewSummary` gains `pub abnormal_fee: bool, pub to_self: bool`.
  - New `summarize(w, psbt, id, fee_rate, recipient: Option<&Script>)`:
    ```rust
    for o in &psbt.unsigned_tx.output {
        let internal = matches!(w.bdk().derivation_of_spk(o.script_pubkey.clone()), Some((KeychainKind::Internal, _)));
        if internal && Some(o.script_pubkey.as_script()) != recipient { change += o.value; continue; }
        amount += o.value;
        to_self |= w.bdk().is_mine(o.script_pubkey.clone());
        recipient_text = Address::from_script(&o.script_pubkey, network).map(|a| a.to_string()).unwrap_or_default();
    }
    let fee = psbt.fee().map_err(|_| EngineError::Wallet("could not compute the fee"))?;
    let abnormal_fee = fee.to_sat() * 100 >= amount.to_sat() * ABNORMAL_FEE_PERCENT || fee_rate > ABNORMAL_RATE;
    ```
  - `build_payment`:
    - start with `check_fee_rate(fee_rate)?`;
    - exclude `spendable::immature_coinbase(w.bdk(), params, tip)` ∪ `spendable::untrusted_unconfirmed(w.bdk())`;
    - use `map_create_tx`;
    - pass `Some(&recipient.script_pubkey())` to `summarize`.
  - `bump_fee(w, params, tip, pending, txid, fee_rate)` ("Speed up" and "Replace" are this one function; BDK keeps every original input):
    ```rust
    check_fee_rate(fee_rate)?;
    let tx = w.bdk().tx_graph().get_tx(txid).ok_or(EngineError::Input("unknown payment"))?;
    let spent: Vec<TxOut> = tx.input.iter().map(|i| w.bdk().tx_graph().get_txout(i.previous_output).cloned()
        .filter(|o| w.bdk().is_mine(o.script_pubkey.clone()))
        .ok_or(EngineError::Signing("only this wallet's own payments can be sped up"))).collect::<Result<_, _>>()?;
    crate::sync::verify_our_p2wpkh_inputs(&tx, &spent, &|s| w.bdk().is_mine(s.to_owned()))
        .map_err(|_| EngineError::Signing("only this wallet's own payments can be sped up"))?;
    let exclude = spendable::immature_coinbase(w.bdk(), params, tip);
    let mut b = w.bdk_mut().build_fee_bump(txid).map_err(map_bump)?;
    b.fee_rate(fee_rate).sighash(..).set_exact_sequence(..).unspendable(exclude);
    // finish().map_err(map_create_tx); summarize(.., None)
    ```
  - Broadcast ordering:
    ```rust
    #[derive(Clone, Debug, PartialEq, Eq)]
    pub enum BroadcastOutcome { Sent(Txid), Rejected { txid: Txid, reason: RejectKind }, Unknown(Txid) }
    /// Err means nothing was sent. Any Ok means the signed tx is recorded as pending, its inputs stay locked,
    /// and Wren rebroadcasts it on sync until it confirms or is replaced (bump_fee).
    pub async fn sign_and_broadcast(w: &mut WrenWallet, pending: &mut PendingPayments, review_id: u64,
        client: &ElectrumClient, now: u64) -> Result<BroadcastOutcome, EngineError> {
        let psbt = pending.take(review_id).ok_or(EngineError::Input("this payment review has expired; review it again"))?;
        let tx = sign_pending(w, &psbt)?;
        let txid = tx.compute_txid();
        w.bdk_mut().apply_unconfirmed_txs([(tx.clone(), now)]);
        if let Err(e) = w.persist() {
            w.bdk_mut().apply_evicted_txs([(txid, now)]); // not sent: unlock the inputs again
            return Err(e);
        }
        Ok(match client.broadcast(&tx).await {
            Ok(_) => BroadcastOutcome::Sent(txid),
            Err(EngineError::Server(ServerError::Rejected(k))) if k != RejectKind::Other => BroadcastOutcome::Rejected { txid, reason: k },
            Err(_) => BroadcastOutcome::Unknown(txid), // a timeout, a dropped connection or unrecognised text may hide a relayed tx
        })
    }
    ```
- **Interfaces changed:**
  - `FeeEstimates.unreliable`;
  - `ReviewSummary.{abnormal_fee, to_self}`;
  - `check_fee_rate` and the cap constants (used by Task 18);
  - `bump_fee(w, params, tip, pending, txid, rate)`;
  - `sign_and_broadcast -> Result<BroadcastOutcome, _>`;
  - `BroadcastOutcome` (used by Tasks 19 and 20).

### Area D — key custody, engine API, PIN

- **Findings:** H2, ca-ruz #5, M13 (single send-all)
- **Supersedes:**
  - `pub(crate) fn sign_pending(w: &WrenWallet, psbt: &Psbt) -> Result<Transaction, EngineError>;`
  - `pub async fn sign_and_broadcast(w: &mut WrenWallet, pending: &mut PendingPayments, review_id: u64,`
  - `let sk = w.secret_key_for(&psbt.inputs[i])?;` (L3992)
  - `pub struct Funded { pub dir: tempfile::TempDir, pub wallet: WrenWallet, …`
- **New failing tests first:**
  - `sign_pending_refuses_input_without_wallet_key`: a PSBT input whose origin fingerprint is foreign gives `Signing("no key in this wallet for an input")`.
  - `pending_remembers_review_kind_and_unsigned_txid`: `build_payment(SendAll)` reports kind `SendAll`, and `unsigned_txid(id)` equals the PSBT's unsigned-tx txid.
- **Changes:**
  - `sign_pending(keys: &KeySource, psbt: &Psbt)`. In the loop: `keys.with_key_for(&psbt.inputs[i], |sk| sign_p2wpkh_unified(&secp, &mut tx, i, &spent, sk))?;`
  - `sign_and_broadcast(w: &mut WrenWallet, keys: &KeySource, pending: &mut PendingPayments, review_id: u64, client: &ElectrumClient, now: u64)`. Keep the return type set by the broadcast-outcome amendment.
  - `#[derive(Clone, Copy, Debug, PartialEq, Eq)] pub enum ReviewKind { Payment, SendAll, Bump }`.
  - `PendingPayments` stores `(Psbt, ReviewKind)`. Add `pub(crate) fn kind(&self, id: u64) -> Option<ReviewKind>` and `pub(crate) fn unsigned_txid(&self, id: u64) -> Option<Txid>`.
  - `build_payment` inserts `Payment` or `SendAll`; `bump_fee` inserts `Bump`.
  - `testutil::Funded` gains `pub keys: KeySource` (from ABANDON). Existing signing tests pass `&f.keys`.
- **Interfaces changed:** `sign_pending`, `sign_and_broadcast`, `PendingPayments` (`kind`, `unsigned_txid`), new `ReviewKind`, `Funded.keys`.

## Task 15

### Area B — server trust in everyday use

- **Findings:** Low "Destinations over 34 bytes"
- **Supersedes:** L4146-4149 `let address = Address::from_str(addr)` … `.map_err(|_| EngineError::Input("address is for a different network"))?;`
- **New failing tests first:**
  - `refuses_unknown_witness_versions_and_anchors`: `xbt:` links and bare forms of `bc1pw508d6qejxtdg4y5r3zarvary0c5xw7kw508d6qejxtdg4y5r3zarvary0c5xw7kt5nd6y` (v1, 40-byte), `BC1SW50QGDZ25J` (v16) and `bc1pfeessrawgf` (P2A) all return `Err(Input(_))`. P2PKH, P2SH, P2WPKH, P2WSH and P2TR are still accepted.
- **Changes:**
  ```rust
  /// Only output types the fork relays and that are not anyone-can-spend (spk <= 34 bytes; RDTS consensus and
  /// Knots mempool, validation.cpp:1021). Unknown witness versions and P2A anchors are refused.
  pub(crate) fn require_payable(a: Address) -> Result<Address, EngineError> {
      use bdk_wallet::bitcoin::AddressType::*;
      match a.address_type() {
          Some(P2pkh | P2sh | P2wpkh | P2wsh | P2tr) if a.script_pubkey().len() <= 34 => Ok(a),
          _ => Err(EngineError::Input("Wren cannot send to this kind of address")),
      }
  }
  ```
  Call `require_payable` after `require_network` in `parse_payment_link`.
- **Interfaces changed:** the crate-visible `require_payable`, also used by Task 20 `make_payment_link`.

### Area E — Claim, links, QR, tests, CI

- **Findings:** ca-ruz #8; Low payment links (control and bidi characters, duplicate parameters, bare address); new: `%+1`
- **Supersedes:**
  - `fn has_prefix_ci(s: &str, p: &str) -> bool {` (L4115-4117).
  - `let h = s.get(i + 1..i + 3).ok_or(...` (L4125).
  - `"label" => label = Some(percent_decode(v)?),` (L4155).
  - The `PaymentLink` struct (L4040 and L4109-4113).
  - Step 4 "PASS (4 tests)".
- **New failing tests first:**
  - `non_ascii_input_is_an_error_not_a_panic`: `["☃☃☃","ab☃","xb☃","bitcoi☃","₿itcoin:x","xbt:☃"]` are all `Err`, and `format!("{}☃","x".repeat(n))` for n in 0..10 never panics.
  - `duplicate_params_refused`: `?amount=0.001&amount=1` returns `Err`.
  - `percent_escape_requires_two_hex_digits`: `label=%+1` returns `Err`.
  - `label_strips_control_and_bidi`: `label=a%E2%80%AEb%07c` gives `"abc"`, capped at 100 characters.
  - `bare_address_is_flagged`: `explicit_xbt` is `false` for a bare address and `true` for `xbt:`/`XBT:`.
- **Changes:**
  ```rust
  fn has_prefix_ci(s: &str, p: &str) -> bool {
      s.as_bytes().get(..p.len()).is_some_and(|b| b.eq_ignore_ascii_case(p.as_bytes()))
  }
  // `&t[4..]` stays: the first 4 bytes matched ASCII "xbt:", so byte 4 is a char boundary.
  // percent_decode:
  let h = s.get(i + 1..i + 3).filter(|h| h.bytes().all(|b| b.is_ascii_hexdigit()))
      .ok_or(EngineError::Input("bad escape in link"))?;
  // parse loop: let mut seen = std::collections::HashSet::new();
  if !seen.insert(k) { return Err(EngineError::Input("this link repeats a field")); }
  "label" => label = Some(clean_label(percent_decode(v)?)),
  fn clean_label(s: String) -> String {
      s.chars().filter(|c| !c.is_control() && !matches!(c, '\u{200E}' | '\u{200F}' | '\u{061C}'
          | '\u{202A}'..='\u{202E}' | '\u{2066}'..='\u{2069}')).take(100).collect()
  }
  ```
  Set `explicit_xbt = has_prefix_ci(t, "xbt:")`. Step 4 now expects "PASS (9 tests)".
- **Interfaces changed:** `PaymentLink { address, amount, label, explicit_xbt: bool }`. The app shows the bare-address warning (spec §3.4) when `explicit_xbt` is false.

## Task 16

### Area A — headers, sync, chain selection

- **Findings:** chrisguida Low "Integer handling" (UR path). Coordinate with the area that owns ca-ruz #10 and UR import.
- **Supersedes:**
  - L4565 `for pair in comps.chunks(2) {`
  - L4566 `let idx = as_u64(&pair[0]).ok_or_else(bad)? as u32;`
  - L4567 `let hardened = matches!(pair.get(1), Some(Value::Bool(true)));`
  - L4570 `let parent = map_get(m, 8).and_then(as_u64).unwrap_or(0) as u32;`
  - L4586 `… as u32).to_be_bytes());` (the master fingerprint)
- **New failing tests first:**
  - `ur_path_with_odd_length_is_refused`
  - `ur_path_index_beyond_u32_is_refused` (index 2^32+84)
  - `ur_fingerprint_beyond_u32_is_refused`
- **Changes:**
  - `if comps.len() % 2 != 0 { return Err(bad()) }`
  - Then `u32::try_from(idx).map_err(|_| bad())?`.
  - Then `let Value::Bool(hardened) = pair[1] else { return Err(bad()) };`.
  - Use `u32::try_from` for the parent and master fingerprints.
- **Interfaces changed:** none.

### Area E — Claim, links, QR, tests, CI

- **Findings:**
  - ca-ruz #10;
  - **N1** (the tag-308 crypto-output wrapper);
  - chrisguida Low "Descriptor import" (`tr` tree, wrapper/version conflicts, multisig);
  - chrisguida Integer Low (`idx as u32`, odd-length paths);
  - chrisguida M14 (QR part).
- **Supersedes:**
  - `let idx = as_u64(&pair[0]).ok_or_else(bad)? as u32;` (L4566) and `for pair in comps.chunks(2) {` (L4565).
  - `network: NetworkKind::from(network),` (L4573).
  - `Value::Tag(404, k) => hdkey_to_account(...` and the `_ => continue,` arms (L4591-4598).
  - `let script_type = wrapper_type.or(implied).or_else(|| type_from_purpose(&path)).ok_or_else(bad)?;` (L4545).
  - `let key = key_and_suffix.split('/').next().unwrap_or("");` (L4542).
  - In `ur_crypto_account_bcr_2020_015`: `Value::Tag(404, Box::new(hdkey(84))),` (L4337-4338), unwrapped outputs.
  - `if lower.matches('/').count() >= 2 {` (L4411) runs with no bounds.
  - Step 5 "PASS (6 tests)".
- **New failing tests first:**
  - `ur_crypto_account_bcr_2020_015`: rewrite it with every output wrapped in `Value::Tag(308, …)`.
  - `bcr_2020_015_paper_vector`: the paper's CBOR hex (master `37b5eed4`) gives the P2PKH, P2SH-P2WPKH, P2WPKH and P2TR accounts with the paper's xpubs.
  - `ur_crypto_account_testnet_use_info_is_refused_on_mainnet`: `(5, Tag(305, {2: 1}))` is refused on `Bitcoin` and accepted on `Testnet`.
  - `ur_use_info_non_btc_type_refused`: `{1: 60}` is refused.
  - `ur_depth_and_source_fingerprint_must_agree`: origin `3: 4` with 3 components is refused, and origin `2: <other fp>` is refused.
  - `ur_odd_length_or_oversized_component_refused`: `[84,true,0,true,0]` and an index of `2^32` are refused.
  - `ur_private_or_master_hdkey_refused`: `2: true` or `1: true` is refused.
  - `ur_multisig_output_refuses_whole_import`: a 308(401(…)) next to a 308(404(…)) gives `Err` naming multisig.
  - `text_depth_and_child_must_match_path`: `[73c5da0a/84h/0h]` plus the depth-3 zpub is refused.
  - `tr_with_script_tree_refused`: `tr([…]xpub/0/*,{pk(…)})` is refused.
  - `wrapper_or_purpose_conflicts_with_key_version_refused`: `wpkh([…/49h/0h/0h]ypub…)` and `[…/84h/0h/0h]ypub…` are refused.
  - `text_multisig_refused_by_name`: `wsh(multi(…))` and `sh(sortedmulti(…))` give the multisig message.
  - `suffix_must_be_wildcard_chain`: `wpkh([…]xpub/0/5)` is refused, while `/0/*`, `/1/*` and `/<0;1>/*` are accepted.
  - `absurd_fragment_counts_are_refused_before_decoding`: forged CRC-valid frames for (4294967295, 4294967294), (1000001, 1000000) and (2002, 2001) all return `Err(Input)`. Build them with ciborium + `ur::bytewords::encode(.., Style::Minimal)`.
  - `endless_distinct_frames_stop_the_scan`: `MAX_UR_FRAMES + 1` valid fountain parts that never complete return `Err`.
  - `mixed_ur_types_refused`: a `crypto-account` frame after `crypto-psbt` frames returns `Err`.
- **Changes:**
  ```rust
  // ur.rs
  pub const MAX_UR_FRAGMENTS: u64 = 2_000;      // placeholder; set from the Task 22a SeedSigner measurement
  pub const MAX_UR_FRAMES: usize = 4 * MAX_UR_FRAGMENTS as usize + 100;
  // QrCollector gains `frames: usize`. In receive(), multipart branch, before self.decoder.receive:
  let idx = lower[3..].split('/').nth(1).ok_or_else(|| qr(()))?;
  let (s, c) = idx.split_once('-').ok_or_else(|| qr(()))?;
  let (seq, count): (u64, u64) = (s.parse().map_err(qr)?, c.parse().map_err(qr)?);
  if count == 0 || count > MAX_UR_FRAGMENTS || seq == 0 || seq > u32::MAX as u64 {
      return Err(EngineError::Input("this QR code is too large"));
  }
  if self.ur_type.as_deref().is_some_and(|t| t != ty) {
      return Err(EngineError::Input("these QR codes belong to different items; start again"));
  }
  self.frames += 1;
  if self.frames > MAX_UR_FRAMES { return Err(EngineError::Input("this QR code isn't completing; start again")); }
  ```
  ```rust
  // xpub_import.rs — from_crypto_account: accept an optional outer Tag(311|40311), then per output:
  let o = match o { Value::Tag(308 | 40308, inner) => inner.as_ref(), other => other };
  let parsed = match o {
      Value::Tag(404 | 40404, k) => hdkey_to_account(k, master, ClaimScriptType::P2wpkh, network),
      Value::Tag(403 | 40403, k) => hdkey_to_account(k, master, ClaimScriptType::P2pkh, network),
      Value::Tag(409 | 40409, k) => hdkey_to_account(k, master, ClaimScriptType::P2trKeyPath, network),
      Value::Tag(400 | 40400, inner) if matches!(inner.as_ref(), Value::Tag(404 | 40404, _)) => { /* P2shP2wpkh */ }
      Value::Tag(400 | 401 | 40400 | 40401, _) => return Err(EngineError::Input(
          "this export includes a multisig wallet, which Wren can't claim; export only the single-signature account")),
      _ => return Err(EngineError::Input("this export includes a wallet type Wren can't claim")),
  };
  ```
  ```rust
  // hdkey_to_account additions
  if matches!(map_get(m, 1), Some(Value::Bool(true))) || matches!(map_get(m, 2), Some(Value::Bool(true))) {
      return Err(EngineError::Input("Wren never accepts private keys; export the public account key"));
  }
  let (coin, net) = match map_get(m, 5) {
      None => (0, 0), // BCR-2020-007 default: mainnet BTC
      Some(Value::Tag(305 | 40305, ci)) => match ci.as_ref() {
          Value::Map(cm) => (map_get(cm, 1).map_or(Some(0), as_u64).ok_or_else(bad)?,
                             map_get(cm, 2).map_or(Some(0), as_u64).ok_or_else(bad)?),
          _ => return Err(bad()),
      },
      _ => return Err(bad()),
  };
  if coin != 0 { return Err(EngineError::Input("this is not a bitcoin-type key")); }
  let kind = match net { 0 => NetworkKind::Main, 1 => NetworkKind::Test, _ => return Err(bad()) };
  if kind != NetworkKind::from(network) { return Err(EngineError::Input("this key is for a different network")); }
  if comps.is_empty() || comps.len() % 2 != 0 { return Err(bad()); }
  for pair in comps.chunks_exact(2) {
      let idx = u32::try_from(as_u64(&pair[0]).ok_or_else(bad)?).map_err(|_| bad())?;
      let Value::Bool(hardened) = pair[1] else { return Err(bad()) };
      /* push */
  }
  if let Some(d) = map_get(om, 3) { if as_u64(d) != Some(children.len() as u64) { return Err(bad()); } }
  if let Some(s) = map_get(om, 2) { if as_u64(s) != Some(u32::from_be_bytes(master.to_bytes()) as u64) { return Err(bad()); } }
  // Xpub { network: kind, … }; then check_account_shape + purpose check (below)
  ```
  ```rust
  // shared: called by from_text and hdkey_to_account
  fn check_account_shape(x: &Xpub, path: &DerivationPath, ty: ClaimScriptType) -> Result<(), EngineError> {
      let c: &[ChildNumber] = path.as_ref();
      if c.is_empty() || x.depth as usize != c.len() || x.child_number != c[c.len() - 1] || !c.iter().all(|n| n.is_hardened()) {
          return Err(EngineError::Input("the key doesn't match its path; export the account key again"));
      }
      if type_from_purpose(path).is_some_and(|p| p != ty) {
          return Err(EngineError::Input("the key's path is for a different wallet type"));
      }
      Ok(())
  }
  // from_text, before parsing:
  let l = t.to_ascii_lowercase();
  if ["multi(", "sortedmulti(", "wsh("].iter().any(|p| l.contains(p)) {
      return Err(EngineError::Input("multisig wallets can't be claimed in v1; export a single-signature account"));
  }
  if l.starts_with("tr(") && l.contains(',') {
      return Err(EngineError::Input("Taproot wallets with scripts can't be claimed in v1"));
  }
  // key and suffix:
  let (key, suffix) = key_and_suffix.split_once('/').map_or((key_and_suffix, None), |(k, s)| (k, Some(s)));
  if !matches!(suffix, None | Some("0/*") | Some("1/*") | Some("<0;1>/*")) {
      return Err(EngineError::Input("export the account key, not a single address"));
  }
  // script type: all present sources must agree
  let mut ts = [wrapper_type, implied].into_iter().flatten();
  let script_type = ts.next().or_else(|| type_from_purpose(&path)).ok_or_else(bad)?;
  if ts.any(|t| t != script_type) { return Err(EngineError::Input("the key's format and its descriptor name different wallet types")); }
  check_account_shape(&xpub, &path, script_type)?;
  ```
  Step 5 now expects "PASS (≈22 tests)". Change the L4351 note to: "Real SeedSigner exports are fixtures in Task 22a."
- **Interfaces changed:**
  - `ur::{MAX_UR_FRAGMENTS, MAX_UR_FRAMES}`.
  - `import_accounts` refuses (does not skip) unsupported outputs.
  - Error messages are named.

## Task 17

### Area B — server trust in everyday use

- **Findings:** H1 (claim scans)
- **Supersedes:**
  - L4827-4831, from `let genesis = BlockId { height: 0, hash: wallet.local_chain().genesis_hash() };` through `let mut update = fetch_confirmed_update(client, chain, genesis, &existing, &found, &have, now).await?;`
  - L4840-4842 `} else { summary.unconfirmed_ignored += 1; }`
- **New failing tests first:**
  - `claim_scan_ignores_server_invented_unconfirmed_spend`: the coin is confirmed at 104, and the server also lists an unconfirmed garbage spend of it. The scan still reports 1 coin and `unconfirmed_ignored == 1`.
- **Changes:**
  - `let r = fetch_confirmed_update(client, chain, wallet, &found, /*admit_unconfirmed=*/ false, now).await?;`
  - `summary.unconfirmed_ignored += r.unconfirmed_seen;`
  - Apply `r.update`. `list_unspent()` now holds confirmed coins only.
- **Interfaces changed:** none new. Overlap: Area D edits the same `scan`; merge onto the new call shape.

### Area E — Claim, links, QR, tests, CI

- **Findings:** chrisguida M4 (honest empty and success reporting) and M5 (isolated session)
- **Supersedes:**
  - `pub struct FoundSummary { pub total: Amount, pub coins: usize, pub by_type: …, pub unconfirmed_ignored: usize }` (L4646, L4787-4793).
  - `pub struct ClaimSession { … }` (L4795-4799).
- **New failing tests first:**
  - `summary_reports_gap_and_types_scanned`: after `scan(…, false)`, `gap_used == 20` and `types_scanned == [P2wpkh]`; after `scan(…, true)`, `gap_used == 200`.
  - `unconfirmed_coins_are_listed_as_not_claimable`: one unconfirmed coin appears in `not_claimable` as `(P2wpkh, Unconfirmed, 1, amt)`.
- **Changes:**
  ```rust
  #[derive(Clone, Copy, Debug, PartialEq, Eq)]
  pub enum NotClaimedReason { Unconfirmed, Uneconomic, TooLargeForQr, Immature /* Area D */, Unverified /* Area B */ }
  #[derive(Clone, Debug, PartialEq, Eq)]
  pub struct NotClaimed { pub script_type: ClaimScriptType, pub reason: NotClaimedReason, pub coins: usize, pub amount: Amount }
  pub struct FoundSummary { pub total: Amount, pub coins: usize, pub by_type: Vec<(ClaimScriptType, Amount, usize)>,
      pub not_claimable: Vec<NotClaimed>, pub gap_used: u32, pub types_scanned: Vec<ClaimScriptType> }
  pub struct ClaimSession { pub(crate) network: Network, pub(crate) wallets: Vec<(ClaimScriptType, Wallet)>,
      pub(crate) plan: Option<build::ClaimPlan>,
      pub(crate) client: Option<ElectrumClient>,           // claim-only isolated session (set by the engine)
      pub(crate) own_destinations: Vec<Address> }          // fresh Wren addresses, reused across rebuilds
  ```
  - `scan` puts unconfirmed coins into `not_claimable` (`Unconfirmed`) instead of `unconfirmed_ignored`.
  - The Area B and Area D predicates add `Unverified` and `Immature` entries, and those coins are not counted in `total`.
  - `scan` keeps taking `&ElectrumClient`. The engine passes the isolated client (Task 20).
- **Interfaces changed:** `FoundSummary` fields; the `NotClaimed` and `NotClaimedReason` types; `ClaimSession.client` and `.own_destinations`.

## Task 18

### Area B — server trust in everyday use

- **Findings:** H4 (claim cap), M8 + ca-ruz #7 (claim maturity), Low "Uneconomic claim chunks", Low "Error strings"
- **Supersedes:**
  - Interfaces L4894-4895 `pub fn build(&mut self, destination: &Address, fee_rate: FeeRate)` and `build_with_limit(...)`
  - L4892 `pub struct ClaimPlan { ...`
  - L5081 `fn werr`
  - L5131-5133 `if fee_rate < FeeRate::from_sat_per_vb_unchecked(1) {`
  - L5136-5141, from `let utxos: Vec<OutPoint> = wallet.list_unspent()` through `let psbt = build_one(wallet, *ty, chunk, destination, fee_rate)?;`
  - Test calls `s.build(&dest(), ..)` and `s.build_with_limit(&dest(), ..)` (they gain `&params, tip`)
- **New failing tests first:**
  - `uneconomic_coins_are_skipped_and_reported`: a fixture variant with per-coin amounts, P2WPKH 70,000 and P2PKH 500 at 5 sat/vB. Expect one tx, and `plan.skipped == [SkippedCoin { P2pkh, 500 sats, Uneconomic }]`.
  - `dust_chunk_does_not_abort_other_types`: a type whose only chunk falls below dust after fees is skipped; the others still build.
  - `claim_skips_relay_immature_coinbase`: a fixture coinbase that is immature under `coinbase_relay_maturity`. It is not an input, and is skipped with `Immature`.
  - `claim_refuses_rate_above_cap`: 1,001 sat/vB returns `Err(Input("fee rate is too high"))`.
- **Changes:**
  - Signatures:
    - `build(&mut self, params: &ChainParams, tip: u32, destination: &Address, fee_rate: FeeRate)`
    - `build_with_limit(.., max_inputs)`
  - The first statement is `crate::payment::check_fee_rate(fee_rate)?;`.
  - `werr` becomes `crate::payment::map_create_tx` in `sweep` and `build_one`; `psbt.fee()` errors become `Claim("could not compute the fee")`.
  - New types:
    ```rust
    #[derive(Clone, Debug, PartialEq, Eq)] pub enum SkipReason { Immature, Uneconomic }
    #[derive(Clone, Debug, PartialEq, Eq)] pub struct SkippedCoin { pub script_type: ClaimScriptType, pub amount: Amount, pub reason: SkipReason }
    // ClaimPlan gains: pub skipped: Vec<SkippedCoin>, pub abnormal_fee: bool
    fn input_fee(wallet: &Wallet, ty: ClaimScriptType, rate: FeeRate) -> Result<Amount, EngineError> {
        let sat_w = wallet.public_descriptor(KeychainKind::External).max_weight_to_satisfy()
            .map_err(|_| EngineError::Claim("could not size this coin type"))?;
        let extra = if ty == ClaimScriptType::P2trKeyPath { Weight::from_wu(1) } else { Weight::ZERO }; // 65-byte UNIFIED sig
        Ok(rate * (Weight::from_wu(164) + sat_w + extra)) // 164 wu = outpoint + sequence + script length byte
    }
    // inside the per-type loop:
    let immature: HashSet<OutPoint> = spendable::immature_coinbase(wallet, params, tip).into_iter().collect();
    let fee_in = input_fee(wallet, *ty, fee_rate)?;
    let mut utxos = Vec::new();
    for u in wallet.list_unspent().filter(|u| u.chain_position.is_confirmed()) {
        let reason = if immature.contains(&u.outpoint) { Some(SkipReason::Immature) }
                     else if u.txout.value <= fee_in { Some(SkipReason::Uneconomic) } else { None };
        match reason { Some(r) => skipped.push(SkippedCoin { script_type: *ty, amount: u.txout.value, reason: r }), None => utxos.push((u.outpoint, u.txout.value)) }
    }
    for chunk in utxos.chunks(max_inputs) {
        let ops: Vec<OutPoint> = chunk.iter().map(|c| c.0).collect();
        match build_one(wallet, *ty, &ops, destination, fee_rate) {
            Ok(psbt) => { /* check_prevouts, fee, amount, qr_frames, push ClaimTx (unchanged) */ }
            Err(EngineError::InsufficientFunds { .. }) => skipped.extend(chunk.iter().map(|c| SkippedCoin { script_type: *ty, amount: c.1, reason: SkipReason::Uneconomic })),
            Err(e) => return Err(e),
        }
    }
    ```
  - If `txs` is empty and `skipped` is not: return `Err(Claim("these coins are worth less than the fee to move them"))`.
  - `abnormal_fee`: `total_fee * 100 >= total_out * 10 || fee_rate > ABNORMAL_RATE`.
- **Interfaces changed:**
  - new `build` / `build_with_limit` signatures;
  - `ClaimPlan.{skipped, abnormal_fee}`;
  - `SkippedCoin` and `SkipReason`.
  - Overlap with Area D/E (per-transaction destinations, scoped success copy): their unconfirmed and unverified skip reasons can extend `SkipReason`. The filters here run before chunking, so a per-transaction destination list still works.

### Area E — Claim, links, QR, tests, CI

- **Findings:**
  - chrisguida M5 (fresh destination per transaction) and M14 (PSBT size);
  - Low "uneconomic claim chunks" (decided: skip and report);
  - ca-ruz #7 and chrisguida H4 hooks (Areas D and C).
- **Supersedes:**
  - `pub fn build(&mut self, destination: &Address, fee_rate: FeeRate) -> Result<&ClaimPlan, EngineError>;` (L4894, L5126).
  - `pub(crate) fn build_with_limit(` (L4895, L5130).
  - `pub struct ClaimPlan { pub destination: String,` (L4892, L5073-5079).
  - `pub struct ClaimTx {` (L4890, L5063-5070).
  - `for chunk in utxos.chunks(max_inputs) {` (L5140-5147). It aborts the whole claim via `?` on an uneconomic chunk.
  - Test `assert_eq!(t.psbt.unsigned_tx.output[0].script_pubkey, dest().script_pubkey());` (L4998).
  - Every `s.build(&dest(), …)` call site in Tasks 18 and 19.
- **New failing tests first:**
  - `own_wallet_destinations_are_fresh_per_tx_and_reused_on_rebuild`: with `ClaimDestination::OwnWallet` and 2 types, the 2 transactions pay distinct addresses. Rebuilding at a new fee reuses the same 2 addresses (the closure is called twice in total).
  - `elsewhere_uses_the_one_address_for_every_tx`.
  - `splits_by_psbt_size`: with 3 coins and `max_bytes` set to one single-input PSBT size + 1, the result is 3 transactions, each `serialize().len() <= max_bytes`.
  - `single_oversized_coin_is_reported_not_claimed`: `max_bytes = 10` puts every coin in `not_claimed` as `TooLargeForQr`, then `Err(Claim("these coins can't be signed over QR"))`.
  - `uneconomic_coins_are_skipped_and_reported`: with coins `[70_000, 100]` at 2 sat/vB, there is 1 transaction with 1 input, and `not_claimed == [(P2wpkh, Uneconomic, 1, 100 sat)]`.
  - `everything_uneconomic_is_an_error`: coins `[100]` give `Err(Claim("these coins are worth less than the fee to move them"))`.
- **Changes:**
  ```rust
  pub const MAX_CLAIM_PSBT_BYTES: usize = 50_000; // placeholder; set from the Task 22a SeedSigner measurement
  pub enum ClaimDestination { OwnWallet, Elsewhere(Address) }
  pub struct ClaimTx { pub script_type: ClaimScriptType, pub destination: Address, pub psbt: Psbt, pub inputs: usize,
      pub amount: Amount, pub fee: Amount, pub qr_frames: Vec<String> }
  pub struct ClaimPlan { pub txs: Vec<ClaimTx>, pub total_in: Amount, pub total_fee: Amount, pub total_out: Amount,
      pub not_claimed: Vec<NotClaimed> }   // scan's not_claimable + build's skips
  impl ClaimSession {
      pub fn build(&mut self, dest: ClaimDestination, next_own: &mut dyn FnMut() -> Result<Address, EngineError>,
                   fee_rate: FeeRate) -> Result<&ClaimPlan, EngineError>;   // = build_with_limits(.., MAX_INPUTS_PER_TX, MAX_CLAIM_PSBT_BYTES)
      pub(crate) fn build_with_limits(&mut self, dest: ClaimDestination, next_own: …, fee_rate: FeeRate,
                   max_inputs: usize, max_bytes: usize) -> Result<&ClaimPlan, EngineError>;
  }
  ```
  - **Destination.** Transaction `n` uses `Elsewhere(a)` → `a`. With `OwnWallet` it uses `self.own_destinations[n]`, calling `next_own()` and pushing the result only when `n >= len`.
  - **Uneconomic coins.** Before chunking, drop every coin whose `value <= ceil(input_weight(ty) * rate / 4)` into `Uneconomic`. The weights are P2WPKH 272, P2SH-P2WPKH 364, P2PKH 592 and P2TR 231 WU. After building, if BDK's `finish()` returns `CreateTxError::OutputBelowDustLimit` or `InsufficientFunds`, put that chunk's coins in `Uneconomic` and continue. To match those errors, `sweep` returns `Result<Psbt, CreateTxError>` and maps them in the caller.
  - **Size split.** Per chunk of at most `max_inputs` coins, call `build_sized(chunk)`:
    - if `psbt.serialize().len() <= max_bytes`, push it;
    - else if `chunk.len() > 1`, split the chunk in half and recurse on each half;
    - else the coin is `TooLargeForQr`.
  - If `txs` is empty and `not_claimed` is not, return a named `Claim` error. The wording depends on the reasons: all uneconomic gives "worth less than the fee"; otherwise "can't be signed over QR".
  - Apply Area D's `MAX_FEE_RATE` check next to the existing floor check.
  - **Deferred, one line:** `.only_witness_utxo()` for non-P2PKH types, only after the firmware maintainer confirms the SeedSigner fork accepts it.
  - **Test helpers:** add `claim_fixture_with(types, amounts: &[u64])`; `claim_fixture` delegates to it. Tests use `s.build(ClaimDestination::Elsewhere(dest()), &mut || unreachable!(), rate)`.
  - Step 5 now expects "PASS (10 tests)".
- **Interfaces changed:** `ClaimDestination`; the `build` / `build_with_limits` signatures; `ClaimTx.destination`; `ClaimPlan.not_claimed` (the single `destination` field is removed); `MAX_CLAIM_PSBT_BYTES`.

## Task 19

### Area B — server trust in everyday use

- **Findings:** M10 (claim retry), M11
- **Supersedes:** L5604 `pub async fn finish(&mut self, signed: &[ScannedPayload], client: &ElectrumClient) -> Result<Vec<Txid>, EngineError> {` through L5623 (the broadcast loop and the `format!("{n} of {} claim transactions were sent; ...")` line).
- **New failing tests first:**
  - `claim_finish_retry_skips_sent_and_reports_per_tx_status`: two txs, `Reject("txn-mempool-conflict")` on the second. Expect `[Sent, Rejected{Conflict}]`. Retry under `Accept`: `world.broadcasts` gains only the second tx, and the result is `[Sent, Sent]`.
  - `claim_finish_treats_known_tx_as_sent`: `RecordThenFail` on tx 0. Expect `Sent` via `transaction_known`.
- **Changes:**
  - `ClaimSession` gains `pub(crate) sent: Vec<bool>`. It is reset whenever `plan` is rebuilt.
  - `finish` returns `Result<Vec<BroadcastOutcome>, EngineError>`:
    - verify every tx before broadcasting any (unchanged);
    - skip `sent[i]`, which reports `Sent(txid)`;
    - broadcast all the others; the txs are independent, so continue after a failure;
    - map each result as Task 14 does;
    - for anything other than `Sent`, if `client.transaction_known(&txid).await? == true`, report `Sent`;
    - set `sent[i]` for every `Sent`.
  - No server text in any message.
- **Interfaces changed:** `ClaimSession::finish -> Result<Vec<BroadcastOutcome>, _>`.

### Area E — Claim, links, QR, tests, CI

- **Findings:** chrisguida M3 (both variants); ca-ruz test defect (dropped input); ca-ruz extra refusal cases
- **Supersedes:**
  - `pub const WRONG_SIGNATURE_MESSAGE: &str = "Your signer made a normal signature, which could also move your BTC. Wren blocked it. …";` (L5511). Removed.
  - `if sig.len() != 65 || sig[64] as u32 != SIGHASH_ALL_UNIFIED {` (L5549-5551) and `if sig.last().map(|b| *b as u32) != Some(SIGHASH_ALL_UNIFIED) {` (L5563-5565).
  - `pub async fn finish(&mut self, signed: &[ScannedPayload], client: &ElectrumClient)` (L5197, L5604).
  - The test signer `/// Signs like the fork firmware. \`type_byte\` … 0x01 = legacy firmware` (L5387-5420), which appends 0x01 to a UNIFIED digest.
  - `let mut input_dropped = base.clone();` / `input_dropped.output.clear();` (L5464-5465).
  - `normal_signature_is_blocked_with_the_firmware_message` (L5442-5451).
- **New failing tests first:**
  - `standard_bip143_signature_is_dangerous`: the P2WPKH signature comes from rust-bitcoin `p2wpkh_signature_hash(.., EcdsaSighashType::All)` with byte 0x01 → `Err(DangerousSignature)`, zero broadcasts.
  - `standard_legacy_p2pkh_signature_is_dangerous`.
  - `standard_taproot_64_byte_is_dangerous`: a BIP341 SIGHASH_DEFAULT 64-byte signature.
  - `raw_0x21_bip143_signature_is_dangerous`: the digest is `bip143_sighash_raw(.., 0x21)` and the byte is 0x21.
  - `raw_0x21_legacy_p2pkh_signature_is_dangerous`.
  - `raw_0x21_taproot_bip341_is_invalid_not_dangerous`: the result is `Claim("a signature does not verify; nothing was sent")`.
  - `dangerous_wins_over_other_errors`: in a 2-transaction plan, tx0 has an invalid signature and tx1 a standard one → `DangerousSignature`.
  - `changed_transaction_is_refused_for_each_kind_of_change`: rewritten with 2 coins and exact messages (below).
  - `wrong_key_signature_does_not_verify`, `malformed_der_does_not_verify`, `sighash_0x22_is_dangerous`.
  - `finalized_psbt_signatures_are_classified`: the signatures sit only in `0x07`/`0x08`. UNIFIED ones are accepted; a standard one gives `DangerousSignature`.
- **Changes:**
  - **Reader.** `read_signed_psbt` also extracts signatures from final fields:
    - `0x08` (witness): for P2WPKH and P2SH-P2WPKH, item 0 is the signature and item 1 the pubkey; for P2TR, item 0.
    - `0x07` (scriptSig): for P2PKH, the first push.
    - These go into the same `SignedInput` fields, so finalized PSBTs are classified, not ignored.
  - **Classifier** (`verify.rs`):
    ```rust
    pub(crate) enum SigClass { Unified, Dangerous, Invalid }
    fn classify_ecdsa(tx: &Transaction, i: usize, spent: &[TxOut], ty: ClaimScriptType, pk: &PublicKey, sig: &[u8]) -> SigClass {
        let Some((&byte, der)) = sig.split_last() else { return SigClass::Invalid };
        if byte as u32 != SIGHASH_ALL_UNIFIED { return SigClass::Dangerous; }          // any other type: assume BTC-valid
        if verify_ecdsa_unified(/* as today */).is_ok() { return SigClass::Unified; }
        let Ok(mut s) = ecdsa::Signature::from_der(der) else { return SigClass::Invalid };
        s.normalize_s();                                                                // BTC consensus allows high-S
        let d = match ty {
            ClaimScriptType::P2pkh => legacy_sighash_raw(tx, i, &spent[i].script_pubkey, 0x21),
            _ => bip143_sighash_raw(tx, i, &p2wpkh_script_code(pk), spent[i].value, 0x21),
        };
        match d { Some(d) if secp.verify_ecdsa(&Message::from_digest(d), &s, pk).is_ok() => SigClass::Dangerous, _ => SigClass::Invalid }
    }
    // Schnorr: len 64 → Dangerous; len 65 && byte != 0x21 → Dangerous; 0x21 and UNIFIED fails → Invalid.
    ```
  - **`finish(&mut self, signed: Vec<ScannedPayload>)`.**
    - It uses `self.client` (the isolated session) and takes no client parameter.
    - It classifies every input of every transaction before any broadcast.
    - If any input is `Dangerous`, it zeroizes every payload and every decoded PSBT byte, then returns `Err(EngineError::DangerousSignature)`. The session stays open so the user can re-sign.
    - Otherwise it returns the first other error as today.
    - Add `impl Zeroize for ScannedPayload`, zeroing the `String` or `Vec<u8>`.
  - **Tests.**
    - Replace `sign()` with three signers: `sign_unified(t)`, `sign_standard(t)` (real rust-bitcoin BTC digests: `legacy_signature_hash`, `p2wpkh_signature_hash`, or `taproot_key_spend_signature_hash` with `TapSighashType::Default`), and `sign_raw_0x21(t)` (the Task 7 raw digests).
    - Add `sign_as(t, &psbt_tx)`. It signs Wren's transaction, then writes the minimal PSBT for `psbt_tx` with signatures truncated or padded to `psbt_tx.input.len()`.
    - The change test uses `plan_n(&[P2wpkh], 2)` and `input_dropped.input.remove(1)`, with exact messages:
      - `out_changed` and `fee_changed` → "the signed transaction pays a different destination, amount, or fee";
      - `input_added` and `input_dropped` → "the signed transaction spends different coins than Wren found".
    - Every refusal test asserts `broadcasts.is_empty()`.
  - Step 7 now expects "PASS (all claim tests, including 15 verification tests)".
- **Interfaces changed:**
  - `WRONG_SIGNATURE_MESSAGE` is removed.
  - `finish(&mut self, signed: Vec<ScannedPayload>) -> Result<Vec<Txid>, EngineError>`.
  - `EngineError::DangerousSignature` is a new outcome.
  - `ScannedPayload: Zeroize`.

## Task 20

### Area A — headers, sync, chain selection

- **Findings:**
  - ca-ruz #3
  - chrisguida M7 (stale tip)
  - chrisguida Low "Header store" (silent reset)
- **Supersedes:**
  - L5655 `pub struct ConnectionStatus { pub connected: bool, pub verified: bool, pub tip_height: Option<u32>, pub protocol: Option<String>, pub tor: bool }`
  - L5861 `_ => HeaderChain::new(cfg.params.clone()), // missing or corrupt: re-verify from the anchor`
  - L5913 `sync_headers(&client, &mut st.chain).await?;`
  - L5941 `if let Err(e) = sync_headers(client, &mut st.chain).await {`
  - L5947 `sync_wallet(client, &st.chain, w.bdk_mut(), now()).await?;` (the report is now returned)
  - `fn status_of(`
- **New failing tests first:**
  - `verified_requires_activation`: connecting to `TestChain::new(100,4,105,1)` gives `Err`, and `!connection_status().verified`.
  - `status_reports_tip_time_and_stale_flag`: after a verified connect, `tip_time == Some(last header time)`. Because `TestChain` times are in 2023, `stale_tip == true`.
  - `corrupt_header_store_is_set_aside`: write garbage to `headers.bin`. `WrenEngine::new` returns `Ok`, `headers.bin.corrupt` exists, and the chain starts empty.
- **Changes:**
  - `pub const STALE_TIP_SECS: u64 = 2 * 60 * 60;`
  - `ConnectionStatus` gains `pub tip_time: Option<u64>, pub stale_tip: bool`. `stale_tip = now() - tip_time > STALE_TIP_SECS`.
  - In `new`:
    ```rust
    let path = cfg.data_dir.join("headers.bin");
    let chain = match load_chain(cfg.params.clone(), &path) {
        Ok(Some(c)) => c,
        Ok(None) => HeaderChain::new(cfg.params.clone()),
        Err(_) => {
            let _ = std::fs::rename(&path, path.with_extension("bin.corrupt"));
            HeaderChain::new(cfg.params.clone())
        }
    };
    ```
  - Wallet checkpoints for `known`:
    ```rust
    fn known_blocks(st: &State) -> Vec<(u32, [u8; 32])> {
        st.wallet.as_ref()
            .map(|w| w.bdk().local_chain().iter_checkpoints().map(|cp| (cp.height(), cp.hash().to_byte_array())).collect())
            .unwrap_or_default()
    }
    ```
  - Both `sync_headers` calls become `sync_headers(client, &mut st.chain, &known_blocks(st), now())`.
  - `sync()` keeps the `WalletSyncReport` and hands it to the restore and pre-split policy (other area).
  - `verified` needs no extra code: `sync_headers` now errors unless activation and minimum work are reached.
- **Interfaces changed:**
  - `ConnectionStatus.{tip_time, stale_tip}`
  - `pub const STALE_TIP_SECS`
  - No second-server field (decision).

### Area B — server trust in everyday use

- **Findings:** M6, M8, M10, M15, H4, Low "Error strings" (wiring)
- **Supersedes:**
  - Interfaces L5670 `pub async fn balance(&self) -> Result<Balance, EngineError>;`, L5675 `sign_and_broadcast(...) -> Result<Txid, ...>`, L5683 `claim_finish(...) -> Result<Vec<Txid>, ...>`
  - L5858 `.map_err(|e| EngineError::Wallet(e.to_string()))?;`
  - Every `EngineError::Wallet("…".into())` (L5954, L5987, L5998, L6008, L6015)
  - L5957-5959 `balance`, L5991-6000 `sign_and_broadcast`, L6002-6010 `bump_fee`
  - L6024-6027 `make_payment_link`
  - L6068-6075 `claim_build` (destination and `build` call)
  - L6097-6099 `let txids = session.finish(&signed, client).await?;` / `st.claim = None;`
- **New failing tests first:**
  - `claim_session_survives_partial_finish`: one tx rejected. `claim_active()` stays true; after a successful retry it is false.
  - `claim_build_reuses_destination_across_rebuilds`: two `claim_build(id, None, ..)` calls give the same destination(s), and the wallet's revealed index does not grow on the second call.
  - `balance_reports_spendable_incoming_immature`: a smoke test of the `WrenBalance` wiring.
- **Changes:**
  - `balance() -> Result<WrenBalance, EngineError>` returns `w.balance(&self.cfg.params)`.
  - `sign_and_broadcast(..) -> Result<BroadcastOutcome, EngineError>`.
  - `bump_fee` passes `&self.cfg.params` and `st.chain.tip_height().ok_or(NotVerified)?`.
  - `make_payment_link` adds `.and_then(crate::uri::require_payable)`.
  - `claim_build`:
    - when `destination` is `None`, use `ClaimSession.default_destination: Option<Address>` (or per-transaction addresses, per Area D/E), filled once from `next_receive_address()` and reused on every rebuild;
    - call `.build(&self.cfg.params, tip, &dest, fee_rate)`;
    - `ClaimReview` gains `skipped: Vec<SkippedCoin>` and `abnormal_fee: bool`.
  - `claim_finish -> Result<Vec<BroadcastOutcome>, _>`. Set `st.claim = None` only when every outcome is `Sent`.
  - All `Wallet(..)` strings become `&'static str`, for example `Wallet("could not create the data folder")`.
- **Interfaces changed:**
  - `balance() -> WrenBalance`;
  - `sign_and_broadcast -> BroadcastOutcome`;
  - `claim_finish -> Vec<BroadcastOutcome>`;
  - `ClaimReview.{skipped, abnormal_fee}`.
  - Overlaps: Area E (claim on a separate Tor session) and Area D (claim destinations, scoped copy) edit the same `claim_*` fns.

### Area C — network, Tor, TLS, Electrum

- **Findings:** ca-ruz #9 (engine part); ca-ruz #6; chrisguida H3 (enforcement); chrisguida Low Tor (ii) lock part and (iv) route; the Claim-requires-Tor decision.
- **Supersedes:**
  - Interfaces: `pub struct EngineConfig { … pub tor: bool }`. It stays, but as the initial value only.
  - `pub struct ConnectionStatus { … pub tor: bool }`.
  - `pub async fn set_server(&self, server: ServerAddress);`.
  - `pub async fn connect(&self)` (both the Task 20 body `if self.cfg.tor { return Err(... "use connect_tor (Task 21)" ...` and `let stream = transport::connect(&server).await?;`).
  - `pub(crate) async fn connect_with_stream(&self, stream: BoxStream)`.
  - `fn status_of(st: &State, tor: bool) -> ConnectionStatus {`.
  - `Self::status_of(&*self.st.lock().await, self.cfg.tor)`.
  - The `claim: Option<(u64, ClaimSession)>,` field.
  - In `claim_scan` and `claim_finish`: `let client = st.client.as_ref().ok_or(EngineError::NotVerified)?;`.
  - Test fixture `server: ServerAddress::Tcp { host: "unused".into(), port: 1 },`, which becomes `"127.0.0.1"`.
  - `e.connect_with_stream(serve(…))`, which becomes `e.connect_with_stream(serve(…), Route::Direct)`.
  - Step 5 "PASS (5 tests)" becomes "PASS (14 tests)".
- **New failing tests first:**
  - `set_server_refuses_disallowed_addresses`: `set_server(Tcp { "electrum.example.com" })` gives `Err(Input)`, and `connection_status` still shows the old server unchanged.
  - `connect_reports_the_route_actually_used`: with an `FnDialer` returning `Route::Direct`, `status.route == Some(Route::Direct)`. With `Route::Tor`, it is `Some(Route::Tor)`. After `disconnect()`, it is `None`.
  - `status_and_cancel_do_not_wait_for_a_stuck_scan` (`start_paused`): the claim session uses `serve_stalling(world, 5)`. While `claim_scan` is spawned, `connection_status()` and `claim_cancel(id)` each finish within 10 ms. The scan returns `Err(Cancelled)`, and the wallet session stays verified.
  - `set_server_aborts_inflight_sync` (`start_paused`): `sync()` on a stalling server, then `set_server(..)` finishes within 10 ms. `sync` returns `Err(Cancelled)`, and status is `connected: false, verified: false`.
  - `poisoned_connection_is_dropped_and_unverified` (`start_paused`): `sync()` against a stalling server returns `Err(Network)` after `REQUEST_DEADLINE`. Status is then `connected: false, verified: false`.
  - `claim_requires_tor_on_mainnet`: with mainnet params, `tor: false` and a restored wallet, `claim_scan` gives `Err(Claim("Claim needs Tor. Turn on Tor in Settings."))` and no dial happens.
  - `claim_scan_uses_its_own_session`: the test dialer serves `Purpose::Wallet` with tag 0 and `Purpose::Claim` with tag 1. After `claim_scan`:
    - every `get_history` for claim scripthashes is on tag 1, and none on tag 0;
    - exactly one claim dial happened;
    - after `claim_cancel`, the log contains `(1, "closed")`.
  - `claim_finish_broadcasts_on_the_claim_session`: the broadcast appears on tag 1 only.
  - `probe_certificate_uses_the_engine_route`: `probe_certificate` dials through the `Purpose::Probe` dialer (recorded) and returns `CertInfo`.
- **Changes:**
  - Remove the field `tor` from `ConnectionStatus` and add `pub route: Option<Route>`. Sub-plan 2 shows "Tor" when `route == Some(Route::Tor)`.
  - Add `#[derive(Clone, Copy, PartialEq, Eq)] pub(crate) enum Purpose { Wallet, Claim, Probe }`.
  - `State` gains `tor: bool` (initialised from `cfg.tor`), `route: Option<Route>`, and `claim: Option<ClaimSlot>`, where `struct ClaimSlot { id: u64, session: ClaimSession, client: Option<Arc<ElectrumClient>> }`.
  - `State.client` becomes `Option<Arc<ElectrumClient>>`.
  - `WrenEngine` gains these fields outside `State`:
```rust
status: std::sync::Mutex<ConnectionStatus>,          // snapshot; connection_status() reads only this
abort_net: tokio::sync::watch::Sender<u64>,          // set_server / set_tor / disconnect
abort_claim: tokio::sync::watch::Sender<u64>,        // claim_cancel
#[cfg(test)] test_dialer: Option<Arc<dyn Fn(Purpose) -> Arc<dyn Dialer> + Send + Sync>>,
```
  - `fn policy(&self) -> RoutePolicy { RoutePolicy { network: self.cfg.params.network } }`.
  - `fn publish(&self, st: &State)` writes the snapshot. Call it after every change to client, verified, tip or route.
  - `new` checks the server with `policy().check(&cfg.server, if cfg.tor { Route::Tor } else { Route::Direct })?`.
  - `set_server(&self, server) -> Result<(), EngineError>`:
    1. Check the policy against the current route setting.
    2. `abort_net.send_modify(|g| *g += 1)`, before locking.
    3. Lock; set the server; `client = None`, `verified = false`, `route = None`; `publish`.
  - `set_tor(&self, on: bool) -> Result<(), EngineError>` works the same way: the current server must pass the policy for the new route, then abort, then drop the connection. `disconnect(&self)` aborts and drops the connection.
  - Cancellation:
```rust
async fn cancellable<T>(&self, abort: &watch::Sender<u64>, f: impl Future<Output = Result<T, EngineError>>) -> Result<T, EngineError> {
    let mut rx = abort.subscribe();
    let start = *rx.borrow_and_update();
    tokio::select! {
        biased;
        _ = rx.wait_for(|g| *g != start) => Err(EngineError::Cancelled), // dropping `f` releases the state lock it holds
        r = f => r,
    }
}
```
    - Wrapped with `abort_net`: `connect` (dial plus `connect_with_stream`), `sync` and `fee_estimates`.
    - Wrapped with both (nested): `claim_scan`.
    - Not cancellable: `sign_and_broadcast` and `claim_finish`, because aborting between a send and its reply is the lost-reply hazard. They are bounded by `REQUEST_DEADLINE`.
    - After any wallet op, if `client.is_poisoned()` or the result is `Cancelled` from `abort_net`, set `client = None`, `verified = false`, `route = None`, then `publish`.
    - `claim_cancel` does `abort_claim.send_modify(..)` first, then locks and drops the `ClaimSlot`, which also drops its client and closes that session.
  - `connect`:
    1. Read `(server, tor)` under a short lock and release it.
    2. `let d = self.dialer(Purpose::Wallet, tor, &server).await?;`.
    3. `connect_via(&server, &*d, &self.policy())`.
    4. `connect_with_stream(stream, d.route())`, which stores the route.
    - In Task 20, `dialer(_, tor = true, _)` returns `Err(Network("this build does not include Tor"))`. Task 21 replaces it.
  - The dialer and claim-client helpers:
```rust
async fn dialer(&self, purpose: Purpose, tor: bool, server: &ServerAddress) -> Result<Arc<dyn Dialer>, EngineError> {
    if purpose == Purpose::Claim && !tor && !self.claim_direct_allowed(server) {
        return Err(EngineError::Claim("Claim needs Tor. Turn on Tor in Settings."));
    }
    #[cfg(test)] if let Some(f) = &self.test_dialer { return Ok(f(purpose)); }
    if tor { return self.tor_dialer().await; } // Task 21
    Ok(Arc::new(DirectDialer))
}
fn claim_direct_allowed(&self, s: &ServerAddress) -> bool { // tests only: regtest + loopback
    self.cfg.params.network == Network::Regtest
        && matches!(s, ServerAddress::Tcp { .. }) && self.policy().check(s, Route::Direct).is_ok()
}
async fn open_claim_client(&self, server: &ServerAddress, tor: bool, chain: &HeaderChain) -> Result<Arc<ElectrumClient>, EngineError> {
    let d = self.dialer(Purpose::Claim, tor, server).await?;               // fresh Tor isolation token
    let client = ElectrumClient::connect(connect_via(server, &*d, &self.policy()).await?).await?;
    let mut probe = chain.clone();
    sync_headers(&client, &mut probe).await?;                              // same verification as the wallet session
    let tip = chain.tip_height().ok_or(EngineError::NotVerified)?;
    if probe.hash_at(tip) != chain.hash_at(tip) { return Err(EngineError::NotVerified); }
    Ok(Arc::new(client))
}
```
  - `claim_scan` checks the Claim/Tor requirement first. It then opens `ClaimSlot.client` lazily: reuse it unless it is poisoned, and if poisoned reopen it, which gives a new isolation token. It calls `session.scan(&claim_client, &st.chain, …)`.
  - `claim_finish` broadcasts with `session.finish(&signed, &claim_client)`. It never uses `st.client`.
  - Sign-gating still requires the wallet session's `st.verified`.
  - Add `pub async fn probe_certificate(&self, host: &str, port: u16) -> Result<CertInfo, EngineError>`. It uses `dialer(Purpose::Probe, tor, …)` and then `fetch_certificate_fingerprint`. It touches `st` only to read `tor`.
  - `connection_status()` returns `self.status.lock().unwrap().clone()` and never takes `st`.
- **Interfaces changed (sub-plan 2 and other areas depend on these):**
  - `set_server(&self, ServerAddress) -> Result<(), EngineError>`.
  - New: `set_tor(&self, bool) -> Result<(), EngineError>`, `disconnect(&self)`, `probe_certificate(&self, host, port) -> Result<CertInfo, EngineError>`.
  - `ConnectionStatus.route: Option<Route>` replaces `tor`.
  - `EngineError::Cancelled` can come from `connect`, `sync`, `fee_estimates` and `claim_scan`.
  - `EngineError::CertificateChanged` can come from `connect` and `probe_certificate`. The app re-pins only after user consent, through `set_server` with the new SPKI pin.
  - For Area E: claim scan and broadcast run on `ClaimSlot.client`, an isolated, separately verified session. `ClaimSession::scan`/`finish` signatures (Tasks 17 and 19) are unchanged; they already take `&ElectrumClient`. Claim errors with `Claim("Claim needs Tor. Turn on Tor in Settings.")` when Tor is off, except on regtest with a loopback server.
  - `pub(crate) connect_with_stream(stream, route)`.

### Area D — key custody, engine API, PIN

- **Findings:** H2, M12 (engine lock lifecycle), M13, ca-ruz #5
- **Supersedes:**
  - `pub async fn create_wallet(&self) -> Result<Vec<String>, EngineError>;                 // createWallet -> words (shown once)`
  - `pub async fn restore_wallet(&self, words: &str, passphrase: &str) -> Result<(), EngineError>;`
  - `pub async fn load_wallet(&self, words: &str, passphrase: &str) -> Result<(), EngineError>;` (deleted)
  - `Rules enforced here: nothing that sends (payment or claim) runs unless…` (extend)
  - `- [ ] **Step 1: Add the feature to Cargo.toml**` (moved to 12a)
  - Implementation blocks `pub async fn create_wallet(&self)`, `pub async fn restore_wallet(`, `pub async fn load_wallet(`, `pub async fn sign_and_broadcast(` and `#[cfg(feature = "test-hooks")] pub async fn debug_sign_review(`
  - The test helper `fn engine(tc: &TestChain) -> (tempfile::TempDir, WrenEngine) {`
  - The tests' `e.restore_wallet(ABANDON, "").await.unwrap();`
- **New failing tests first** (`engine.rs`; the helper builds `WrenEngine::with_vault(cfg, TestVault::new())`; `setup_restored(&e, &tc)` = connect a MockWorld, then `restore_wallet` → `restore_check` → `finish_setup`):
  - `new_without_installed_vault_fails`: `WrenEngine::new` gives `Err(EngineError::KeyVault)` (no vault installed in this test binary).
  - `send_without_vault_authorization_broadcasts_nothing`: `answer_next(Cancel)`; `sign_and_broadcast` gives `AuthCancelled`; `world.broadcasts` is empty.
  - `wrong_pin_reports_failure_and_sends_nothing`: `answer_next(Wrong)` gives `WrongPin`; `reports == [false]`; no broadcast.
  - `send_asks_the_vault_with_the_engine_summary`: the recorded purpose is `Send(SendApproval{review_id, recipient, amount, fee, ..})`, matching the review.
  - `review_changed_during_authorization_is_refused`: the review is replaced while authorization is pending; the result is `Input("this payment review changed; review it again")`.
  - `locked_engine_refuses_reads_and_sends`: after `lock()`, `balance`, `next_receive_address`, `build_payment` (on a verified connection) and `claim_start` give `Locked`.
  - `idle_lock_after_five_minutes` (`start_paused`): advance 301 s; `balance` gives `Locked`; `unlock()` restores access.
  - `lock_drops_pending_reviews_claim_and_setup`
  - `created_wallet_cannot_send_until_backup_confirmed`: create → finish_setup → `sign_and_broadcast` gives `BackupNotConfirmed`; then `backup_challenge` plus correct answers → `confirm_backup` is true → sending allowed.
  - `restored_wallet_can_send`
  - `reveal_words_and_change_pin_use_pin_only_purposes`: the recorded purposes are `RevealWords` and `ChangePin`; `reveal_words` equals the ABANDON words.
  - `restore_of_seed_with_pre_split_history_is_refused_and_nothing_is_stored`: history at `split_height - 1`; `restore_check` gives `SeedHoldsBtc`; `vault.sealed` is `None`; `wallet.sqlite` doesn't exist; `finish_setup` gives an error.
  - `finish_setup_requires_restore_check`
  - `pre_split_history_found_later_allows_one_send_all_after_acknowledgement`: a later sync sets the flag. Then:
    - `build_payment(exact)` gives `SeedHoldsBtc`.
    - `build_payment(send_all)` before `acknowledge_btc_seed` gives `SeedHoldsBtc`.
    - After acknowledging, one send-all broadcasts.
    - A second send-all gives `SeedHoldsBtc`; `bump_fee` gives `SeedHoldsBtc`.
  - `earlier_install_seed_is_unlocked_and_wallet_rebuilt`: the sealed blob is present and the data dir is empty. `wallet_presence() == SealedSeedOnly`; `unlock()` rebuilds `wallet.sqlite`; `backup_confirmed` is false.
  - `no_secret_bytes_on_disk`: ABANDON run of finish_setup → build → sign → lock. No file in `data_dir` (`wallet.sqlite`, `-wal`, `-shm`, `headers.bin`) or `vault.sealed` contains the 16-byte entropy, the 64-byte BIP39 seed, the master/account/child secret keys or chain codes, or the ASCII words.
  - Existing tests: `sending_requires_a_verified_server` (call `set_server` after setup to drop verification), `wrong_chain_server_is_refused_and_engine_stays_unverified`, `verified_connection_syncs_and_reports_tip` and `claim_requires_wallet_and_matching_id_and_cancel_drops_session` move to `setup_restored`. `create_returns_12_words_once_and_refuses_second_wallet` becomes create → finish_setup → second `create_wallet` gives an error.
- **Changes:**
  1. Replace the API block (L5663-5665) with:
     ```rust
     #[derive(Clone, Copy, Debug, PartialEq, Eq)] pub enum WalletPresence { None, Ready, SealedSeedOnly }
     #[derive(Clone, Debug)] pub struct RestoreCheck { pub tx_count: usize }
     #[derive(Clone, Debug, Default)] pub struct WalletFlags { pub backup_confirmed: bool, pub btc_history_detected: bool, pub btc_sweep_available: bool }
     pub const IDLE_LOCK: std::time::Duration = std::time::Duration::from_secs(300);
     impl WrenEngine {
         pub fn new(cfg: EngineConfig) -> Result<Self, EngineError>;               // uses install_key_vault(); else KeyVault
         #[cfg(any(test, feature = "test-hooks"))]
         pub fn with_vault(cfg: EngineConfig, vault: Arc<dyn KeyVault>) -> Result<Self, EngineError>;
         pub async fn wallet_presence(&self) -> Result<WalletPresence, EngineError>;
         // setup — words cross the bridge only here and in reveal_words
         pub async fn create_wallet(&self) -> Result<Vec<String>, EngineError>;    // seed held in memory only
         pub async fn restore_wallet(&self, words: String, passphrase: String) -> Result<(), EngineError>; // owned, wrapped in Zeroizing
         pub async fn restore_check(&self) -> Result<RestoreCheck, EngineError>;   // verified connection; in-memory scan; SeedHoldsBtc
         pub async fn finish_setup(&self) -> Result<(), EngineError>;              // enroll -> seal -> store -> SQLite; leaves engine unlocked
         pub async fn abandon_setup(&self);
         pub async fn backup_challenge(&self) -> Result<[u8; 3], EngineError>;     // three distinct word positions 0..12
         pub async fn confirm_backup(&self, answers: Vec<String>) -> Result<bool, EngineError>;
         // session
         pub async fn unlock(&self) -> Result<(), EngineError>;
         pub async fn lock(&self);
         pub async fn is_unlocked(&self) -> bool;
         pub async fn reveal_words(&self) -> Result<Vec<String>, EngineError>;
         pub async fn change_pin(&self) -> Result<(), EngineError>;
         pub async fn wallet_flags(&self) -> Result<WalletFlags, EngineError>;
         pub async fn acknowledge_btc_seed(&self) -> Result<(), EngineError>;
         // existing calls keep their signatures; sign_and_broadcast authorizes itself
         #[cfg(any(test, feature = "test-hooks"))] pub async fn debug_allow_pre_split_history(&self);
         #[cfg(feature = "test-hooks")] pub async fn debug_sign_review(&self, review_id: u64) -> Result<Transaction, EngineError>; // goes through the vault
     }
     ```
  2. State additions:
     - `vault: Arc<dyn KeyVault>` on `WrenEngine`.
     - In `State`: `setup: Option<PendingSetup>`, `last_activity: Option<tokio::time::Instant>`, `challenge: Option<[u8; 3]>`, `allow_pre_split_history: bool`.
     - `struct PendingSetup { seed: SeedSecret, kind: SetupKind, backup_confirmed: bool, checked: bool }` with `enum SetupKind { Created, Restored }`.
     - "Unlocked" means `wallet.is_some()` and `last_activity` is within `IDLE_LOCK`.
     - `fn touch(st: &mut State) -> Result<(), EngineError>` locks and returns `Locked` when idle has expired. Otherwise it refreshes the timer.
  3. Replace "Rules enforced here" (L5688) with:
     > Rules enforced here:
     > - Nothing that sends (payment or claim) runs unless the current server connection reached the tip through the verified header chain (`EngineError::NotVerified`, checked first). A wrong-chain server leaves the engine unverified.
     > - Every wallet read or write (`balance`, `transactions`, `next_receive_address`, `build_payment`, `bump_fee`, `sign_and_broadcast`, all `claim_*`) returns `EngineError::Locked` unless the engine is unlocked. The engine locks itself after `IDLE_LOCK` (5 minutes) without a call; the app also calls `lock()` when it goes to the background.
     > - `sync` always syncs headers, and syncs the wallet only when unlocked.
     > - The engine holds no private key between calls. The seed exists only as a sealed blob in the native key vault. Every signature needs a fresh `KeyVault::authorize`, and the key is dropped before broadcast.
     > - A created wallet cannot send until `confirm_backup` passes (`BackupNotConfirmed`). A restored wallet counts as confirmed.
     > - A restored phrase is checked for pre-split history before anything is stored (`SeedHoldsBtc`).
     > - If pre-split history appears later, only one send-all is allowed after `acknowledge_btc_seed`; `bump_fee` is refused.
     > - Claim requires Wren's own wallet to exist first (spec §5). One claim session at a time; `claim_finish`/`claim_cancel` drop it.
  4. Key method bodies:
     ```rust
     async fn open_seed(&self, purpose: AuthPurpose) -> Result<SeedSecret, EngineError> {
         let (vault, network) = (self.vault.clone(), self.cfg.params.network);
         tokio::task::spawn_blocking(move || {
             let key = vault.authorize(&purpose)?;
             let blob = vault.load_sealed()?.ok_or(EngineError::Wallet("no wallet on this device".into()))?;
             let r = open_sealed(&blob, &key, network);
             vault.report(r.is_ok());
             r
         }).await.map_err(|_| EngineError::Wallet("internal task failed".into()))?
     }

     pub async fn unlock(&self) -> Result<(), EngineError> {
         let seed = self.open_seed(AuthPurpose::Unlock).await?;           // mutex NOT held while the PIN pad is up
         let keys = KeySource::from_seed(&seed, self.cfg.params.network)?;
         drop(seed);
         let mut st = self.st.lock().await;
         let w = if WrenWallet::exists(&self.db()) {
             WrenWallet::open(&self.db(), self.cfg.params.network)?
         } else {
             WrenWallet::create(&self.db(), self.cfg.params.network, &keys)? // wallet from an earlier install; backup_confirmed stays false
         };
         if w.fingerprint() != keys.fingerprint() {
             return Err(EngineError::Wallet("the saved wallet does not match the saved recovery phrase".into()));
         }
         st.wallet = Some(w);
         st.last_activity = Some(tokio::time::Instant::now());
         Ok(())
     }

     pub async fn sign_and_broadcast(&self, review_id: u64) -> Result<Txid, EngineError> {
         let (approval, txid_before) = {
             let mut st = self.st.lock().await;
             if !st.verified { return Err(EngineError::NotVerified); }
             Self::touch(&mut st)?;
             let w = st.wallet.as_ref().ok_or(EngineError::Locked)?;
             if !w.flag(MetaFlag::BackupConfirmed)? { return Err(EngineError::BackupNotConfirmed); }
             Self::btc_seed_gate(w, st.pending.kind(review_id))?;
             let summary = payment::review_summary(w, &st.pending, review_id)?;   // the stored ReviewSummary
             (SendApproval::from(&summary), st.pending.unsigned_txid(review_id))
         };
         let seed = self.open_seed(AuthPurpose::Send(approval)).await?;
         let keys = KeySource::from_seed(&seed, self.cfg.params.network)?;
         drop(seed);
         let mut guard = self.st.lock().await;
         let st = &mut *guard;
         if st.pending.unsigned_txid(review_id) != txid_before || txid_before.is_none() {
             return Err(EngineError::Input("this payment review changed; review it again"));
         }
         let client = st.client.as_ref().ok_or(EngineError::NotVerified)?;
         let w = st.wallet.as_mut().ok_or(EngineError::Locked)?;
         if keys.fingerprint() != w.fingerprint() { return Err(EngineError::Signing("no key in this wallet for an input")); }
         let kind = st.pending.kind(review_id);
         let out = payment::sign_and_broadcast(w, &keys, &mut st.pending, review_id, client, now()).await;
         drop(keys);
         if w.flag(MetaFlag::BtcHistoryDetected)? && kind == Some(ReviewKind::SendAll) && !out_is_definite_rejection(&out) {
             w.set_flag(MetaFlag::BtcSweepUsed, true)?;
         }
         out
     }

     fn btc_seed_gate(w: &WrenWallet, kind: Option<ReviewKind>) -> Result<(), EngineError> {
         if !w.flag(MetaFlag::BtcHistoryDetected)? { return Ok(()); }
         let ok = kind == Some(ReviewKind::SendAll)
             && w.flag(MetaFlag::BtcSweepAcknowledged)? && !w.flag(MetaFlag::BtcSweepUsed)?;
         if ok { Ok(()) } else { Err(EngineError::SeedHoldsBtc) }
     }
     ```
     `SendApproval::from(&ReviewSummary)` maps `review_id`, `recipient`, `amount`, `fee` and `fee_rate`, plus `to_self` and `abnormal_fee` when those fields exist (false until then). `build_payment` calls `btc_seed_gate` with the kind it is about to create; `bump_fee` refuses outright while the flag is set.
  5. Other methods:
     - **`create_wallet`:** refuses if `wallet_presence() != None`. Sets `setup = Created { seed: SeedSecret::generate() }` and returns `seed.words()`, moved out of `Zeroizing` only at the return.
     - **`restore_wallet`:** refuses if `wallet_presence() != None`. Builds `SeedSecret::from_words(Zeroizing::new(words), Zeroizing::new(passphrase))` with `kind = Restored` and `backup_confirmed = true`.
     - **`restore_check`:** requires `verified`.
       - Takes `setup`, builds an in-memory wallet (`Wallet::create(ext, int).network(n).create_wallet_no_persist()`) from `KeySource::descriptors()`, and runs `sync_wallet`.
       - If `report.pre_split_history && !st.allow_pre_split_history`, it leaves `setup = None` (the seed is zeroized on drop) and returns `SeedHoldsBtc`.
       - Otherwise it sets `checked = true`, puts `setup` back, and returns `RestoreCheck { tx_count }`.
     - **`finish_setup`:** requires `setup`, and `checked` when the setup is a restore.
       1. `vault.enroll()` via spawn_blocking.
       2. `seal` → `store_sealed`.
       3. `WrenWallet::create(.., &KeySource::from_seed(..))`. If that fails, call `vault.delete_sealed()` and return the error.
       4. `set_flag(BackupConfirmed, setup.backup_confirmed)`.
       5. Drop `setup`, set `wallet = Some`, and touch the idle timer.
     - **`backup_challenge` / `confirm_backup`:** during setup, compare against `setup.seed.words()`. After setup, use `open_seed(ConfirmBackup)`. On success persist `BackupConfirmed` (or set `setup.backup_confirmed`).
     - **`reveal_words`:** `open_seed(RevealWords)` → `words()`.
     - **`change_pin`:** `open_seed(ChangePin)` → `vault.enroll()` → `seal` → `store_sealed`.
     - **`sync`:** after `sync_wallet`, if `report.pre_split_history && !st.allow_pre_split_history`, call `set_flag(BtcHistoryDetected, true)` (sticky).
     - **`lock`:** sets `wallet`, `setup`, `claim`, `challenge` and `last_activity` to `None`, and `pending` to its default.
     - **`wallet_presence`:** combines `WrenWallet::exists` with `vault.load_sealed().is_some()`. A database with no sealed blob reports `None` (the database is not usable without it).
  6. Run `cargo test -p wren-engine engine::`. Expected: PASS (20 tests). Then run the gate with `core/scripts/check.sh`.
- **Interfaces changed:**
  - Removed `load_wallet`.
  - Changed `restore_wallet` (owned `String`s).
  - Added `with_vault`, `wallet_presence`, `restore_check`, `finish_setup`, `abandon_setup`, `backup_challenge`, `confirm_backup`, `unlock`, `lock`, `is_unlocked`, `reveal_words`, `change_pin`, `wallet_flags`, `acknowledge_btc_seed` and `debug_allow_pre_split_history`.
  - New types `WalletPresence`, `RestoreCheck`, `WalletFlags` and `IDLE_LOCK`.
  - `lib.rs` re-exports them, plus `custody::{install_key_vault, KeyVault, AuthPurpose, SendApproval, VaultError, WrappingKey, PinKdfParams, PIN_KDF_DEFAULT, stretch_pin, pin_point, derive_wrapping_key}`.
  - **Sub-plan 2 UniFFI rule:** export none of `install_key_vault`, `KeyVault`, `WrappingKey`, `stretch_pin`, `pin_point` or `derive_wrapping_key` to TypeScript. These are for the native channel only.

### Area E — Claim, links, QR, tests, CI

- **Findings:** chrisguida M5 (Tor required, isolated session, fresh destinations) and M4 (review carries `not_claimed`); ca-ruz #10 and N1 surface through `claim_start`
- **Supersedes:**
  - `let client = st.client.as_ref().ok_or(EngineError::NotVerified)?;` in `claim_scan` (L6060) and in `claim_finish` (L6092).
  - `None => Self::wallet_mut(st)?.next_receive_address()?,` (L6073).
  - `pub struct ClaimReview { pub claim_id: u64, pub destination: String, …` (L5657, L5827-5833).
  - `pub struct ClaimReviewTx { pub inputs: usize, …` (L5659, L5819-5824).
- **New failing tests first:**
  - `claim_scan_refuses_without_tor_on_mainnet`: a Mainnet engine with `tor: false`, a restored wallet and `claim_start` OK; `claim_scan` gives `Err(Claim("Claim needs Tor. Turn on Tor in Settings."))` before any connection.
  - `claim_uses_its_own_connection`: with Area C's connection-counting test connector, `claim_scan` opens a new connection that is not the sync connection, and `claim_finish` broadcasts on that claim connection.
  - `claim_build_reuses_own_destinations_on_rebuild`: two `claim_build(id, None, …)` calls give the same `ClaimReviewTx.destination` list, and the wallet's revealed index advances only once per transaction.
  - `claim_cancel_and_finish_drop_the_claim_connection`.
- **Changes:**
  ```rust
  pub const CLAIM_NEEDS_TOR: &str = "Claim needs Tor. Turn on Tor in Settings.";
  // claim_scan, first lines:
  if !self.cfg.tor && self.cfg.params.network != Network::Regtest { return Err(EngineError::Claim(CLAIM_NEEDS_TOR)); }
  if !st.verified { return Err(EngineError::NotVerified); }
  let session = Self::session(st, claim_id)?;
  if session.client.is_none() { session.client = Some(self.connect_isolated().await?); } // Area C: fresh Arti isolation token
  session.scan(session.client.as_ref().unwrap(), &st.chain, search_further, now()).await
  // claim_build:
  let dest = match destination {
      Some(s) => ClaimDestination::Elsewhere(parse_payment_link(s, self.cfg.params.network)?.address),
      None => ClaimDestination::OwnWallet,
  };
  let wallet = Self::wallet_mut(st)?;   // split borrows: take session and wallet from st separately
  let plan = session.build(dest, &mut || wallet.next_receive_address(), fee_rate)?.clone();
  // claim_finish: the same Tor check; session.finish(signed).await (it uses its own client)
  ```
  - Regtest is exempt from the Tor check, but still uses a separate connection. The Task 22 suite runs `tor: false`.
  - `next_receive_address` here becomes whatever Area D defines under `MAX_UNUSED_RECEIVE`.
- **Interfaces changed:**
  ```rust
  pub struct ClaimReview { pub claim_id: u64, pub total_in: Amount, pub total_fee: Amount, pub total_out: Amount,
      pub transactions: Vec<ClaimReviewTx>, pub not_claimed: Vec<NotClaimed> }
  pub struct ClaimReviewTx { pub destination: String, pub inputs: usize, pub amount: Amount, pub fee: Amount, pub qr_frames: Vec<String> }
  ```
  `claim_scan` and `claim_finish` can return `Claim(CLAIM_NEEDS_TOR)` and `DangerousSignature`.

## Task 21

### Area C — network, Tor, TLS, Electrum

- **Findings:** chrisguida Low Tor (ii) bootstrap with no deadline and under the mutex, plus the compile bug; (iii) no stream isolation; H3 (Tor path); the Claim isolated-session decision.
- **Supersedes:**
  - The sentence starting "With Tor on (the default the app will use), **every** connection goes through Tor: onion addresses directly, clearnet hosts through an exit." It becomes: "With Tor on, every connection (wallet, claim, certificate probe) goes through Tor. Onion addresses are reached directly. Clearnet servers are reached only over TLS through an exit. Each wallet connection and each claim session gets its own isolation token."
  - Interfaces: `pub async fn bootstrap(state_dir: &Path, cache_dir: &Path) -> Result<Tor, EngineError>;` and `pub async fn stream(&self, host: &str, port: u16) -> Result<BoxStream, EngineError>;`.
  - `client: TorClient<PreferredRuntime>,`. This is a compile error: arti 0.46 returns `Arc<TorClient<_>>`.
  - `let client = TorClient::create_bootstrapped(cfg).await.map_err(net)?;`.
  - All of Step 3 "Generalize the transport", which moved to Task 10.
  - In Step 4: `#[cfg(feature = "tor")] tor: Option<std::sync::Arc<crate::electrum::tor::Tor>>` in `State`, and the `tor_stream` body that does `let mut st = self.st.lock().await; if st.tor.is_none() { … Tor::bootstrap(…).await? …`.
  - Step 5's `"tcp" => ServerAddress::Tcp { host, port },`.
  - Step 7's `WREN_TOR_TEST="<fork node onion>:50001:tcp"`.
- **New failing tests first:**
  - `hung_tor_bootstrap_times_out_and_does_not_block_status` (engine, `start_paused`). Uses a `#[cfg(test)]` Tor factory whose `bootstrap` never finishes. While `connect()` is pending, `connection_status()` and `set_server(..)` each finish within 10 ms. `connect()` returns `Err(Cancelled)` after `set_server`, or `Err(Network)` at `TOR_BOOTSTRAP_DEADLINE` if left alone.
  - `isolated_dialers_have_distinct_tokens` (unit, `--features tor`): two `Tor::isolated()` dialers have unequal `IsolationToken`s, and both report `Route::Tor`.
  - Manual (`tests/tor.rs`, ignored), with `WREN_TOR_TEST` forms `<onion>:<port>:onion`, `<host>:<port>:tls:<spki hex>` and `<host>:<port>:probe`:
    - The probe form prints `display_fingerprint(cert_sha256)` and the SPKI hex.
    - The test also opens two isolated dials to the same target, and both must succeed. Whether they really use different circuits can't be checked offline; leave that as a reviewer check.
- **Changes:**
  - Step 1: `arti-client` features additionally need nothing new (`onion-service-client` is already present).
  - `electrum/tor.rs`:
```rust
use arti_client::{BootstrapBehavior, IsolationToken, StreamPrefs, TorClient, config::BoolOrAuto};
pub const TOR_BOOTSTRAP_DEADLINE: Duration = Duration::from_secs(120);
const TOR_DIAL_DEADLINE: Duration = Duration::from_secs(60);
pub struct Tor { client: Arc<TorClient<PreferredRuntime>> }
impl Tor {
    pub async fn create(state_dir: &Path, cache_dir: &Path) -> Result<Tor, EngineError> {
        let cfg = TorClientConfigBuilder::from_directories(state_dir, cache_dir).build().map_err(net)?;
        let client = TorClient::builder().config(cfg).bootstrap_behavior(BootstrapBehavior::Manual)
            .create_unbootstrapped_async().await.map_err(net)?;
        Ok(Tor { client })
    }
    /// Returns at once if already bootstrapped; a retry after a timeout reuses downloaded directory data.
    pub async fn bootstrap(&self) -> Result<(), EngineError> {
        tokio::time::timeout(TOR_BOOTSTRAP_DEADLINE, self.client.bootstrap()).await
            .map_err(|_| EngineError::Network("Tor did not start within 2 minutes".into()))?.map_err(net)
    }
    pub fn isolated(&self) -> TorDialer { TorDialer { client: self.client.clone(), token: IsolationToken::new() } }
}
pub struct TorDialer { client: Arc<TorClient<PreferredRuntime>>, token: IsolationToken }
impl Dialer for TorDialer {
    fn route(&self) -> Route { Route::Tor }
    fn dial<'a>(&'a self, host: &'a str, port: u16) -> DialFuture<'a> {
        Box::pin(async move {
            let mut prefs = StreamPrefs::new();
            prefs.set_isolation(self.token).connect_to_onion_services(BoolOrAuto::Explicit(true));
            let s = tokio::time::timeout(TOR_DIAL_DEADLINE, self.client.connect_with_prefs((host, port), &prefs))
                .await.map_err(net)?.map_err(net)?;
            Ok(Box::new(s) as BoxStream)
        })
    }
}
```
  - `transport.rs`: `#[cfg(feature = "tor")] pub use super::tor::{Tor, TorDialer};`. The `#[cfg(not(feature = "tor"))] pub enum Tor {}` is deleted.
  - Engine:
    - Add the field `#[cfg(feature = "tor")] tor: tokio::sync::OnceCell<Arc<Tor>>` to `WrenEngine`, not to `State`.
    - Implement `tor_dialer()`:
```rust
#[cfg(feature = "tor")]
async fn tor_dialer(&self) -> Result<Arc<dyn Dialer>, EngineError> {
    let tor = self.tor.get_or_try_init(|| async {
        Tor::create(&self.cfg.data_dir.join("tor-state"), &self.cfg.data_dir.join("tor-cache")).await.map(Arc::new)
    }).await?;
    tor.bootstrap().await?;                    // no state lock held; caller wraps in cancellable(abort_net)
    Ok(Arc::new(tor.isolated()))               // fresh isolation token per wallet connection / claim session / probe
}
#[cfg(not(feature = "tor"))]
async fn tor_dialer(&self) -> Result<Arc<dyn Dialer>, EngineError> { Err(EngineError::Network("this build does not include Tor".into())) }
```
    - Optional: expose Arti's `bootstrap_status()` as `ConnectionStatus.tor_progress: Option<u8>` for the spec's "Tor status visible on Home".
- **Interfaces changed:**
  - `Tor::create(state_dir, cache_dir)`, `Tor::bootstrap(&self)`, `Tor::isolated() -> TorDialer`, `TorDialer: Dialer`, `TOR_BOOTSTRAP_DEADLINE`.
  - `Tor::stream` is removed.

## Task 22

### Area A — headers, sync, chain selection

- **Findings:** ca-ruz #3 (a pre-activation legacy server on regtest)
- **Supersedes:** nothing is removed. Insert the block below immediately **before** L6669 `core(None, "generatetoaddress", json!([30, ca])).unwrap(); // legacy chain grows past 200 with v1 headers`.
- **New failing tests first:** the block itself:
  ```rust
  // Core is still at 199 (shared history only): a legacy server below activation must never verify.
  let (_d1, early) = engine("WREN_FULCRUM_CORE");
  early.restore_wallet(WREN_WORDS, "").await.unwrap();
  let mut refused = false;
  for _ in 0..60 {   // Network errors while Fulcrum-on-Core starts; Chain(_) once it answers with tip 199
      if let Err(EngineError::Chain(_)) = early.connect().await { refused = true; break; }
      tokio::time::sleep(std::time::Duration::from_millis(500)).await;
  }
  assert!(refused && !early.connection_status().await.verified, "pre-activation legacy server was not refused");
  ```
- **Changes:**
  - None beyond the block above.
  - Note in the task text: `ChainParams::regtest(SPLIT)` now enforces `fixed_bits 0x207fffff`, version ≥ 4, MTP and the future bound. Knots regtest blocks satisfy all of them.
- **Interfaces changed:** none.

---

### Area C — network, Tor, TLS, Electrum

- **Findings:** chrisguida H3 (compatibility check only).
- **Changes:**
  - No code change. `server()` keeps building `ServerAddress::Tcp`, and `WREN_FULCRUM` and `WREN_FULCRUM_CORE` must stay loopback IP literals (`127.0.0.1:…`) so the regtest exception in `RoutePolicy` applies.
  - Add one note: claims in this suite run on their own Electrum session over `DirectDialer`, under the regtest-plus-loopback exception (Task 20 `claim_direct_allowed`). Mainnet never takes that path.

### Area D — key custody, engine API, PIN

- **Findings:** H2, M13 (test conflict)
- **Supersedes:**
  - `let e = WrenEngine::new(EngineConfig { data_dir: dir.path().into(), …` (in `fn engine(var: &str)`)
  - `e.restore_wallet(WREN_WORDS, "").await.unwrap();`
  - `legacy.restore_wallet(WREN_WORDS, "").await.unwrap();`
- **Changes:**
  - `engine(var)` uses `WrenEngine::with_vault(cfg, TestVault::new())` (feature `test-hooks`, already required).
  - **Lifecycle:** after `mine(101)` and before funding, run:
    ```rust
    sync_to_tip(&e).await;
    e.restore_wallet(WREN_WORDS.into(), "".into()).await.unwrap();
    e.restore_check().await.unwrap();
    e.finish_setup().await.unwrap();
    e.debug_allow_pre_split_history().await; // this wallet deliberately holds a pre-split coin for the replay proof
    ```
    If any step returns `Locked` (more than 5 minutes idle), call `e.unlock().await.unwrap()`; the TestVault answers automatically.
  - **Wrong-chain proof:** replace `legacy.restore_wallet(..)` with `legacy.restore_wallet(WREN_WORDS.into(), "".into()).await.unwrap();`. After the refusal loop, add:
    - `assert!(matches!(legacy.restore_check().await, Err(EngineError::NotVerified | EngineError::WrongChain)));`
    - `assert!(legacy.finish_setup().await.is_err());`

    Keep the `build_payment` → `NotVerified` assertion.
  - **Add a BTC-seed check case:** a fresh engine with `OLD_WORDS` (funded before the split), on fork Fulcrum, *without* the override. `restore_check()` gives `SeedHoldsBtc`, and `wallet_presence()` stays `None`.
- **Interfaces changed:** none.

### Area E — Claim, links, QR, tests, CI

- **Findings:**
  - ca-ruz #11 and #12;
  - chrisguida M1, M2 (incl. nLockTime) and M9;
  - ca-ruz "first prove both nodes hold the same unspent funding outpoints".
- **Supersedes:**
  - `"-rpcbind=0.0.0.0", "-rpcallowip=0.0.0.0/0", "-fallbackfee=0.0001", "-testactivationheight=blake2b@200"]` (L6418).
  - `cargo test --features test-hooks --test regtest -- --ignored --test-threads=1 --nocapture` (L6455).
  - `fn with_checksum(desc: &str) -> String {` (L6535-6537).
  - `fn assert_legacy_chain_rejects(tx_hex: &str) {` (L6545-6551).
  - `mine(101);` (L6559).
  - `knots(Some("miner"), "sendtoaddress", json!([wren_addr0, 1.0]));`, the L6579 send, and `sendtoaddress` at L6620.
  - `knots(Some("old"), "importdescriptors", json!([imports]));` (L6582).
  - `if n == 0 {` … `}` (L6644-6655).
  - `assert_legacy_chain_rejects(&hex::encode(serialize(&ours)));` (L6613) and `assert_legacy_chain_rejects(&hex_tx);` (L6660).
  - `core(None, "createwallet", json!(["c"])).ok();` (L6667), which moves to the start.
  - The "Deviations from spec §8" paragraph (L6356).
- **New failing tests first:** the suite itself. These are the new assertions:
  - imports succeeded and the signer can sign;
  - the standard signature from Core is really standard;
  - consensus-level rejection on Core;
  - the positive controls are accepted;
  - the maturity boundary;
  - over-34-byte destination refused by Wren and RDTS active on Knots;
  - `run.sh` fails if 0 tests ran.
- **Changes:**
  1. **Compose (knots):** append `"-rdtsexpiry=4102444800", "-testcoinbasematuritylong=210:210:330"`. That gives `CoinbaseMaturityLong` = 120, starting after the split as on mainnet.
  2. **Helpers** (`regtest.rs`):
     ```rust
     fn with_checksum(desc: &str) -> String {
         let info = knots(None, "getdescriptorinfo", json!([desc]));
         format!("{desc}#{}", info["checksum"].as_str().unwrap())
     }
     fn import_ok(r: Value) { for (i, x) in r.as_array().unwrap().iter().enumerate() { assert_eq!(x["success"], json!(true), "import {i}: {x}"); } }
     /// Knots mempool needs CoinbaseMaturityLong confirmations for EVERY coinbase; fund only from coins that have them.
     fn fund(addr: &str, btc: f64) { knots(Some("miner"), "send", json!([{addr: btc}, null, "unset", null, {"minconf": 120}])); }
     fn core_addr() -> String { core(Some("c"), "getnewaddress", json!([])).unwrap().as_str().unwrap().into() }
     /// Core must be past nLockTime (BDK anti-fee-sniping sets it to the fork tip) and must hold every input.
     fn legacy_ready(tx: &Transaction) {
         let need = tx.lock_time.to_consensus_u32() as u64 + 1;
         let have = core(None, "getblockcount", json!([])).unwrap().as_u64().unwrap();
         if have < need { core(None, "generatetoaddress", json!([need - have, core_addr()])).unwrap(); }
         for i in &tx.input {
             assert!(!core(None, "gettxout", json!([i.previous_output.txid.to_string(), i.previous_output.vout, false])).unwrap().is_null(),
                     "input {} is not live on the legacy chain", i.previous_output);
         }
     }
     const ECDSA_FALSE: &str = "Script evaluated without error but finished with a false/empty top stack element";
     const SCHNORR_HASHTYPE: &str = "Invalid Schnorr signature hash type";
     fn assert_legacy_consensus_rejects(tx: &Transaction, script_err: &str) {
         legacy_ready(tx);
         let err = core(None, "generateblock", json!([core_addr(), [hex::encode(serialize(tx))], false])).unwrap_err();
         let m = err["message"].as_str().unwrap_or("");
         assert!((m.contains("mandatory-script-verify-flag-failed") && !m.contains("non-mandatory"))
                 || m.contains("block-script-verify-flag-failed"), "not a consensus rejection: {m}");
         assert!(m.contains(script_err), "wrong script error: {m}");
     }
     fn assert_legacy_accepts(tx_hex: &str) { // positive control: same inputs, standard signature
         assert!(core(None, "generateblock", json!([core_addr(), [tx_hex], false])).unwrap()["hex"].is_string());
     }
     ```
  3. **Setup.**
     - Create Core wallet `c` first. Use `mine(121)` instead of `mine(101)`, with a comment on the mempool long-maturity policy.
     - Replace every miner `sendtoaddress` with `fund(…)`.
     - Keep `old_addrs`.
     - `import_ok(knots(Some("old"), "importdescriptors", json!([imports])))`.
     - For each address in `old_addrs`, `getaddressinfo` must show `ismine` and `solvable` true, and `getwalletinfo.private_keys_enabled` must be true.
     - Also run `core(None, "createwallet", json!(["old_std", false, true]))` and `import_ok(core(Some("old_std"), "importdescriptors", json!([imports])).unwrap())`.
  4. **Payment replay proof:**
     - `assert_legacy_consensus_rejects(&ours, ECDSA_FALSE)`.
     - Positive control: Core `signrawtransactionwithkey <unsigned> [wif] [prevtx] "ALL"` → `assert_legacy_accepts(hex)`.
     - Assert `ours.lock_time` equals the Knots tip at build time.
  5. **Claim loop, for EVERY type** (replaces the `if n == 0` block):
     ```rust
     let mut p = Psbt::deserialize(&psbt_bytes).unwrap();
     for i in &mut p.inputs { i.sighash_type = None; }
     let std = core(Some("old_std"), "walletprocesspsbt", json!([b64(&p.serialize()), true, "ALL", true, false])).unwrap();
     let dec = core(None, "decodepsbt", json!([std["psbt"]])).unwrap();
     for inp in dec["inputs"].as_array().unwrap() {
         let sig = inp["partial_signatures"].as_object().and_then(|m| m.values().next()).or(inp.get("taproot_key_path_sig"))
             .and_then(Value::as_str).unwrap();
         assert!(sig.ends_with("01") || sig.len() == 128, "Core did not make a standard signature: {sig}");
     }
     assert!(matches!(e.claim_finish(id, vec![ScannedPayload::Text(std["psbt"].as_str().unwrap().into())]).await,
                      Err(EngineError::DangerousSignature)), "standard-signed claim was not refused");
     let fin = core(None, "finalizepsbt", json!([std["psbt"]])).unwrap();
     legacy_ready(&deserialize::<Transaction>(&hex::decode(fin["hex"].as_str().unwrap()).unwrap()).unwrap());
     assert_legacy_accepts(fin["hex"].as_str().unwrap());               // the twin is live on Core; only the sighash differs
     ```
     After the UNIFIED claim is broadcast, call `assert_legacy_consensus_rejects(&claim_tx, if kind == "tr" { SCHNORR_HASHTYPE } else { ECDSA_FALSE })`. Both calls use `submit=false`, so Core's chain is unchanged.
  6. **Long-maturity boundary (M9)**, after the claims:
     - `if height() < 210 { mine(210 - height()) }`.
     - `knots(None,"generatetoaddress",[1, wren_addr])`, giving `cb = height()`.
     - `mine(118)`, then sync: `balance().immature > 0`.
     - `mine(1)`, then sync: `immature == 0`, and a send-all `sign_and_broadcast` returns `Ok` and is in the Knots mempool.
  7. **Over-34-byte destination (M9):**
     - `let v2 = Address::from_witness_program(WitnessProgram::new(WitnessVersion::V2, &[7u8; 40]).unwrap(), KnownHrp::Regtest)`.
     - `e.build_payment(&v2.to_string(), Some(1000 sat), false, 1 sat/vB)` must be `Err(EngineError::Input(_))`.
     - Oracle: Knots `createrawtransaction`, then `fundrawtransaction`, then `signrawtransactionwithwallet` paying `v2`, then `generateblock … false` errors containing `bad-txns-vout-script-toolarge`. This proves RDTS consensus is on.
  8. **`run.sh`:**
     ```bash
     trap 'docker compose logs --no-color > integration.log 2>&1 || true; docker compose down -v' EXIT
     cargo test --features test-hooks --test regtest -- --ignored --exact fork_regtest_suite --test-threads=1 --nocapture 2>&1 | tee regtest.out
     grep -q "test result: ok. 1 passed" regtest.out || { echo "integration suite did not run"; exit 1; }
     ```
  9. **Deviations paragraph** (replaces L6356):

     **Deviations from spec §8, recorded for Gate B:**
     - (a) The byte-exact cross-sign reference is **Knots**, the consensus implementation, not Shrike. Shrike is exercised in the real-chain trial.
     - (b) The fork-firmware signer in this suite is **Knots `walletprocesspsbt "ALL|UNIFIED"`**. Real SeedSigner firmware code runs headless in Task 22a, and the device or emulator runs at Gate B.
     - (c) The standard ("no fork firmware") signer and the legacy-chain oracle are **Bitcoin Core 29.1**, not "stock main-chain Knots". Knots signs UNIFIED whenever the fork is scheduled (`sign.cpp:70`, `psbt.cpp:455-457`) unless `-walletoldsigs=1`, so it cannot stand in for a standard signer.
     - (d) The replay proof uses `generateblock` (consensus) instead of `testmempoolaccept` (policy). It applies to transactions with pre-split inputs: the payment and all four claims. A post-split input would be rejected as missing, which proves nothing.
  10. **New Step 6: "CI workflow".** The old Step 6 becomes Step 7, and Gate B records the CI run URL. Create `.github/workflows/engine.yml`:
      ```yaml
      name: engine
      on:
        pull_request: { branches: [main] }
        push: { branches: [main] }
      permissions: { contents: read }
      jobs:
        check:
          runs-on: ubuntu-24.04-arm
          steps:
            - uses: actions/checkout@<pinned-sha>   # pin by full commit SHA
            - run: rustup show                       # honours core/rust-toolchain.toml
            - run: cargo install cargo-audit --locked --version <pinned>
            - run: core/scripts/check.sh
        integration:
          runs-on: ubuntu-24.04-arm                  # Dockerfiles fetch aarch64/arm64 release binaries
          timeout-minutes: 90
          steps:
            - uses: actions/checkout@<pinned-sha>
            - run: rustup show
            - run: core/integration/run.sh           # includes the Task 22a headless SeedSigner signer
            - if: failure()
              uses: actions/upload-artifact@<pinned-sha>
              with: { name: integration-logs, path: "core/integration/integration.log\ncore/wren-engine/regtest.out" }
      ```
      Owner action, outside the repo: mark `check` and `integration` as required status checks on `main`. Commit with `git add .github/workflows/engine.yml`.
- **Interfaces changed:** none in the engine. The suite depends on `EngineError::DangerousSignature`, `WrenBalance.immature` (Area D) and the Area D destination refusal.

## Task 22a — SeedSigner fixtures (NEW, after Task 22)

### Area E — Claim, links, QR, tests, CI

**Files:**
- Create: `core/wren-engine/tests/fixtures/seedsigner/SOURCES.md`, plus the `*.urca.txt`, `*.spl.txt`, `*.unsigned.psbt`, `*.signed.frames.txt` and `standard.p2wpkh.signed.frames.txt` fixtures.
- Create: `core/wren-engine/tests/seedsigner_fixtures.rs`, `core/integration/seedsigner_sign.py`.
- Modify: `core/integration/run.sh` (headless signer setup) and `core/wren-engine/tests/regtest.rs` (sign with the headless signer as well).

**Interfaces:**
- Consumes: `import_accounts`, `QrCollector`, `psbt_from_payload` (Task 16); `read_signed_psbt` and `verify_and_finalize` (Task 19); a test-hooks `ClaimTx::from_fixture(psbt: Psbt, ty: ClaimScriptType) -> ClaimTx`.
- Produces: fixture-backed tests on every build, and a headless signer that runs the pinned firmware's own PSBT parser and signer in the `integration` job.

- [ ] **Step 0: Confirm feasibility with the firmware maintainer.**
  - Check that `models/psbt_parser.py` and the embit signing path of `privkeyio/seedsigner` (branch `unified-sighash`, the commit tagged `v0.8.7-unified-sighash.6`) run outside the GUI. The research cites `tests/unified/`, which already drives them against Knots regtest.
  - Get the SeedSigner UR fragment size, so `MAX_UR_FRAGMENTS` can be set.
  - Get the largest PSBT a Pi Zero scans comfortably, so `MAX_CLAIM_PSBT_BYTES` can be set.
  - Ask whether a segwit input without `non_witness_utxo` is accepted (the `only_witness_utxo` question).
  - Record the answers in `SOURCES.md`.
  - If headless isn't feasible, keep Steps 1-3, drop Step 4, and record that the fixtures alone cover per-build interop until Gate B.
- [ ] **Step 1: Write the failing tests** (`tests/seedsigner_fixtures.rs`; they fail because the files are missing):
  - `seedsigner_xpub_exports_import`: for each of the 4 types, the `urca` frames go through `QrCollector` and `import_accounts`, giving the expected type, fingerprint, `{44|49|84|86}'/1'/0'` path and xpub. The Specter `spl` text gives the same account.
  - `seedsigner_signed_frames_reassemble_and_verify`: for each type, the signed frames go through `psbt_from_payload`, `read_signed_psbt`, then `verify_and_finalize(ClaimTx::from_fixture(unsigned, ty))`, which returns `Ok`.
  - `seedsigner_standard_signature_is_dangerous`: the standard fixture gives `SigClass::Dangerous` / `DangerousSignature`.
  - `seedsigner_frames_within_bounds`: every frame set satisfies `seq-count <= MAX_UR_FRAGMENTS`.
- [ ] **Step 2: Produce the unsigned PSBTs.**
  - In the regtest suite under `WREN_WRITE_FIXTURES=1`, write each type's claim PSBT (fixed coins, fee and nLockTime) to `tests/fixtures/seedsigner/<type>.unsigned.psbt`, base64.
  - The test seed is `OLD_WORDS` ("legal winner thank year wave sausage worth useful legal winner thank yellow"), which is public, so the fixtures carry no real keys.
- [ ] **Step 3: Capture on a real device (owner, once).**
  - Use firmware `v0.8.7-unified-sighash.6`, verified per the SeedSigner guide, with the test seed loaded.
  - Capture:
    - the `urca` and `spl` exports for all 4 script types (enable legacy in settings);
    - each `<type>.unsigned.psbt`, signed;
    - `p2wpkh.unsigned.psbt` with its sighash field removed, signed (the screen shows "Standard sighash").
  - Save each frame sequence as one frame per line.
  - In `SOURCES.md`, record the firmware tag, release SHA256, capture date and fragment size.
- [ ] **Step 4: Headless firmware signer in CI.**
  - `seedsigner_sign.py` loads the pinned seedsigner checkout and embit (both pinned by commit or sha256 in `run.sh`) from a separate venv, run with `python3 -I`.
  - It parses Wren's PSBT with `PSBTParser`, asserting acceptance (`SIGNABLE_SIGHASH_TYPES`, no `REFUSED_MIXED` or `REFUSED_PARTIAL`), signs it with `OLD_WORDS`, and prints the signed base64.
  - `run.sh` sets up the venv and exports `WREN_SEEDSIGNER_PY`.
  - In the claim loop of `regtest.rs`, sign each type with both the headless signer and Knots, and assert:
    - both `claim_finish` attempts verify (finish with the headless one; check Knots's result with `verify_and_finalize` only);
    - the two finalized transactions match exactly except for the signatures. ECDSA low-R grinding and Schnorr aux-rand differ between embit and Knots, so the bytes need not be identical.
- [ ] **Step 5: Run.** Run `cargo test -p wren-engine --test seedsigner_fixtures`, expecting PASS (4 tests), then `core/integration/run.sh`, expecting PASS.
- [ ] **Step 6: Commit.**
  ```bash
  git add core/wren-engine/tests/fixtures/seedsigner core/wren-engine/tests/seedsigner_fixtures.rs core/integration/seedsigner_sign.py core/integration/run.sh core/wren-engine/tests/regtest.rs
  git commit -m "test(engine): SeedSigner firmware fixtures and headless firmware signer in CI"
  ```
- [ ] **Step 7: Gate B.** Run the full claim flow once with the SeedSigner device or emulator against the regtest world. Record it in the Gate B review file.

---

## Owner data: pinned constants to capture before Task 5

### Area A — headers, sync, chain selection

Fill all of these on the owner's own nodes and cross-check them before Task 5 starts; Task 6 Step 0 checks them again. Notation: `fk` is `bitcoin-cli` against the fork Knots node; `btc` is `bitcoin-cli` against any legacy-chain (BTC) Core or Knots node. Use this Electrum helper (TLS):
```sh
E() { (printf '%s\n' '{"id":0,"method":"server.version","params":["wren-pin",["1.4","1.8"]]}' "$2"; sleep 8) \
      | openssl s_client -quiet -connect "$1" 2>/dev/null | sed -n 2p; }
```
For an onion server, wrap the pipe in `torsocks` and use `ncat` instead of `openssl` for plaintext.

1. **`PRE_ANCHOR_HEADER_ROOT`** (`ChainParams::mainnet().pre_anchor_root.root`, display hex). Placeholder: `"<OWNER:PRE_ANCHOR_HEADER_ROOT>"`.
   - R1, from the fork Fulcrum (default server):
     `E <fork-fulcrum>:50002 '{"id":1,"method":"blockchain.block.header","params":[0,961631]}' | jq -r .result.root`
   - R2, from a **BTC** Fulcrum or ElectrumX server (electrs and esplora don't support `cp_height`): the same command.
   - R3, recomputed from block hashes with `python3 core/scripts/header_merkle_root.py <rpc-url> <user> <pass> 961631`, run against `fk`'s RPC and, if possible, `btc`'s. The script, using only the standard library:
     ```python
     import base64, hashlib, json, sys, urllib.request
     url, user, pw, cp = sys.argv[1], sys.argv[2], sys.argv[3], int(sys.argv[4])
     auth = "Basic " + base64.b64encode(f"{user}:{pw}".encode()).decode()
     def rpc(calls):
         req = urllib.request.Request(url, json.dumps(calls).encode(), {"Authorization": auth, "Content-Type": "application/json"})
         return json.load(urllib.request.urlopen(req))
     leaves = []
     for s in range(0, cp + 1, 2000):
         r = sorted(rpc([{"id": h, "method": "getblockhash", "params": [h]} for h in range(s, min(s + 2000, cp + 1))]), key=lambda x: x["id"])
         leaves += [bytes.fromhex(x["result"])[::-1] for x in r]
     d = lambda b: hashlib.sha256(hashlib.sha256(b).digest()).digest()
     while len(leaves) > 1:
         if len(leaves) % 2: leaves.append(leaves[-1])
         leaves = [d(leaves[i] + leaves[i + 1]) for i in range(0, len(leaves), 2)]
     print(leaves[0][::-1].hex())
     ```
   - Requirement: **R1 = R2 = R3.**
   - Also save the fixture: `E <fork-fulcrum>:50002 '{"id":1,"method":"blockchain.block.header","params":[900000,961631]}' | jq .result > core/wren-engine/tests/fixtures/cp-proof-900000.json`.
   - Confirm the branch has 20 entries.
2. **`PRE_ANCHOR_CHAIN_WORK`** (chainwork at 961,631, display hex). Placeholder: `"<OWNER:PRE_ANCHOR_CHAIN_WORK>"`.
   - `fk getblockheader $(fk getblockhash 961631) | jq -r .chainwork` gives W1.
   - `btc getblockheader $(btc getblockhash 961631) | jq -r .chainwork` gives W2.
   - Require W1 = W2 and the same 961,631 hash on both.
   - Also capture **`KNOTS_CHAINWORK_961632`** for the Task 6 test: `fk getblockheader 0000000000000000000169eb6f811ddbd0daf343af7b62180cdb13e7c78dbc16 | jq -r .chainwork`. The test checks that it equals W1 + `block_proof(anchor bits)`.
3. **`ANCHOR_PREV_TIMES`** (10 × u32, heights 961,622-961,631, oldest first). Placeholder: `/* <OWNER:ANCHOR_PREV_TIMES> */`.
   - `for h in $(seq 961622 961631); do fk getblockheader $(fk getblockhash $h) | jq .time; done`
   - Repeat with `btc`; the output must be identical (shared history).
   - Fixture: `python3 core/scripts/export_headers.py <fork-server> 50002 961622 961631 core/wren-engine/tests/fixtures/mainnet-961622-961631.hex --tls`.
   - Repeat the export from a BTC Electrum server into a temporary file; `cmp` must report no difference.
   - The Task 6 test re-derives the times and the links into the anchor.
4. **`tests/vectors/mainnet-retargets.txt`** (one line per real retarget since activation):
   ```sh
   for R in $(seq 963648 2016 $(fk getblockcount)); do
     F=$(fk getblockheader $(fk getblockhash $((R-2016))) | jq .time)
     P=$(fk getblockheader $(fk getblockhash $((R-1))))
     X=$(fk getblockheader $(fk getblockhash $R) | jq -r .bits)
     echo "$R $(echo "$P" | jq -r .bits) $F $(echo "$P" | jq .time) $X"
   done > core/wren-engine/tests/vectors/mainnet-retargets.txt
   ```
   - Cross-check: for each R, export headers R-2016, R-1 and R from a second fork Electrum server (default Fulcrum, or `electrum.lazarus-xbt.xyz:50002`) with `export_headers.py`. Confirm that the times (effective time) and bits match the file.
   - Record the file's sha256 in `tests/vectors/SOURCES.md`.
5. **`min_chain_work`** comes from source, not the node. Confirm `consensus.nMinimumChainWork` at `src/kernel/chainparams.cpp:149` of tag `v29.4.2.knots20260508` equals `00000000000000000000000000000000000000013e00277374c9f9eeadc70200`. If a newer pinned Knots release is adopted, re-pin it from that tag.

## Spec changes (applied to the spec in this pull request)

### Area A — headers, sync, chain selection

- **§4 "Wrong-chain guard"** (spec L227-233), replacement:
  > - **Wrong-chain guard:** on every connect, the engine fetches the server's header at the fork's BLAKE2b activation height and checks that it is a valid 164-byte BLAKE2b header matching the pinned checkpoint. It then verifies every header from the pinned split block to the server's tip. A connection counts as verified only when that chain has reached activation and carries at least the minimum total work pinned from the signed fork Knots release. The engine does not rely on the server's own claims, because not every fork indexer reports its fork point. It **refuses** any server that fails, including every main-chain server and any server that has not reached activation. Sending is impossible without a verified fork connection. When a server offers a different branch, the engine switches only to a fully verified branch with more total work, never more than 144 blocks back. Confirmations of coins from before the split count only when the server proves the block against a pinned checkpoint of the shared pre-split history. Otherwise those coins are shown as unverified and cannot be claimed.
- **§5 Home**: replace "A status line showing the Tor and server connection." with:
  > A status line showing the Tor and server connection, with a warning when the server's latest block is more than 2 hours old.
- **§8 Testing**: add the bullet:
  > **Header-chain tests:** weaker, equal-work and invalid competing branches are refused and the stored chain is unchanged; a legacy server below activation is refused; a fabricated pre-split confirmation is never counted.
- **§10 Risks**: add the row:
  > | One server shows a privately mined branch with fake confirmations | Full header verification, minimum total work and most-work selection limit what it can fake; stale-tip warning; Nest asks for 6 confirmations (Nest plan). A second-server cross-check is deferred past v1. |

### Area B — server trust in everyday use

- **§3.1 Public API (L117-140):**
  - `balance()` becomes `balance() -> { spendable, incomingPending, immature }`.
  - `transactions()` becomes `transactions()` (each with a confirmation count).
  - `signAndBroadcast(reviewId)` becomes `signAndBroadcast(reviewId) -> Sent | Rejected(reason) | Unknown`.
  - `feeEstimates()` becomes `feeEstimates() -> slow, normal, fast, unreliable`.
  - `claimFinish(claimId, signedPsbtQr) -> txids` becomes `claimFinish(claimId, signedPsbtQr) -> per-transaction Sent | Rejected(reason) | Unknown`.
- **§4, after "Send review" (L247-250), add:**
  - The review shows a "To yourself" notice when the destination is one of the wallet's own addresses.
  - When the fee is 10% or more of the amount, or the rate is above 200 sat/vB, the review asks for a second confirmation.
  - Wren never builds a payment above 1,000 sat/vB.
  - Server error text is never shown; Wren uses its own fixed messages.
- **§5 Home (L310):** replace "The XBT balance as the centerpiece…" with:
  - "The spendable XBT balance as the centerpiece on a frosted panel, with a sats toggle. Unconfirmed incoming payments appear on a separate 'Incoming (unconfirmed)' line and become spendable only once confirmed. The wallet's own unconfirmed change counts as spendable."
- **§5 Receive (L315-316):** replace "and a fresh address each time" with:
  - "and a fresh address each time, until 10 shown addresses are still unpaid. From then on Wren shows the oldest unpaid address again, so a restore from the 12 words always finds every payment."
- **§5 Send step 3 (L320):** replace with:
  - "Choose a fee: slow, normal, or fast from server estimates, or custom (1 to 1,000 sat/vB). If the server's estimates look unreasonable, Wren uses the minimum fee and says so."
- **§5 Send step 6 (Success screen):** replace with:
  - "Result: 'Sent'. Or 'Sent, but the server did not confirm it' (Wren keeps rebroadcasting; *Speed up* stays available). Or 'The server rejected it: <reason>' (the payment stays listed; *Speed up* re-sends the same coins, so it can never pay twice)."
- **§5 Speed up (L324):** replace with:
  - "**Speed up:** offered only on a pending payment signed by this wallet's own key, including one whose result was 'not confirmed' or 'rejected'. It replaces the payment by re-spending the same coins at a higher fee (RBF)."
- **§5 Activity detail (L348-349):** keep "Confirmation count and transaction ID." Incoming unconfirmed items carry an "Unconfirmed — not spendable yet" tag.

### Area C — network, Tor, TLS, Electrum

- **§3.1, Tor bullet (L114-116, "The engine opens all server connections itself, over Tor or direct SSL"):** append "Unencrypted connections are allowed only to onion addresses. A custom SSL server with a self-signed certificate is trusted by its public key (SPKI SHA-256). That key is captured over the same route the wallet uses (Tor when Tor is on)."
- **§4.1, Scanning (L264-267, "Queries go to the fork server over Tor."):** replace with "Claim requires Tor. The scan and the broadcast run on their own Tor circuit and Electrum session, separate from the wallet's, so the server cannot link the two. With Tor off, Claim asks the user to turn it on."
- **§5, Home (L313, "A status line showing the Tor and server connection."):** becomes "A status line showing the route actually in use (Tor or direct), the server connection, and Tor start-up progress."
- **§5, Settings (L353-354):** add the following.
  - "Custom SSL server with a self-signed certificate: Wren shows the certificate's SHA-256 fingerprint (colon-separated hex, as `openssl` shows it). The user compares it with their node before trusting it."
  - "If the server's key later changes, Wren stops and asks before trusting the new one."
  - "Unencrypted servers only as onion addresses."
  - "Turning Tor off asks for confirmation and never happens automatically."
- **§10, Risks, "Arti onion-client maturity" row (L497, "SSL direct mode as a fallback; Tor status visible on Home."):** becomes "SSL direct mode as a fallback the user chooses (never automatic). Tor start-up gives up after 2 minutes with a clear message. The actual route is shown on Home."

### Area D — key custody, engine API, PIN

- **§3 diagram, App box right column.** Replace `screens, camera, app lock,` / `secure storage of the seed` with:
  `screens, camera; native key vault:` / `Wren PIN pad, sealed seed`
- **§3.1, replace the first three API lines (L118-120):**
  > - `createWallet() -> words` (shown once)
  > - `restoreWallet(words, passphrase?)`, then `restoreCheck()` and `finishSetup()`
  > - `confirmBackup(answers)`
  > - `unlock()`, `lock()`
  > - `revealWords()` (always asks for the Wren PIN)
  > - `changePin()`

  Add after the list:
  > Sending asks the phone's key vault for the Wren PIN (or Face ID/fingerprint) inside the engine itself, so the app can never sign on its own. Between actions the engine holds no private key.
- **§3.2, replace the last sentence:**
  > The app never receives keys, the Wren PIN, or the key that unlocks the seed. The 12 words pass through the app only three times: when they are shown at Create, typed at Restore, and shown by View recovery phrase after the Wren PIN.
- **§3.3, replace the `expo-secure-store` and hardware-key bullets:**
  > - The engine encrypts the seed. Wren's own native key-vault module (Swift on iOS, Kotlin on Android) keeps the encrypted seed on this phone only (Keychain "this device only" on iOS, private no-backup storage on Android). It is never in iCloud or Google backups. The app's JavaScript cannot read, change, or delete it.
  > - The Wren PIN pad and the final "Confirm send" sheet are native screens. The PIN is never typed into the app's JavaScript, and the confirm sheet shows exactly what the engine will sign.
  > - The key behind the Wren PIN lives in the Secure Enclave (iOS) or a non-exportable Keystore key, StrongBox where available (Android).
- **§4, Storage bullet:**
  > - **Storage:** the engine encrypts the seed, and Wren's native key vault keeps it on this phone only, never in cloud backups.
- **§4, PIN length (replace "6 digits," in the first App lock bullet):**
  > Every wallet has a **Wren PIN**: 6 digits by default, set during Create or Restore, entered twice, and separate from the phone's own passcode. In Settings the user can switch to a longer PIN: 8 to 12 digits, or a passcode with letters. Obvious PINs (one repeated digit, straight runs such as 123456 or 654321) are refused at every length.
- **§4, add after "Every app open and every send asks…":**
  > Wren locks itself when the user leaves the app and after 5 minutes without use.
- **§4, replace "The PIN cannot be guessed off the phone." (L208-211):**
  > - **How the PIN protects the seed.** The seed is locked with a key made from two parts. The first is the Wren PIN, slowed down on purpose (Argon2id, about 1 second per try on the slowest supported phone). The second is a result that only this phone's security chip can produce, and that changes with every PIN tried. So every guess needs this exact phone.
  > - **What that means in practice:**
  >   - A copy of Wren's storage or a phone backup is useless anywhere else.
  >   - Someone holding the locked phone gets 5 tries, then growing waits.
  >   - Someone who can run their own code inside Wren on that phone (for example on a jailbroken or rooted phone) is not slowed by the waits. They can try every 6-digit PIN in roughly a week. A longer PIN makes that take years.
  >   - Face ID and fingerprint are protected by the phone's own hardware limits.
- **§4, replace the Wrong PIN bullet:**
  > - **Wrong PIN:** after 5 wrong tries, waits grow: 1 minute, 5 minutes, 15 minutes, 1 hour, then 24 hours per try. Each try is counted and saved before it is checked, so closing the app mid-try still counts. Waits are timed by the phone's internal clock, so changing the date or time does not shorten them; restarting the phone starts the current wait again. The counter is kept on this phone only. Wren never auto-erases the wallet, because that could destroy coins the user has not backed up yet.
- **§4, add "Typing safety":**
  > - **Typing safety:** the Wren PIN uses Wren's own number pad, never the phone keyboard. The 12-word field turns off autocorrect, spell-check, word suggestions, autofill and keyboard learning. On iOS, Wren does not allow third-party keyboards anywhere in the app. On Android, Wren asks the keyboard not to learn or suggest, but cannot block third-party keyboards.
- **§4, add "App switcher":**
  > - **App switcher:** when Wren leaves the screen, it is covered with a solid cover, so the recovery phrase, balance and addresses never show in the recent-apps view. Screenshots are blocked on seed screens (on Android always; on iOS as far as the system allows).
- **§4, replace the Reinstall bullet (L218-220):**
  > - **Wallet from an earlier install:** iOS can keep Wren's saved seed after the app is deleted. If Wren finds one, it never deletes it silently. It shows: "A Wren wallet from an earlier install is on this phone. Unlock it with its Wren PIN, or delete it." Deleting needs the user to type a confirmation and warns: "Only delete it if you have your 12 words." The wrong-PIN count carries over. After unlocking, sending stays locked until the backup check is done again.
- **§4, L238-239.** Replace "the Restore warning below prevents the mix-up it could cause" with "the Restore warning and the BTC-seed check below guard against the mix-up it could cause".
- **§4, add after the Restore warning.** (This is the BTC-seed check. It goes in §4 rather than §4.1, because it is about Restore, not Claim.)
  > - **BTC-seed check:** after the restore scan, and before anything is saved, Wren looks for wallet activity from before the split (block 961,632). If it finds any, this phrase was used for bitcoin and holds or held BTC. Wren does not save it and shows: "This recovery phrase was used for bitcoin before the split, so it holds or held BTC. Wren will not keep it. Typing it into a phone put your BTC at risk: move your BTC to a new wallet with your hardware wallet now. To claim the XBT twins, use Claim with a hardware signer, or restore this phrase in Shrike on a computer."
  >   If such activity only shows up later, Wren blocks all sending except one "Send all" to move the XBT out, after the user confirms they understand.
  >   Wren cannot spot a phrase that was only used for bitcoin after the split, so the Restore warning stays.
- **§5 Restore, replace:**
  > - **Restore:** the Restore warning (§4), 12-word entry with BIP39 autocomplete, a passphrase field under "Advanced," a history scan with the BTC-seed check (§4), then the Wren PIN setup (Create, step 4). Restoring needs a verified server connection; nothing is saved until the check passes.

  Settings, App lock bullet: add "or switch to a longer PIN".
- **§8 App lock tests, add:**
  > - A wrong PIN still counts if the app is closed during the check.
  > - Changing the phone's date or time does not shorten a wait.
  > - The saved seed copied to another phone cannot be unlocked with the right PIN.
  > - A wallet from an earlier install is never deleted without the typed confirmation.
  > - The app's JavaScript cannot send, view the recovery phrase, or change the PIN without the native PIN pad or biometrics.
  > - Wren locks on leaving the app and after 5 minutes idle.

  Engine tests, add:
  > - **BTC-seed check:** a restored phrase with pre-split history is never saved; history found later allows only one send-all.
  > - No seed or key bytes are ever written to Wren's files.
- **§10, replace the PIN risk row:**
  > | A 6-digit PIN has only 1,000,000 combinations | Each guess needs this phone's security chip, so a copy of Wren's storage or a backup is useless elsewhere. Someone holding the locked phone gets 5 tries, then waits up to 24 hours per try. But someone who can run their own code inside Wren on that phone (a jailbroken or rooted phone, or a tampered app) skips the waits and could try every 6-digit PIN in about a week. Users can choose a longer PIN (8 to 12 digits, or with letters), which raises this to years. Argon2id cost and the wait schedule are reviewed at Gate B and in the external review. |

### Area E — Claim, links, QR, tests, CI

**§2 L69 (Claim your XBT row), replace the last sentence.** Replace "No standalone "check for twins" tool: if a claim scan finds nothing, the user knows they have no twins." with:

> No standalone "check for twins" tool. When a claim scan finds nothing, Wren says what it checked and why it may have missed coins.

**§3.4, append:**

> A plain address, scanned or pasted without `xbt:`, is accepted, but Send warns: "This is a plain address. Make sure the person expects **XBT (BLAKE2b chain)**, not BTC." "XBT" is also used for BTC on some exchanges, so the app labels the coin "XBT (BLAKE2b chain)" wherever a mix-up would cost money.

**§4.1 L264-267 (Scanning), replace "Queries go to the fork server over Tor. The server sees the scanned addresses (as script hashes), never the xpub." with:**

> Queries always go over Tor, on a separate connection that is never used by the Wren wallet, so the server can't link the claimed wallet to the Wren wallet. If Tor is off, Claim says "Claim needs Tor. Turn on Tor in Settings." The server sees the scanned addresses (as script hashes), never the xpub. The server must prove pre-split coins with a checkpoint proof; coins it can't prove are listed as not claimable.

**§4.1 L268-272 (Supported wallet types), append:**

> An export that includes any other type (multisig, Taproot with scripts) is refused as a whole, naming the type.

**§4.1 L273-274 (Unsigned claim), replace with:**

> **Unsigned claim:** every input's PSBT sighash field is set to SIGHASH_UNIFIED. All found coins are swept; there is no change output. Coins worth less than the fee to move them, unconfirmed coins and coins the server couldn't prove are left out and listed with the reason. Each claim transaction pays a fresh address in the Wren wallet.

**§4.1 L281-283 (Wrong signature type), replace with:**

> - **Unsafe signature:** if any signature could be valid on the BTC chain (a normal signature, or a 0x21 byte over a normal digest), Wren sends nothing, wipes the scanned QR data, and shows the "Stop" screen (§5, Claim step 8). Signatures that are simply wrong get "A signature doesn't check out. Nothing was sent."

**§4.1 L284-286 (Many coins), replace with:**

> - **Many coins:** the claim is split so that each transaction's QR stays small enough for the SeedSigner to scan (a 50 kB PSBT limit to start, set by device testing), and so no transaction exceeds standard size. The transactions are signed one after another. The review screen says how many there are before signing starts.

**§5 Claim steps 3, 4, 6, 7, 8, 9 (L331-344), replace with:**

> 3. Scan: shows the XBT and coin count found, and any coins that can't be claimed, with the reason. If none: "**This server found no XBT for this account.**" It then lists the reasons it may have missed coins: a different wallet type or account, a missing passphrase, coins past the first {20 | 200} addresses ("Search further"), coins received after the split, coins not yet confirmed. Then: "This doesn't prove the wallet has none." Then stop.
> 4. Destination: the user's Wren wallet by default, with a fresh address for each claim transaction. "Send elsewhere" accepts any XBT address by scan or paste, shown in full, with the clipboard flag.
> 6. Review: sweep of all claimable coins, no change, the Replay-safe badge, and "This moves XBT only. Your BTC stays where it is." It also says: "These coins are spent together, which shows they belong to one wallet, on both chains."
> 7. Sign. First, a "**Before you sign**" card: "On your SeedSigner, look for the line **Unified sighash (0x21)**. If it says **Standard sighash**, don't sign. Press back. Your signer needs the fork firmware. See the guide." Then Wren shows an animated QR; the SeedSigner scans it, shows the details on its own screen, and signs; Wren scans the signed QR back.
> 8. Verify and broadcast (§4.1). If a signature is unsafe, Wren shows the "Stop" screen:
>    **Stop. This signature isn't safe.**
>    Your signer made a normal bitcoin signature. Wren did not send it.
>    But the signature still exists, on your signer's screen and in the QR code. Anyone who copied it could use it to move these coins to {destination}, on the BTC chain and on the XBT chain.
>    To make it useless:
>    1. Update your SeedSigner to the fork firmware (see the guide).
>    2. Sign this claim again and finish it. This moves your XBT.
>    3. Then move your BTC to a new address with the wallet you normally use for BTC.
>    Until you've done step 3, keep your signer and this phone with you.
>    [Open the guide]  [Sign again]
> 9. Success: "**Done. The {N} coins listed are now split: their BTC and XBT are separate.**" Then any coins not claimed, each with its reason (not yet confirmed, worth less than the fee, too large for QR, couldn't be proven). Then: "**Claim before you spend BTC from this wallet.** Coins this scan didn't find are not split, and spending their BTC could also move their XBT." The xpub is wiped.

**§8, replace the Replay proof and Claim tests bullets (L429-444):**

> - **Replay proof, on every build (required CI check):** every transaction the engine signs that spends a pre-split coin, including all four claim types, is mined on a Bitcoin Core regtest node that shares the pre-split history, and **must be rejected by consensus** with the expected script error. As a control, the same transaction signed normally must be accepted there.
> - **Claim tests, on every build (required CI check):**
>   - **Signer compatibility:** real SeedSigner fork-firmware exports and signed PSBTs (captured from a device) are imported and verified, and the firmware's own PSBT parser and signer, run without its screen, sign each claim type in the local test chain. The full device or emulator run happens at Gate B and before release.
>   - **Refusals:** the engine must refuse to broadcast when a signature is unsafe (normal, or BTC-valid), an output was changed after review, an input was added or removed, or the fee was altered. Unsafe signatures made by real software that doesn't know the fork (Bitcoin Core) must be refused for every claim type.
>   - **QR round trip:** large animated BC-UR PSBTs survive encode, scan, and decode intact. QR codes claiming absurd sizes, or that never complete, are refused.
>   - **Wipe:** after finish, cancel, or a force-quit mid-claim, no trace of the xpub remains in app storage.
>   - **Supported types:** every script type Claim accepts is signed and verified end to end; every other type is refused before scanning.
> - **Mainnet data rules in the test chain:** the local test chain turns on the fork's data limits and long coinbase maturity, so tests run under mainnet rules.

**§8 Claim trials (L462-464), replace the first sub-bullet with:**

> - At Gate B: a SeedSigner (privkeyio unified-sighash firmware, real device or emulator) claims from a test wallet on the local test chain, running the full flow on a real phone. Its exports and signed PSBTs become the test fixtures.

## docs/future-ideas.md

### Area E — Claim, links, QR, tests, CI

Replace the "**Open question:**" bullet (L29-32) and the "Revisit when" line (L34-35) with:

> - **Wrong-chain invoices (answered in the Gate A review):** XBT and BTC Lightning share the genesis hash (the chain hash in BOLT12 and gossip) and the address prefixes, so BOLT11 invoices on both start `lnbc`. The prefix and chain hash can't tell the chains apart, and an invoice's `f` fallback address is valid on both chains. If Wren adds Lightning, it must require the fork's even BLAKE2b feature bit (512) in the invoice's own features, refuse invoices without it, and ignore `f` fallbacks.
> - **Still open:** whether the fork's Lightning software sets bit 512 in invoices, or only in node and connection features. Ask the fork's Lightning developers.
>
> **Revisit when:** there is more than one LSP, the fork's Lightning software has a track record, and invoices are confirmed to carry the BLAKE2b feature bit.

---

**Merge notes:**
- In Tasks 15-22, every `EngineError::Server(format!(…{e}))` on the claim broadcast path is left to Area C/D's "no raw server text" change.
- Claim build's fee cap, the immature-coinbase predicate, and `Unverified` coins come from Areas D and B; the `NotClaimedReason` variants above are the hooks for them.
