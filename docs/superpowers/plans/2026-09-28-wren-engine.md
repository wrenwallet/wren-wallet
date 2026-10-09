# Wren Engine (Rust core) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build `wren-engine`, the Rust library that holds every key- and money-touching part of Wren Wallet: BLAKE2b fork header verification, SIGHASH_UNIFIED signing, an Electrum client, a BDK-backed wallet, payments, and the hardware-signer Claim flow. It is proven on a local fork regtest chain before any phone code exists.

**Architecture:** BDK (`bdk_wallet`) does wallet bookkeeping only: descriptors, addresses, the transaction graph, coin selection, and PSBT construction. Everything fork-specific is Wren's own small, test-vector-driven code: header parsing/hashing/PoW (`header`, `pow`, `headerchain`), the unified sighash digest (`sighash`), signing (`sign`), our own Electrum JSON-RPC client (`electrum`), the chain source that feeds BDK through `Wallet::apply_update` (`sync`), and Claim (`claim`). We do not use `bdk_electrum`/`electrum-client` (they decode 80-byte headers) or BDK's signer (it refuses non-standard sighash bytes), and we never deserialize a signed PSBT with `rust-bitcoin` (its partial-signature decoder rejects the `0x21` byte). The app plan (sub-plan 2) wraps this crate with UniFFI; this plan exposes a plain Rust API.

**Tech Stack:** Rust (pinned toolchain), `bdk_wallet` 3.2 (features `rusqlite`, `keys-bip39`) and its re-exported `bitcoin` 0.32 / `secp256k1`, `blake2` 0.10, `tokio`, `tokio-rustls`/`rustls`/`webpki-roots`, `serde_json`, `thiserror`, `zeroize`, `ur` 0.5 (BC-UR fountain codes), `ciborium` (CBOR), `hex`, `base64`. Integration: Docker (colima) running fork Knots `v29.4.2.knots20260508`, privkeyio Fulcrum `v2.1.2-blake-3`, and Bitcoin Core 29 as the independent legacy-chain node; Knots `walletprocesspsbt` is the automated stand-in for the SeedSigner signer (the real device is tested in sub-plan 2).

**Spec:** `docs/superpowers/specs/2026-09-24-wren-wallet-design.md` (commit `cec2608`).
**Research (facts this plan relies on, with sources):** `docs/superpowers/research/2026-09-28-fork-consensus-facts.md`, `docs/superpowers/research/2026-09-28-fork-wallet-tooling.md`.

This is **sub-plan 1 of 3** for Wren Wallet v1:
1. **Engine** (this plan).
2. **Bridge + app:** UniFFI bindings (`uniffi-bindgen-react-native`), Expo React Native screens, Arti/Tor on-device, secure storage. Written after this plan's signing-milestone review.
3. **Server:** rented fork Knots + Fulcrum host (`infra/`). Written when beta is near (spec §7).

## Global Constraints

- XBT only. The engine never builds, signs, or broadcasts anything for the legacy chain, and never connects to a legacy-chain server (spec §2, §4).
- Every signature the engine produces uses hash type `0x21` (`SIGHASH_ALL | SIGHASH_UNIFIED`). There is no code path, flag, or setting that signs anything else (spec §4).
- Every Claim PSBT input carries `PSBT_IN_SIGHASH_TYPE = 0x21` (`PsbtSighashType::from_u32(0x21)`); the SeedSigner fork firmware only signs UNIFIED when the PSBT asks, and refuses mixed types (research: tooling §SeedSigner).
- Wren never accepts, parses, or stores a seed for the wallet being claimed. Claim input is an xpub only (spec §4.1).
- The claim xpub and ephemeral claim wallet live in memory only and are dropped on finish, cancel, or engine drop (spec §4.1).
- Addresses: Wren's own wallet is P2WPKH, BIP84 `m/84'/0'/0'`, network `Bitcoin` (fork uses mainnet address params) (spec §4).
- BLAKE2b activation height `961_640`; chain split (first BIP110 block) `961_632`; checkpoint hashes exactly as in the research file.
- Electrum protocol: negotiate the range `["1.4", "1.8"]`; accept both the `hex` (concatenated, variable-width) and `headers` (list) forms of `blockchain.block.headers`.
- Fee floor: 1 sat/vB (min relay fee 1000 sat/kvB).
- No analytics, no telemetry, no logging of addresses, xpubs, amounts, txids, or keys at any log level (spec §3.3).
- Private key material (`Mnemonic`, seed bytes, `Xpriv`) is held in `zeroize`-on-drop wrappers and never crosses the public API except `create_wallet`'s one-time word list (spec §3.1, §4).
- Dependencies are added with `cargo add` and pinned by the committed `Cargo.lock`; `bitcoin`/`secp256k1`/`miniscript` are only used through `bdk_wallet`'s re-exports so versions never split.
- Money amounts are `bitcoin::Amount` (sats) inside the engine; no floating point except when parsing Electrum fee estimates.

## Review Focus

Inputs and conditions the spec implies that are most likely to hurt a real user, each pinned by a test in the task that owns it:

1. **A server that lies about or omits a transaction's prevout** (wrong amount, missing input). Expected: the engine fetches every parent transaction, checks its txid, and takes amounts only from it; a claim or payment with any missing prevout refuses to build. Test in Task 13 (`sync_rejects_tx_whose_hash_mismatches`) and Task 18 (`claim_build_requires_every_prevout`).
2. **A reorg across the verified tip while the app was closed.** Expected: header sync walks back to the fork point (up to 144 blocks), re-verifies, and never keeps a stale tip; deeper than 144 is refused. Test in Task 11 (`sync_handles_reorg_within_window`, `sync_refuses_reorg_deeper_than_window`).
3. **A signed PSBT whose unsigned transaction differs from the one Wren built** (swapped output, added input, changed fee, dropped input). Expected: refused before broadcast, nothing sent. Tests in Task 19 (`changed_transaction_is_refused_for_each_kind_of_change`, `nothing_is_broadcast_unless_every_transaction_verifies`).
4. **Coinbase outputs under the long-maturity rule** (created at height ≥ 973,440 need 6,480 confirmations until height 979,920). Expected: never selected for spending until mature. Test in Task 14 (`long_maturity_coinbase_is_not_spendable`).
5. **A pasted or scanned address with the wrong network/format, or a `bitcoin:` URI.** Expected: `bitcoin:` URIs are rejected with a clear "not an XBT link" error; testnet/regtest addresses rejected on mainnet. Test in Task 15.

## Review Gates (humans, outside the task list)

- **Gate A — before Task 1 starts:** an experienced bitcoin-wallet developer (ideally the privkeyio/Shrike maintainer, or another fork wallet developer) reviews this plan, the spec, and both research files. Focus: the header chain verification approach (Tasks 6, 11), the unified sighash port (Task 7), the decision to parse signed PSBT signatures ourselves (Task 19), the trust model for pre-split merkle proofs (Task 13), and the two test-reference deviations in Task 22 (Knots instead of Shrike for byte-exact cross-signing; Knots `walletprocesspsbt` as the automated SeedSigner stand-in). Findings are recorded in `docs/superpowers/reviews/` and the plan is amended before coding.
- **Gate B — the signing milestone, after Task 22 passes:** the same reviewer (or a second one) reviews the engine code for Tasks 3–8 and 17–19 plus the integration results (replay proof, wrong-chain proof, cross-sign byte equality). Sub-plan 2 is not written until Gate B passes.
- **Gate C — before public release** (spec §8): formal external security review. Out of scope for this plan.

---

## File Structure

```
WrenWallet/
  CLAUDE.md                              project scope guard + hard rules for agents
  core/
    Cargo.toml                           workspace (members: wren-engine)
    rust-toolchain.toml                  pinned stable toolchain
    scripts/check.sh                     fmt + clippy + test + audit, used locally and in CI
    wren-engine/
      Cargo.toml
      src/
        lib.rs                           module list + public re-exports
        error.rs                         EngineError (one error enum for the public API)
        hashes.rs                        sha256, sha256d, tagged_sha256, blake2b256
        header.rs                        ForkHeader: parse/serialize 80|164 bytes, time, hash
        pow.rs                           compact <-> target, check_pow, next_work_required
        params.rs                        ChainParams (Mainnet, Regtest) + checkpoints
        headerchain.rs                   HeaderChain: verified header list, extend, reorg, persist
        sighash.rs                       unified_sighash(): the SIGHASH_UNIFIED digest
        sign.rs                          sign_p2wpkh_unified(): hot-wallet signing + witness
        electrum/
          mod.rs                         ElectrumClient: typed methods over JSON-RPC
          rpc.rs                         newline JSON-RPC framing over any AsyncRead+AsyncWrite
          transport.rs                   TCP, TLS (webpki or pinned self-signed), connector trait
        sync.rs                          header sync + script sync -> bdk_wallet::Update
        wallet.rs                        WrenWallet: BDK PersistedWallet + in-memory signing key
        payment.rs                       fee estimates, build/review, sign+broadcast, bump
        uri.rs                           xbt: payment links
        ur.rs                            BC-UR animated QR: crypto-psbt encode/decode
        xpub_import.rs                   claim xpub formats: UR crypto-account, [fp/path]xpub, bare
        claim/
          mod.rs                         ClaimSession lifecycle (start/scan/build/finish/cancel)
          scan.rs                        ephemeral watch-only wallet + gap scan
          build.rs                       sweep PSBTs (0x21 on every input), splitting
          psbt_sigs.rs                   minimal BIP174 reader: per-input sigs from a signed PSBT
          verify.rs                      the five pre-broadcast checks + finalization
        engine.rs                        WrenEngine: the public API the app will call
      tests/
        vectors/                         Knots + drongo test vectors, SOURCES.md with sha256
        fixtures/                        real mainnet fork headers exported from a fork node
        vectors_test.rs                  vector-file integrity check
        regtest.rs                       integration tests (ignored unless WREN_REGTEST=1)
    integration/
      docker-compose.yml                 fork Knots regtest, Fulcrum, Bitcoin Core regtest, Fulcrum-on-Core
      knots/Dockerfile, fulcrum/Dockerfile, core/Dockerfile   signature-verified release installs
      run.sh                             fresh world + `cargo test --test regtest`
```

---

### Task 1: Repository guardrails, Rust workspace, and test vectors

**Files:**
- Create: `CLAUDE.md`
- Create: `core/Cargo.toml`, `core/rust-toolchain.toml`, `core/scripts/check.sh`
- Create: `core/wren-engine/Cargo.toml`, `core/wren-engine/src/lib.rs`, `core/wren-engine/src/error.rs`
- Create: `core/wren-engine/tests/vectors/SOURCES.md`, `core/wren-engine/tests/vectors/unified_sighash.json`, `core/wren-engine/tests/vectors/block_header_v2.json`, `core/wren-engine/tests/vectors/mainnet-activation.txt`
- Test: `core/wren-engine/tests/vectors_test.rs`

**Interfaces:**
- Consumes: nothing.
- Produces: crate `wren_engine`; `wren_engine::error::EngineError` (enum, `thiserror`); vector files at `tests/vectors/` read by later tasks via `concat!(env!("CARGO_MANIFEST_DIR"), "/tests/vectors/<file>")`.

- [ ] **Step 1: Write `CLAUDE.md` at the repo root**

```markdown
# Wren Wallet — agent rules

## Project scope
This repo is the XBT project (Wren Wallet; Wren Nest later). Work only inside this repository.

## Hard rules (never break)
- XBT (BLAKE2b fork chain) only. Never build, sign, or broadcast for the legacy SHA256d chain.
- Every signature Wren makes uses hash type 0x21 (SIGHASH_ALL | SIGHASH_UNIFIED). No exceptions,
  no flags, no test-only bypass in non-test code.
- Payment links use `xbt:` only. Never emit or accept `bitcoin:`.
- Never ask for, accept, parse, or store a seed for a wallet being claimed. Claim takes an xpub.
- Never log keys, seeds, xpubs, addresses, amounts, or txids.
- Never ship a developer's personal node as a default server.
- Facts about the fork come from docs/superpowers/research/*.md; if a fact is missing, stop and
  ask — do not guess consensus details.

## Where things are
- Spec: docs/superpowers/specs/2026-09-24-wren-wallet-design.md
- Engine plan: docs/superpowers/plans/2026-09-28-wren-engine.md
- Engine crate: core/wren-engine ; checks: core/scripts/check.sh
```

- [ ] **Step 2: Create the workspace and toolchain files**

`core/Cargo.toml`:
```toml
[workspace]
resolver = "2"
members = ["wren-engine"]

[workspace.package]
edition = "2021"
license = "MIT"

[profile.release]
lto = true
codegen-units = 1
panic = "abort"
```

`core/rust-toolchain.toml`:
```toml
[toolchain]
channel = "1.98.1"
components = ["rustfmt", "clippy"]
```

- [ ] **Step 3: Create the crate and add dependencies**

Run:
```bash
cd core && cargo new --lib wren-engine --vcs none
cd wren-engine
cargo add bdk_wallet@3.2 --features rusqlite,keys-bip39
cargo add blake2@0.10 thiserror@2 zeroize@1 --features zeroize/derive
cargo add tokio@1 --features rt-multi-thread,net,io-util,macros,time,sync
cargo add rustls@0.23 --no-default-features --features ring,std,tls12
cargo add tokio-rustls@0.26 --no-default-features --features ring,tls12
cargo add webpki-roots@1
cargo add serde@1 --features derive
cargo add serde_json@1 hex@0.4 base64@0.22 ur@0.5 ciborium@0.2
cargo add --dev tempfile@3
```
Expected: `Cargo.toml` lists these; `cargo build` succeeds.

- [ ] **Step 4: Write `src/lib.rs` and `src/error.rs`**

`src/lib.rs`:
```rust
//! Wren Wallet engine: every key- and money-touching part of Wren.
#![forbid(unsafe_code)]

pub mod error;

pub use bdk_wallet::bitcoin;
pub use error::EngineError;
```

`src/error.rs`:
```rust
use thiserror::Error;

/// The single error type the public API returns. Variants carry no keys, addresses, or amounts.
#[derive(Debug, Error)]
pub enum EngineError {
    #[error("invalid block header: {0}")]
    Header(&'static str),
    #[error("header chain rejected: {0}")]
    Chain(&'static str),
    #[error("server is not on the XBT (BLAKE2b) chain")]
    WrongChain,
    #[error("signing refused: {0}")]
    Signing(&'static str),
    #[error("server error: {0}")]
    Server(String),
    #[error("network error: {0}")]
    Network(String),
    #[error("wallet error: {0}")]
    Wallet(String),
    #[error("invalid input: {0}")]
    Input(&'static str),
    #[error("claim refused: {0}")]
    Claim(&'static str),
    #[error("not connected to a verified XBT server")]
    NotVerified,
}
```

- [ ] **Step 5: Copy the test vectors and record their sources**

Run (from `core/wren-engine`):
```bash
mkdir -p tests/vectors
curl -sL https://raw.githubusercontent.com/bitcoinknots/bitcoin/v29.4.2.knots20260508/src/test/data/unified_sighash.json -o tests/vectors/unified_sighash.json
curl -sL https://raw.githubusercontent.com/privkeyio/drongo/1577a746cdaa7650de9e30b83dd0fa6be8a59771/src/test/resources/block_header_v2.json -o tests/vectors/block_header_v2.json
curl -sL https://raw.githubusercontent.com/privkeyio/drongo/1577a746cdaa7650de9e30b83dd0fa6be8a59771/src/test/resources/headers/mainnet-activation.txt -o tests/vectors/mainnet-activation.txt
shasum -a 256 tests/vectors/*
```
Expected output exactly:
```
ed496bcaf60d212fce20a795d8d57e538fe3a8713ca502e7392e25f027576837  tests/vectors/block_header_v2.json
1ba97e30f147eb78c5bb985122dd3dcb32278ecd32cce2a5cdce955c9c10b628  tests/vectors/mainnet-activation.txt
5c5e95fc1ab8ef9ce6b3cb6e76b8c74a987d182ebd01d8e2b98b6d0fbb26f630  tests/vectors/unified_sighash.json
```
If any hash differs, stop and report — do not continue with different vectors.

`tests/vectors/SOURCES.md`:
```markdown
# Test vector sources

| File | Source | sha256 |
|---|---|---|
| unified_sighash.json | bitcoinknots/bitcoin tag v29.4.2.knots20260508 (tag object 80a7d1f31c5db19e5223fc42e0d9a974737c6c8a), src/test/data/unified_sighash.json, MIT | 5c5e95fc1ab8ef9ce6b3cb6e76b8c74a987d182ebd01d8e2b98b6d0fbb26f630 |
| block_header_v2.json | privkeyio/drongo commit 1577a746cdaa7650de9e30b83dd0fa6be8a59771, src/test/resources/block_header_v2.json, Apache-2.0 | ed496bcaf60d212fce20a795d8d57e538fe3a8713ca502e7392e25f027576837 |
| mainnet-activation.txt | privkeyio/drongo commit 1577a746cdaa7650de9e30b83dd0fa6be8a59771, src/test/resources/headers/mainnet-activation.txt, Apache-2.0. Raw headers for fork mainnet heights 961,632..961,641 (8 × 80-byte SHA256d, then 2 × 164-byte v2). | 1ba97e30f147eb78c5bb985122dd3dcb32278ecd32cce2a5cdce955c9c10b628 |
```

- [ ] **Step 6: Write the failing vector-integrity test**

`tests/vectors_test.rs`:
```rust
use bdk_wallet::bitcoin::hashes::{sha256, Hash};

fn file_sha256(name: &str) -> String {
    let path = format!("{}/tests/vectors/{}", env!("CARGO_MANIFEST_DIR"), name);
    let bytes = std::fs::read(&path).expect("vector file present");
    sha256::Hash::hash(&bytes).to_string()
}

#[test]
fn vector_files_match_recorded_hashes() {
    assert_eq!(file_sha256("unified_sighash.json"), "5c5e95fc1ab8ef9ce6b3cb6e76b8c74a987d182ebd01d8e2b98b6d0fbb26f630");
    assert_eq!(file_sha256("block_header_v2.json"), "ed496bcaf60d212fce20a795d8d57e538fe3a8713ca502e7392e25f027576837");
    assert_eq!(file_sha256("mainnet-activation.txt"), "1ba97e30f147eb78c5bb985122dd3dcb32278ecd32cce2a5cdce955c9c10b628");
}
```

- [ ] **Step 7: Run the test**

Run: `cd core && cargo test -p wren-engine --test vectors_test`
Expected: PASS (the files were fetched in Step 5). If run before Step 5, it fails with "vector file present".

- [ ] **Step 8: Write `core/scripts/check.sh`**

```bash
#!/usr/bin/env bash
# Local + CI gate for the engine. Fails on the first error.
set -euo pipefail
cd "$(dirname "$0")/.."
cargo fmt --all -- --check
cargo clippy --all-targets -- -D warnings
cargo test --workspace
if command -v cargo-audit >/dev/null; then cargo audit; else echo "cargo-audit not installed: run 'cargo install cargo-audit --locked'"; exit 1; fi
```
Run: `chmod +x core/scripts/check.sh && cargo install cargo-audit --locked && core/scripts/check.sh`
Expected: all steps pass.

- [ ] **Step 9: Commit**

```bash
git add CLAUDE.md core/
git commit -m "feat(engine): workspace, guardrails, and pinned fork test vectors"
```

---

### Task 2: Hash primitives

**Files:**
- Create: `core/wren-engine/src/hashes.rs`
- Modify: `core/wren-engine/src/lib.rs` (add `pub mod hashes;`)
- Test: inline `#[cfg(test)] mod tests` in `hashes.rs`

**Interfaces:**
- Consumes: nothing.
- Produces:
  - `pub fn sha256(data: &[u8]) -> [u8; 32]`
  - `pub fn sha256d(data: &[u8]) -> [u8; 32]`
  - `pub fn tagged_sha256(tag: &str, msg: &[u8]) -> [u8; 32]` (BIP340: `SHA256(SHA256(tag) ‖ SHA256(tag) ‖ msg)`)
  - `pub fn blake2b256(data: &[u8]) -> [u8; 32]` (unkeyed BLAKE2b, 32-byte digest, as Knots `blake2b_nokey`)

- [ ] **Step 1: Write the failing tests**

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn blake2b256_matches_rfc7693_style_known_answer() {
        // BLAKE2b-256("") — from the reference implementation (b2sum -l 256 of empty input).
        assert_eq!(
            hex::encode(blake2b256(b"")),
            "0e5751c026e543b2e8ab2eb06099daa1d1e5df47778f7787faab45cdf12fe3a8"
        );
    }

    #[test]
    fn tagged_sha256_matches_bip340_tapleaf_style() {
        // tagged_sha256 must equal SHA256(SHA256(tag)||SHA256(tag)||msg); check by construction.
        let t = sha256(b"TapLeaf");
        let mut pre = Vec::new();
        pre.extend_from_slice(&t);
        pre.extend_from_slice(&t);
        pre.extend_from_slice(b"abc");
        assert_eq!(tagged_sha256("TapLeaf", b"abc"), sha256(&pre));
    }

    #[test]
    fn sha256d_is_double_sha256() {
        assert_eq!(sha256d(b"abc"), sha256(&sha256(b"abc")));
    }
}
```

- [ ] **Step 2: Run to verify failure**

Run: `cargo test -p wren-engine hashes`
Expected: FAIL to compile — `blake2b256` / `tagged_sha256` not found.

- [ ] **Step 3: Implement**

```rust
//! Hash primitives used by the fork's header hash and sighash.
use bdk_wallet::bitcoin::hashes::{sha256 as bsha256, Hash, HashEngine};
use blake2::{digest::consts::U32, Blake2b, Digest};

pub fn sha256(data: &[u8]) -> [u8; 32] {
    bsha256::Hash::hash(data).to_byte_array()
}

pub fn sha256d(data: &[u8]) -> [u8; 32] {
    sha256(&sha256(data))
}

pub fn tagged_sha256(tag: &str, msg: &[u8]) -> [u8; 32] {
    let t = sha256(tag.as_bytes());
    let mut e = bsha256::Hash::engine();
    e.input(&t);
    e.input(&t);
    e.input(msg);
    bsha256::Hash::from_engine(e).to_byte_array()
}

pub fn blake2b256(data: &[u8]) -> [u8; 32] {
    let out = Blake2b::<U32>::digest(data);
    let mut r = [0u8; 32];
    r.copy_from_slice(&out);
    r
}
```
Add `pub mod hashes;` to `lib.rs`.

- [ ] **Step 4: Run tests**

Run: `cargo test -p wren-engine hashes`
Expected: PASS (3 tests).

- [ ] **Step 5: Commit**

```bash
git add core/wren-engine/src/hashes.rs core/wren-engine/src/lib.rs
git commit -m "feat(engine): sha256, tagged hash, blake2b-256 primitives"
```

---

### Task 3: ForkHeader parse and serialize (80 and 164 bytes)

**Files:**
- Create: `core/wren-engine/src/header.rs`
- Modify: `core/wren-engine/src/lib.rs` (add `pub mod header;`)
- Test: inline tests in `header.rs`

**Interfaces:**
- Consumes: `EngineError`.
- Produces:
```rust
pub const HEADER_V1_LEN: usize = 80;
pub const HEADER_V2_LEN: usize = 164;
pub const VERSION_HEADER_V2_FLAG: u32 = 0x8000_0000;
pub const FLAG_USE_TIME_OFFSET: u8 = 4;

#[derive(Clone, Debug, PartialEq, Eq)]
pub struct ForkHeader {
    pub version: u32,          // complete version word as on the wire (v2 flag included)
    pub prev_hash: [u8; 32],   // internal (wire) byte order
    pub merkle_root: [u8; 32], // internal (wire) byte order
    pub time_on_wire: u32,
    pub bits: u32,
    pub nonce: u32,
    pub v2: Option<HeaderV2>,
}

#[derive(Clone, Debug, PartialEq, Eq)]
pub struct HeaderV2 {
    pub nonce2: u32, pub nonce3: u32, pub extranonce: [u8; 16], pub time_offset: u32,
    pub txcount: u16, pub flags: u8, pub xor_key_mask_clear_bits: u8, pub xor_key: [u8; 16],
    pub height: i32, pub mm_rhs: [u8; 32],
}

impl ForkHeader {
    pub fn parse(raw: &[u8]) -> Result<ForkHeader, EngineError>;
    pub fn serialize(&self) -> Vec<u8>;
    pub fn is_v2(&self) -> bool;
    /// Effective block time (nTime): time_on_wire + time_offset (wrapping) when flags & 4.
    pub fn time(&self) -> u32;
    /// Declared length of a header that starts with this version word: 164 if the v2 bit is set.
    pub fn len_for_version(version: u32) -> usize;
}
```

- [ ] **Step 1: Write the failing tests**

```rust
#[cfg(test)]
mod tests {
    use super::*;

    fn activation_lines() -> Vec<Vec<u8>> {
        let path = concat!(env!("CARGO_MANIFEST_DIR"), "/tests/vectors/mainnet-activation.txt");
        std::fs::read_to_string(path).unwrap().split_whitespace().map(|l| hex::decode(l).unwrap()).collect()
    }

    #[test]
    fn parses_and_reserializes_every_activation_header() {
        let lines = activation_lines();
        assert_eq!(lines.len(), 10);
        for (i, raw) in lines.iter().enumerate() {
            let h = ForkHeader::parse(raw).unwrap();
            assert_eq!(h.is_v2(), i >= 8, "line {i}");
            assert_eq!(&h.serialize(), raw, "round trip line {i}");
        }
        let h961640 = ForkHeader::parse(&lines[8]).unwrap();
        assert_eq!(h961640.v2.as_ref().unwrap().height, 961_640);
    }

    #[test]
    fn rejects_wrong_lengths() {
        let lines = activation_lines();
        let mut v1 = lines[0].clone();
        v1.push(0);
        assert!(ForkHeader::parse(&v1).is_err());
        let v2_truncated = &lines[8][..163];
        assert!(ForkHeader::parse(v2_truncated).is_err());
        // A v1-length buffer whose version word claims v2 is rejected.
        let mut fake = lines[0].clone();
        fake[3] |= 0x80;
        assert!(ForkHeader::parse(&fake).is_err());
    }

    #[test]
    fn time_applies_offset_only_with_flag() {
        let path = concat!(env!("CARGO_MANIFEST_DIR"), "/tests/vectors/block_header_v2.json");
        let v: serde_json::Value = serde_json::from_str(&std::fs::read_to_string(path).unwrap()).unwrap();
        for case in v["headers"].as_array().unwrap() {
            let raw = hex::decode(case["serialized"].as_str().unwrap()).unwrap();
            let h = ForkHeader::parse(&raw).unwrap();
            let want = case["fields"]["nTime"].as_u64().unwrap() as u32;
            assert_eq!(h.time(), want, "{}", case["name"]);
        }
    }
}
```

- [ ] **Step 2: Run to verify failure**

Run: `cargo test -p wren-engine header`
Expected: FAIL to compile — `ForkHeader` not found.

- [ ] **Step 3: Implement**

```rust
//! The fork's block header: 80-byte SHA256d headers below 961,640, 164-byte v2 headers at and above.
//! Layout source: bitcoinknots/bitcoin v29.4.2.knots20260508 src/primitives/block.h:103-125.
use crate::error::EngineError;

pub const HEADER_V1_LEN: usize = 80;
pub const HEADER_V2_LEN: usize = 164;
pub const VERSION_HEADER_V2_FLAG: u32 = 0x8000_0000;
pub const FLAG_USE_TIME_OFFSET: u8 = 4;

#[derive(Clone, Debug, PartialEq, Eq)]
pub struct ForkHeader {
    pub version: u32,
    pub prev_hash: [u8; 32],
    pub merkle_root: [u8; 32],
    pub time_on_wire: u32,
    pub bits: u32,
    pub nonce: u32,
    pub v2: Option<HeaderV2>,
}

#[derive(Clone, Debug, PartialEq, Eq)]
pub struct HeaderV2 {
    pub nonce2: u32,
    pub nonce3: u32,
    pub extranonce: [u8; 16],
    pub time_offset: u32,
    pub txcount: u16,
    pub flags: u8,
    pub xor_key_mask_clear_bits: u8,
    pub xor_key: [u8; 16],
    pub height: i32,
    pub mm_rhs: [u8; 32],
}

fn u32_at(b: &[u8], o: usize) -> u32 {
    u32::from_le_bytes(b[o..o + 4].try_into().unwrap())
}
fn arr<const N: usize>(b: &[u8], o: usize) -> [u8; N] {
    b[o..o + N].try_into().unwrap()
}

impl ForkHeader {
    pub fn len_for_version(version: u32) -> usize {
        if version & VERSION_HEADER_V2_FLAG != 0 { HEADER_V2_LEN } else { HEADER_V1_LEN }
    }

    pub fn parse(raw: &[u8]) -> Result<ForkHeader, EngineError> {
        if raw.len() < 4 {
            return Err(EngineError::Header("too short"));
        }
        let version = u32_at(raw, 0);
        if raw.len() != Self::len_for_version(version) {
            return Err(EngineError::Header("length does not match version flag"));
        }
        let v2 = if version & VERSION_HEADER_V2_FLAG != 0 {
            Some(HeaderV2 {
                nonce2: u32_at(raw, 80),
                nonce3: u32_at(raw, 84),
                extranonce: arr(raw, 88),
                time_offset: u32_at(raw, 104),
                txcount: u16::from_le_bytes(arr(raw, 108)),
                flags: raw[110],
                xor_key_mask_clear_bits: raw[111],
                xor_key: arr(raw, 112),
                height: i32::from_le_bytes(arr(raw, 128)),
                mm_rhs: arr(raw, 132),
            })
        } else {
            None
        };
        Ok(ForkHeader {
            version,
            prev_hash: arr(raw, 4),
            merkle_root: arr(raw, 36),
            time_on_wire: u32_at(raw, 68),
            bits: u32_at(raw, 72),
            nonce: u32_at(raw, 76),
            v2,
        })
    }

    pub fn serialize(&self) -> Vec<u8> {
        let mut out = Vec::with_capacity(HEADER_V2_LEN);
        out.extend_from_slice(&self.version.to_le_bytes());
        out.extend_from_slice(&self.prev_hash);
        out.extend_from_slice(&self.merkle_root);
        out.extend_from_slice(&self.time_on_wire.to_le_bytes());
        out.extend_from_slice(&self.bits.to_le_bytes());
        out.extend_from_slice(&self.nonce.to_le_bytes());
        if let Some(v) = &self.v2 {
            out.extend_from_slice(&v.nonce2.to_le_bytes());
            out.extend_from_slice(&v.nonce3.to_le_bytes());
            out.extend_from_slice(&v.extranonce);
            out.extend_from_slice(&v.time_offset.to_le_bytes());
            out.extend_from_slice(&v.txcount.to_le_bytes());
            out.push(v.flags);
            out.push(v.xor_key_mask_clear_bits);
            out.extend_from_slice(&v.xor_key);
            out.extend_from_slice(&v.height.to_le_bytes());
            out.extend_from_slice(&v.mm_rhs);
        }
        out
    }

    pub fn is_v2(&self) -> bool {
        self.v2.is_some()
    }

    pub fn time(&self) -> u32 {
        match &self.v2 {
            Some(v) if v.flags & FLAG_USE_TIME_OFFSET != 0 => self.time_on_wire.wrapping_add(v.time_offset),
            _ => self.time_on_wire,
        }
    }
}
```
Add `pub mod header;` to `lib.rs`.

- [ ] **Step 4: Run tests**

Run: `cargo test -p wren-engine header`
Expected: PASS (3 tests).

- [ ] **Step 5: Commit**

```bash
git add core/wren-engine/src/header.rs core/wren-engine/src/lib.rs
git commit -m "feat(engine): parse and serialize 80/164-byte fork headers"
```

---

### Task 4: Block hash (SHA256d for v1, BLAKE2b pipeline for v2)

**Files:**
- Modify: `core/wren-engine/src/header.rs` (add `block_hash`, `BlockHash` helpers)
- Test: inline tests in `header.rs`

**Interfaces:**
- Consumes: `hashes::{sha256d, tagged_sha256, blake2b256}`, `ForkHeader` (Task 3).
- Produces:
  - `impl ForkHeader { pub fn block_hash(&self) -> [u8; 32] }` — returns the hash in **internal (wire) byte order**, i.e. the order that appears in the next header's `prev_hash`.
  - `pub fn display_hex(internal: &[u8; 32]) -> String` — reversed hex, as explorers show it.

This is a byte-exact port of `CBlockHeader::GetHash()` (Knots `src/primitives/block.cpp:13-105`). It was verified during planning with a throwaway reference script against all five drongo vectors (including every intermediate: `xor_key_hash`, `h1`, `h2`, `blake2b_1`, `blake2b_2`, `mask`) and against the real headers 961,632..961,641. Two details that are easy to get wrong: the `zeros` constant in Knots is a `uint128` (16 bytes), so `zeros << zeros` writes **32** bytes; and the XOR mask's first `clear_bits / 8` bytes are zeroed, then byte `[clear_bits / 8]` is ANDed with `0xff >> (clear_bits % 8)`.

- [ ] **Step 1: Write the failing tests**

```rust
    #[test]
    fn v2_hash_matches_every_drongo_vector() {
        let path = concat!(env!("CARGO_MANIFEST_DIR"), "/tests/vectors/block_header_v2.json");
        let v: serde_json::Value = serde_json::from_str(&std::fs::read_to_string(path).unwrap()).unwrap();
        for case in v["headers"].as_array().unwrap() {
            let raw = hex::decode(case["serialized"].as_str().unwrap()).unwrap();
            let h = ForkHeader::parse(&raw).unwrap();
            // drongo's "block_hash" is the display order (reversed from internal).
            assert_eq!(display_hex(&h.block_hash()), case["block_hash"].as_str().unwrap(), "{}", case["name"]);
        }
    }

    #[test]
    fn activation_headers_hash_to_knots_checkpoints_and_link() {
        let lines = activation_lines();
        let hashes: Vec<[u8; 32]> = lines.iter().map(|r| ForkHeader::parse(r).unwrap().block_hash()).collect();
        assert_eq!(display_hex(&hashes[0]), "0000000000000000000169eb6f811ddbd0daf343af7b62180cdb13e7c78dbc16"); // 961,632
        assert_eq!(display_hex(&hashes[7]), "00000000000000000001bbc439e13f749dca850d32c7a2834165338713027e65"); // 961,639
        assert_eq!(display_hex(&hashes[8]), "0000000000000050c1e5f69672f459293be14f46e5a494e7a8c8541396f18eeb"); // 961,640
        assert_eq!(display_hex(&hashes[9]), "0000000000000010ef13157db08c138ea82aa1ac0ec360bdb9f101ce3ed7f7b6"); // 961,641
        for i in 1..lines.len() {
            let h = ForkHeader::parse(&lines[i]).unwrap();
            assert_eq!(h.prev_hash, hashes[i - 1], "link at line {i}");
        }
    }
```

- [ ] **Step 2: Run to verify failure**

Run: `cargo test -p wren-engine header`
Expected: FAIL to compile — `block_hash` / `display_hex` not found.

- [ ] **Step 3: Implement**

Add to `header.rs`:
```rust
use crate::hashes::{blake2b256, sha256d, tagged_sha256};

pub fn display_hex(internal: &[u8; 32]) -> String {
    let mut r = *internal;
    r.reverse();
    hex::encode(r)
}

impl ForkHeader {
    /// Block hash in internal byte order. Port of Knots CBlockHeader::GetHash().
    pub fn block_hash(&self) -> [u8; 32] {
        let v = match &self.v2 {
            None => return sha256d(&self.serialize()),
            Some(v) => v,
        };

        let xor_key_hash = tagged_sha256("Bitcoin block hash PoW XOR key", &v.xor_key);
        let mut mask = [0u8; 32];
        if v.xor_key != [0u8; 16] {
            mask = tagged_sha256("Bitcoin block hash PoW XOR mask", &v.xor_key);
            let clear_bytes = (v.xor_key_mask_clear_bits / 8) as usize;
            for b in mask.iter_mut().take(clear_bytes) {
                *b = 0;
            }
            mask[clear_bytes] &= 0xffu8 >> (v.xor_key_mask_clear_bits % 8);
        }

        let mut prev_sane = self.prev_hash;
        prev_sane.reverse();
        let mut prev_hidden = tagged_sha256("Bitcoin prevblock header, hashed", &prev_sane);

        let mut h1 = Vec::with_capacity(119);
        h1.extend_from_slice(&self.version.to_le_bytes()); // GetCompleteVersion(): wire word incl. v2 flag
        h1.extend_from_slice(&prev_sane);
        h1.extend_from_slice(&v.height.to_le_bytes());
        h1.extend_from_slice(&self.merkle_root);
        h1.extend_from_slice(&self.time_on_wire.to_le_bytes());
        h1.push(0); // reserved for 40-bit time
        h1.extend_from_slice(&self.bits.to_le_bytes());
        h1.extend_from_slice(&(v.txcount as u32).to_le_bytes());
        h1.push(v.flags);
        h1.push(v.xor_key_mask_clear_bits);
        h1.extend_from_slice(&xor_key_hash);
        debug_assert_eq!(h1.len(), 119);
        let h1 = tagged_sha256("Bitcoin block header 1", &h1);

        let mut h2 = Vec::with_capacity(96);
        h2.extend_from_slice(&h1);
        h2.extend_from_slice(&[0u8; 32]); // two uint128 zeros
        h2.extend_from_slice(&v.mm_rhs);
        let h2 = tagged_sha256("Merge-mining hook", &h2);

        let mut s = Vec::with_capacity(52);
        s.extend_from_slice(&0u32.to_le_bytes());
        s.extend_from_slice(&h2);
        s.extend_from_slice(&v.extranonce);
        let first = blake2b256(&s);

        let n = self.nonce.to_le_bytes();
        let n2 = v.nonce2.to_le_bytes();
        let n3 = v.nonce3.to_le_bytes();
        let to = v.time_offset.to_le_bytes();
        let mut s = Vec::with_capacity(160);
        match v.flags & 3 {
            3 | 2 => {
                if v.flags & 3 == 3 {
                    s.extend_from_slice(&[0u8; 32]); // zeros << zeros
                }
                s.extend_from_slice(&[0u8; 48]); // zeros << zeros << zeros
                s.extend_from_slice(&h2);
                s.extend_from_slice(&n);
                s.extend_from_slice(&n2);
                s.extend_from_slice(&to);
                s.extend_from_slice(&n3);
                s.extend_from_slice(&first);
            }
            0 => {
                prev_hidden[..6].fill(0);
                s.extend_from_slice(&prev_hidden);
                s.extend_from_slice(&n);
                s.extend_from_slice(&n2);
                s.extend_from_slice(&to);
                s.extend_from_slice(&n3);
                s.extend_from_slice(&first);
            }
            _ => {
                s.extend_from_slice(&n);
                s.extend_from_slice(&n2);
                s.extend_from_slice(&n3);
                s.extend_from_slice(&to);
                s.extend_from_slice(&first);
                s.extend_from_slice(&h2);
            }
        }
        let second = blake2b256(&s);

        let mut out = [0u8; 32];
        for i in 0..32 {
            out[31 - i] = second[i] ^ mask[i];
        }
        out
    }
}
```

- [ ] **Step 4: Run tests**

Run: `cargo test -p wren-engine header`
Expected: PASS (5 tests).

- [ ] **Step 5: Commit**

```bash
git add core/wren-engine/src/header.rs
git commit -m "feat(engine): fork block hash (SHA256d v1, BLAKE2b v2) matching Knots"
```

---

### Task 5: Proof of work and difficulty

**Files:**
- Create: `core/wren-engine/src/pow.rs`
- Modify: `core/wren-engine/src/lib.rs` (add `pub mod pow;`)
- Test: inline tests in `pow.rs`

**Interfaces:**
- Consumes: `ForkHeader::block_hash` (Task 4).
- Produces:
```rust
pub const RETARGET_INTERVAL: u32 = 2016;
pub const TARGET_TIMESPAN: i64 = 14 * 24 * 60 * 60;
pub const BLAKE2B_TARGET_SHIFT: u32 = 22;
pub fn pow_limit() -> U256;                          // 0x00000000ffff…ff
pub fn regtest_pow_limit() -> U256;                  // 0x7fff…ff
pub fn target_from_compact(bits: u32) -> Option<U256>; // None if negative/zero/overflow/above mainnet limit
pub fn target_from_compact_limit(bits: u32, limit: U256) -> Option<U256>;
pub fn compact_from_target(t: U256) -> u32;
pub fn check_pow(hash_internal: &[u8; 32], bits: u32) -> bool;          // mainnet limit
pub fn check_pow_limit(hash_internal: &[u8; 32], bits: u32, limit: U256) -> bool;
/// Knots GetNextWorkRequired for mainnet/regtest-without-min-difficulty.
/// `prev_bits`: bits of the previous block; `height`: height of the block being checked;
/// `period_first_time` and `prev_time`: needed only when height % 2016 == 0.
pub fn next_bits(height: u32, prev_bits: u32, period_first_time: i64, prev_time: i64, blake2b_height: u32, no_retarget: bool) -> u32;
```
`U256` is `bdk_wallet::bitcoin::pow::Target`'s underlying arithmetic is not public in 0.32, so this task implements a minimal 256-bit unsigned type (`[u64; 4]`, little-endian limbs) with `shl`, `shr`, `mul_u64`, `div_u64`, `cmp`. Only the operations Knots uses are needed.

Ported from Knots `src/pow.cpp` (`GetNextWorkRequired`, `CalculateNextWorkRequired`, `ApplyBlake2bTargetShift`, `DeriveTarget`, `CheckProofOfWorkImpl`) and `arith_uint256::SetCompact/GetCompact`.

- [ ] **Step 1: Write the failing tests**

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::header::ForkHeader;

    fn activation() -> Vec<ForkHeader> {
        let path = concat!(env!("CARGO_MANIFEST_DIR"), "/tests/vectors/mainnet-activation.txt");
        std::fs::read_to_string(path).unwrap().split_whitespace()
            .map(|l| ForkHeader::parse(&hex::decode(l).unwrap()).unwrap()).collect()
    }

    #[test]
    fn compact_round_trips_real_bits() {
        for bits in [0x1702353du32, 0x1a008d4f, 0x1d00ffff] {
            assert_eq!(compact_from_target(target_from_compact(bits).unwrap()), bits);
        }
    }

    #[test]
    fn target_rejects_invalid_compacts() {
        assert!(target_from_compact(0).is_none());          // zero
        assert!(target_from_compact(0x04923456).is_none());  // negative (sign bit)
        assert!(target_from_compact(0x2100ffff).is_none());  // above pow limit
        assert!(target_from_compact(0xff123456).is_none());  // overflow
    }

    #[test]
    fn every_activation_header_meets_its_own_target() {
        for h in activation() {
            assert!(check_pow(&h.block_hash(), h.bits));
        }
    }

    #[test]
    fn blake2b_shift_turns_1702353d_into_1a008d4f_at_961640() {
        // Height 961,640 is not a retarget height (961,640 % 2016 = 8), so only the shift applies.
        assert_eq!(next_bits(961_640, 0x1702353d, 0, 0, 961_640, false), 0x1a008d4f);
        // The next block keeps it.
        assert_eq!(next_bits(961_641, 0x1a008d4f, 0, 0, 961_640, false), 0x1a008d4f);
    }

    #[test]
    fn retarget_clamps_to_four_times() {
        let bits = 0x1a008d4f;
        // Took 1/10 of the timespan: clamps to /4 (4x harder).
        let fast = next_bits(963_648, bits, 0, TARGET_TIMESPAN / 10, 961_640, false);
        let expect_fast = compact_from_target(target_from_compact(bits).unwrap().div_u64(4));
        assert_eq!(fast, expect_fast);
        // Took 10x the timespan: clamps to x4 (easier), capped at pow limit.
        let slow = next_bits(963_648, bits, 0, TARGET_TIMESPAN * 10, 961_640, false);
        let expect_slow = compact_from_target(target_from_compact(bits).unwrap().mul_u64(4).min(pow_limit()));
        assert_eq!(slow, expect_slow);
    }
}
```

- [ ] **Step 2: Run to verify failure**

Run: `cargo test -p wren-engine pow`
Expected: FAIL to compile — module `pow` missing.

- [ ] **Step 3: Implement**

```rust
//! Proof of work and difficulty. Port of Knots v29.4.2 src/pow.cpp + arith_uint256 compact encoding.
use crate::header::ForkHeader;
use std::cmp::Ordering;

pub const RETARGET_INTERVAL: u32 = 2016;
pub const TARGET_TIMESPAN: i64 = 14 * 24 * 60 * 60;
pub const BLAKE2B_TARGET_SHIFT: u32 = 22;

/// Minimal unsigned 256-bit integer, little-endian u64 limbs.
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub struct U256(pub [u64; 4]);

impl U256 {
    pub const ZERO: U256 = U256([0; 4]);
    pub fn from_le_bytes(b: &[u8; 32]) -> U256 {
        let mut l = [0u64; 4];
        for (i, limb) in l.iter_mut().enumerate() {
            *limb = u64::from_le_bytes(b[i * 8..i * 8 + 8].try_into().unwrap());
        }
        U256(l)
    }
    pub fn is_zero(&self) -> bool {
        self.0 == [0; 4]
    }
    pub fn bits(&self) -> u32 {
        for i in (0..4).rev() {
            if self.0[i] != 0 {
                return 64 * i as u32 + (64 - self.0[i].leading_zeros());
            }
        }
        0
    }
    pub fn shl(&self, n: u32) -> U256 {
        let mut r = [0u64; 4];
        let (w, b) = ((n / 64) as usize, n % 64);
        for i in (0..4).rev() {
            if i >= w {
                r[i] = self.0[i - w] << b;
                if b > 0 && i > w {
                    r[i] |= self.0[i - w - 1] >> (64 - b);
                }
            }
        }
        U256(r)
    }
    pub fn shr(&self, n: u32) -> U256 {
        let mut r = [0u64; 4];
        let (w, b) = ((n / 64) as usize, n % 64);
        for i in 0..4 {
            if i + w < 4 {
                r[i] = self.0[i + w] >> b;
                if b > 0 && i + w + 1 < 4 {
                    r[i] |= self.0[i + w + 1] << (64 - b);
                }
            }
        }
        U256(r)
    }
    /// Multiply by a u64, saturating is not needed: callers keep results below 2^256 (Knots semantics
    /// wrap; targets here are <= 2^224 * 4).
    pub fn mul_u64(&self, m: u64) -> U256 {
        let mut r = [0u64; 4];
        let mut carry: u128 = 0;
        for i in 0..4 {
            let v = self.0[i] as u128 * m as u128 + carry;
            r[i] = v as u64;
            carry = v >> 64;
        }
        U256(r)
    }
    pub fn div_u64(&self, d: u64) -> U256 {
        let mut r = [0u64; 4];
        let mut rem: u128 = 0;
        for i in (0..4).rev() {
            let cur = (rem << 64) | self.0[i] as u128;
            r[i] = (cur / d as u128) as u64;
            rem = cur % d as u128;
        }
        U256(r)
    }
    pub fn min(self, o: U256) -> U256 {
        if self <= o { self } else { o }
    }
}

impl PartialOrd for U256 {
    fn partial_cmp(&self, o: &Self) -> Option<Ordering> {
        Some(self.cmp(o))
    }
}
impl Ord for U256 {
    fn cmp(&self, o: &Self) -> Ordering {
        for i in (0..4).rev() {
            match self.0[i].cmp(&o.0[i]) {
                Ordering::Equal => continue,
                x => return x,
            }
        }
        Ordering::Equal
    }
}

pub fn pow_limit() -> U256 {
    // 00000000ffffffffffffffffffffffffffffffffffffffffffffffffffffffff
    U256([u64::MAX, u64::MAX, u64::MAX, 0x0000_0000_ffff_ffff])
}

/// arith_uint256::SetCompact, returning (value, negative, overflow).
fn set_compact(bits: u32) -> (U256, bool, bool) {
    let size = bits >> 24;
    let word = bits & 0x007f_ffff;
    let value = if size <= 3 {
        U256([(word >> (8 * (3 - size))) as u64, 0, 0, 0])
    } else {
        U256([word as u64, 0, 0, 0]).shl(8 * (size - 3))
    };
    let negative = word != 0 && (bits & 0x0080_0000) != 0;
    let overflow = word != 0 && (size > 34 || (word > 0xff && size > 33) || (word > 0xffff && size > 32));
    (value, negative, overflow)
}

pub fn compact_from_target(t: U256) -> u32 {
    let mut size = (t.bits() + 7) / 8;
    let mut compact: u32 = if size <= 3 {
        (t.0[0] << (8 * (3 - size))) as u32
    } else {
        t.shr(8 * (size - 3)).0[0] as u32
    };
    if compact & 0x0080_0000 != 0 {
        compact >>= 8;
        size += 1;
    }
    compact | (size << 24)
}

/// Knots DeriveTarget: None if negative, zero, overflowing, or above `limit`.
pub fn target_from_compact_limit(bits: u32, limit: U256) -> Option<U256> {
    let (t, neg, ovf) = set_compact(bits);
    if neg || ovf || t.is_zero() || t > limit {
        return None;
    }
    Some(t)
}

pub fn target_from_compact(bits: u32) -> Option<U256> {
    target_from_compact_limit(bits, pow_limit())
}

/// hash (internal byte order, i.e. little-endian number) <= target(bits), with an explicit limit.
pub fn check_pow_limit(hash_internal: &[u8; 32], bits: u32, limit: U256) -> bool {
    match target_from_compact_limit(bits, limit) {
        Some(t) => U256::from_le_bytes(hash_internal) <= t,
        None => false,
    }
}

/// Mainnet PoW check.
pub fn check_pow(hash_internal: &[u8; 32], bits: u32) -> bool {
    check_pow_limit(hash_internal, bits, pow_limit())
}

/// Regtest powLimit 7fffffff…ff (Knots chainparams.cpp:623).
pub fn regtest_pow_limit() -> U256 {
    U256([u64::MAX, u64::MAX, u64::MAX, 0x7fff_ffff_ffff_ffff])
}

fn apply_blake2b_shift(bits: u32) -> u32 {
    let (t, _, _) = set_compact(bits);
    let limit = pow_limit();
    let shifted = if t > limit.shr(BLAKE2B_TARGET_SHIFT) { limit } else { t.shl(BLAKE2B_TARGET_SHIFT) };
    compact_from_target(shifted)
}

pub fn next_bits(height: u32, prev_bits: u32, period_first_time: i64, prev_time: i64, blake2b_height: u32, no_retarget: bool) -> u32 {
    let mut bits = if height % RETARGET_INTERVAL != 0 || no_retarget {
        prev_bits
    } else {
        let mut span = prev_time - period_first_time;
        span = span.clamp(TARGET_TIMESPAN / 4, TARGET_TIMESPAN * 4);
        let (t, _, _) = set_compact(prev_bits);
        let t = t.mul_u64(span as u64).div_u64(TARGET_TIMESPAN as u64).min(pow_limit());
        compact_from_target(t)
    };
    if height == blake2b_height {
        bits = apply_blake2b_shift(bits);
    }
    bits
}

/// Convenience used by headerchain: is `h` a valid successor in difficulty terms?
pub fn expected_bits_for(h_height: u32, prev: &ForkHeader, period_first: Option<&ForkHeader>, blake2b_height: u32, no_retarget: bool) -> Option<u32> {
    if h_height % RETARGET_INTERVAL == 0 && !no_retarget {
        let first = period_first?;
        Some(next_bits(h_height, prev.bits, first.time() as i64, prev.time() as i64, blake2b_height, no_retarget))
    } else {
        Some(next_bits(h_height, prev.bits, 0, 0, blake2b_height, no_retarget))
    }
}
```
Add `pub mod pow;` to `lib.rs`.

- [ ] **Step 4: Run tests**

Run: `cargo test -p wren-engine pow`
Expected: PASS (5 tests).

- [ ] **Step 5: Commit**

```bash
git add core/wren-engine/src/pow.rs core/wren-engine/src/lib.rs
git commit -m "feat(engine): compact targets, PoW check, retarget and BLAKE2b shift"
```

---

### Task 6: Chain params and the verified header chain (includes the wrong-chain guard)

**Files:**
- Create: `core/wren-engine/src/params.rs`, `core/wren-engine/src/headerchain.rs`
- Create: `core/scripts/export_headers.py`, `core/wren-engine/tests/fixtures/README.md`, `core/wren-engine/tests/fixtures/mainnet-961632-963700.hex`
- Modify: `core/wren-engine/src/lib.rs` (add `pub mod params; pub mod headerchain;`)
- Test: inline tests in `headerchain.rs`

**Interfaces:**
- Consumes: `ForkHeader` (Tasks 3–4), `pow::{expected_bits_for, check_pow_limit, pow_limit, regtest_pow_limit, U256}` (Task 5), `EngineError`.
- Produces:
```rust
// params.rs
#[derive(Clone, Debug, PartialEq, Eq)]
pub struct Checkpoint { pub height: u32, pub hash: [u8; 32] } // internal byte order
#[derive(Clone, Debug)]
pub struct ChainParams {
    pub network: bdk_wallet::bitcoin::Network,
    pub split_height: u32,            // 961_632 mainnet
    pub blake2b_height: u32,          // 961_640 mainnet
    pub anchor: Checkpoint,           // first header the engine verifies from (a retarget boundary on mainnet)
    pub checkpoints: Vec<Checkpoint>, // must-match hashes
    pub pow_limit: U256,
    pub enforce_difficulty: bool,     // false on regtest (min-difficulty rules)
    pub long_maturity_start: u32,     // 973_440 mainnet; u32::MAX regtest
    pub long_maturity_end: u32,       // 979_920 mainnet; u32::MAX regtest
}
impl ChainParams {
    pub fn mainnet() -> Self;
    pub fn regtest(blake2b_height: u32) -> Self;
    pub fn with_anchor(self, anchor: Checkpoint) -> Self; // tests and regtest harness
}
pub fn internal_from_display(hex_display: &str) -> [u8; 32];

// headerchain.rs
pub struct HeaderChain { /* private */ }
impl HeaderChain {
    pub fn new(params: ChainParams) -> Self;
    pub fn params(&self) -> &ChainParams;
    pub fn is_empty(&self) -> bool;
    pub fn base_height(&self) -> u32;
    pub fn tip_height(&self) -> Option<u32>;
    pub fn tip_hash(&self) -> Option<[u8; 32]>;
    pub fn hash_at(&self, height: u32) -> Option<[u8; 32]>;
    pub fn header_at(&self, height: u32) -> Option<&ForkHeader>;
    pub fn set_anchor(&mut self, raw: &[u8]) -> Result<(), EngineError>;          // WrongChain on mismatch
    pub fn extend(&mut self, start_height: u32, raws: &[Vec<u8>]) -> Result<(), EngineError>;
    pub fn truncate_above(&mut self, height: u32);                                // never removes the anchor
    pub fn to_bytes(&self) -> Vec<u8>;
    pub fn from_bytes(params: ChainParams, bytes: &[u8]) -> Result<Self, EngineError>; // re-verifies everything
}
```

**The wrong-chain guard lives here.** A legacy-chain server fails in one of two places, whichever comes first: its header at 961,632 does not hash to the pinned anchor, or its header at 961,640 is not a v2 header / does not hash to the pinned checkpoint. Both return `EngineError::WrongChain`. Nothing downstream can send without a `HeaderChain` that reached the tip through these checks (enforced in Tasks 14 and 21).

- [ ] **Step 1: Write `params.rs`**

```rust
//! Chain parameters. Mainnet values: Knots v29.4.2 src/kernel/chainparams.cpp:110-149, 240-242.
use crate::pow::{pow_limit, regtest_pow_limit, U256};
use bdk_wallet::bitcoin::Network;

#[derive(Clone, Debug, PartialEq, Eq)]
pub struct Checkpoint {
    pub height: u32,
    pub hash: [u8; 32],
}

#[derive(Clone, Debug)]
pub struct ChainParams {
    pub network: Network,
    pub split_height: u32,
    pub blake2b_height: u32,
    pub anchor: Checkpoint,
    pub checkpoints: Vec<Checkpoint>,
    pub pow_limit: U256,
    pub enforce_difficulty: bool,
    pub long_maturity_start: u32,
    pub long_maturity_end: u32,
}

pub fn internal_from_display(hex_display: &str) -> [u8; 32] {
    let mut b: [u8; 32] = hex::decode(hex_display).expect("hex").try_into().expect("32 bytes");
    b.reverse();
    b
}

impl ChainParams {
    pub fn mainnet() -> Self {
        let anchor = Checkpoint {
            height: 961_632, // 2016 * 477: a retarget boundary, and the first BIP110 block
            hash: internal_from_display("0000000000000000000169eb6f811ddbd0daf343af7b62180cdb13e7c78dbc16"),
        };
        ChainParams {
            network: Network::Bitcoin,
            split_height: 961_632,
            blake2b_height: 961_640,
            checkpoints: vec![
                anchor.clone(),
                Checkpoint { height: 961_639, hash: internal_from_display("00000000000000000001bbc439e13f749dca850d32c7a2834165338713027e65") },
                Checkpoint { height: 961_640, hash: internal_from_display("0000000000000050c1e5f69672f459293be14f46e5a494e7a8c8541396f18eeb") },
            ],
            anchor,
            pow_limit: pow_limit(),
            enforce_difficulty: true,
            long_maturity_start: 973_440,
            long_maturity_end: 979_920,
        }
    }

    /// Regtest: anchor is the regtest genesis block; BLAKE2b activates at `blake2b_height`
    /// (Knots `-testactivationheight=blake2b@<height>`).
    pub fn regtest(blake2b_height: u32) -> Self {
        ChainParams {
            network: Network::Regtest,
            split_height: blake2b_height,
            blake2b_height,
            anchor: Checkpoint {
                height: 0,
                hash: internal_from_display("0f9188f13cb7b2c71f2a335e3a4fc328bf5beb436012afca590b1a11466e2206"),
            },
            checkpoints: vec![],
            pow_limit: regtest_pow_limit(),
            enforce_difficulty: false,
            long_maturity_start: u32::MAX,
            long_maturity_end: u32::MAX,
        }
    }

    pub fn with_anchor(mut self, anchor: Checkpoint) -> Self {
        self.anchor = anchor;
        self
    }
}
```

- [ ] **Step 2: Export the real-chain fixture**

`core/scripts/export_headers.py`:
```python
#!/usr/bin/env python3
"""Export raw fork headers [START, END] from an Electrum server, one hex header per line.
The engine test re-verifies every header's proof of work from the pinned anchor, so the
fixture is self-authenticating: any fork server may be used.
Usage: export_headers.py HOST PORT START END OUT [--tls]"""
import json, socket, ssl, sys

def main():
    host, port, start, end, out = sys.argv[1], int(sys.argv[2]), int(sys.argv[3]), int(sys.argv[4]), sys.argv[5]
    s = socket.create_connection((host, port), timeout=60)
    if "--tls" in sys.argv:
        s = ssl.create_default_context().wrap_socket(s, server_hostname=host)
    f = s.makefile("rwb")
    rid = 0

    def call(method, params):
        nonlocal rid
        rid += 1
        f.write((json.dumps({"jsonrpc": "2.0", "id": rid, "method": method, "params": params}) + "\n").encode())
        f.flush()
        while True:
            r = json.loads(f.readline())
            if r.get("id") == rid:
                if r.get("error"):
                    raise SystemExit(f"server error: {r['error']}")
                return r["result"]

    call("server.version", ["wren-fixture", ["1.4", "1.8"]])
    lines, h = [], start
    while h <= end:
        r = call("blockchain.block.headers", [h, min(2016, end - h + 1)])
        if "headers" in r:
            raws = [bytes.fromhex(x) for x in r["headers"]]
        else:
            b, raws, i = bytes.fromhex(r["hex"]), [], 0
            while i < len(b):
                n = 164 if int.from_bytes(b[i:i + 4], "little") & 0x80000000 else 80
                raws.append(b[i:i + n])
                i += n
        if not raws:
            raise SystemExit(f"server returned no headers at {h}")
        lines += [x.hex() for x in raws]
        h += len(raws)
    with open(out, "w") as fh:
        fh.write("\n".join(lines) + "\n")
    print(f"wrote {len(lines)} headers {start}..{start + len(lines) - 1} to {out}")

main()
```

Run (any fork Electrum server; during development the owner's own node on the LAN, or `electrum.lazarus-xbt.xyz 50002 --tls`):
```bash
python3 core/scripts/export_headers.py <host> <port> 961632 963700 core/wren-engine/tests/fixtures/mainnet-961632-963700.hex --tls
```
Expected: `wrote 2069 headers 961632..963700 to …`.

`core/wren-engine/tests/fixtures/README.md`:
```markdown
# Fixtures

`mainnet-961632-963700.hex` — raw fork mainnet headers 961,632..963,700, one per line, exported with
`core/scripts/export_headers.py`. Self-authenticating: `headerchain` tests re-verify every header's
link, height field, difficulty (including the 963,648 retarget) and proof of work from the pinned
961,632 anchor, so a dishonest exporter cannot produce a passing file.
Record here: exporting server, date, and the sha256 of the file.
```

- [ ] **Step 3: Write the failing tests**

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::header::ForkHeader;
    use crate::params::{ChainParams, Checkpoint};

    fn lines(path: &str) -> Vec<Vec<u8>> {
        std::fs::read_to_string(path).unwrap().split_whitespace().map(|l| hex::decode(l).unwrap()).collect()
    }
    fn activation() -> Vec<Vec<u8>> {
        lines(concat!(env!("CARGO_MANIFEST_DIR"), "/tests/vectors/mainnet-activation.txt"))
    }

    #[test]
    fn verifies_activation_headers_from_anchor() {
        let raws = activation();
        let mut c = HeaderChain::new(ChainParams::mainnet());
        c.set_anchor(&raws[0]).unwrap();
        c.extend(961_633, &raws[1..]).unwrap();
        assert_eq!(c.tip_height(), Some(961_641));
    }

    #[test]
    fn verifies_real_chain_across_retarget_963648() {
        let raws = lines(concat!(env!("CARGO_MANIFEST_DIR"), "/tests/fixtures/mainnet-961632-963700.hex"));
        let mut c = HeaderChain::new(ChainParams::mainnet());
        c.set_anchor(&raws[0]).unwrap();
        for (i, chunk) in raws[1..].chunks(500).enumerate() {
            c.extend(961_633 + (i * 500) as u32, chunk).unwrap();
        }
        assert_eq!(c.tip_height(), Some(963_700));
        // The retarget block's bits must differ from the pre-retarget bits only via the retarget rule,
        // which extend() already enforced; assert the boundary was actually crossed.
        assert!(c.header_at(963_648).is_some());
    }

    #[test]
    fn wrong_anchor_is_wrong_chain() {
        let raws = activation();
        let mut c = HeaderChain::new(ChainParams::mainnet());
        assert!(matches!(c.set_anchor(&raws[1]), Err(EngineError::WrongChain)));
    }

    #[test]
    fn v1_header_at_activation_is_wrong_chain() {
        let raws = activation();
        let mut c = HeaderChain::new(ChainParams::mainnet());
        c.set_anchor(&raws[0]).unwrap();
        c.extend(961_633, &raws[1..8]).unwrap();
        // Craft a v1 header that links to 961,639 (as a legacy-chain server would serve at 961,640).
        let mut fake = ForkHeader::parse(&raws[7]).unwrap();
        fake.prev_hash = c.tip_hash().unwrap();
        assert!(matches!(c.extend(961_640, &[fake.serialize()]), Err(EngineError::WrongChain)));
    }

    #[test]
    fn rejects_broken_link_bad_pow_bad_height_and_reserved_flags() {
        let raws = activation();
        let base = || {
            let mut c = HeaderChain::new(ChainParams::mainnet());
            c.set_anchor(&raws[0]).unwrap();
            c.extend(961_633, &raws[1..9]).unwrap();
            c
        };
        let good = ForkHeader::parse(&raws[9]).unwrap();

        let mut h = good.clone();
        h.prev_hash[0] ^= 1;
        assert!(base().extend(961_641, &[h.serialize()]).is_err());

        let mut h = good.clone();
        h.nonce ^= 1; // changes the hash; mainnet target makes a pass astronomically unlikely
        assert!(base().extend(961_641, &[h.serialize()]).is_err());

        let mut h = good.clone();
        h.v2.as_mut().unwrap().height = 961_642;
        assert!(base().extend(961_641, &[h.serialize()]).is_err());

        let mut h = good.clone();
        h.v2.as_mut().unwrap().flags |= 0x40;
        assert!(base().extend(961_641, &[h.serialize()]).is_err());

        let mut h = good;
        h.bits = 0x1a008d50;
        assert!(base().extend(961_641, &[h.serialize()]).is_err());
    }

    #[test]
    fn extend_must_start_at_tip_plus_one() {
        let raws = activation();
        let mut c = HeaderChain::new(ChainParams::mainnet());
        c.set_anchor(&raws[0]).unwrap();
        assert!(c.extend(961_634, &raws[2..3]).is_err());
    }

    #[test]
    fn bytes_round_trip_reverifies() {
        let raws = activation();
        let mut c = HeaderChain::new(ChainParams::mainnet());
        c.set_anchor(&raws[0]).unwrap();
        c.extend(961_633, &raws[1..]).unwrap();
        let bytes = c.to_bytes();
        let c2 = HeaderChain::from_bytes(ChainParams::mainnet(), &bytes).unwrap();
        assert_eq!(c2.tip_hash(), c.tip_hash());
        let mut corrupt = bytes.clone();
        let last = corrupt.len() - 40;
        corrupt[last] ^= 1;
        assert!(HeaderChain::from_bytes(ChainParams::mainnet(), &corrupt).is_err());
    }

    #[test]
    fn truncate_never_removes_anchor() {
        let raws = activation();
        let mut c = HeaderChain::new(ChainParams::mainnet());
        c.set_anchor(&raws[0]).unwrap();
        c.extend(961_633, &raws[1..]).unwrap();
        c.truncate_above(100);
        assert_eq!(c.tip_height(), Some(961_632));
        let _ = Checkpoint { height: 0, hash: [0; 32] };
    }
}
```

- [ ] **Step 4: Run to verify failure**

Run: `cargo test -p wren-engine headerchain`
Expected: FAIL to compile — `HeaderChain` missing.

- [ ] **Step 5: Implement `headerchain.rs`**

```rust
//! A verified list of fork headers from the pinned anchor to the tip.
use crate::error::EngineError;
use crate::header::ForkHeader;
use crate::params::ChainParams;
use crate::pow::{check_pow_limit, expected_bits_for};

const MAGIC: &[u8; 8] = b"WRENHDR1";
const MTP_WINDOW: usize = 11;

pub struct HeaderChain {
    params: ChainParams,
    headers: Vec<ForkHeader>,
    hashes: Vec<[u8; 32]>,
}

impl HeaderChain {
    pub fn new(params: ChainParams) -> Self {
        HeaderChain { params, headers: Vec::new(), hashes: Vec::new() }
    }
    pub fn params(&self) -> &ChainParams {
        &self.params
    }
    pub fn is_empty(&self) -> bool {
        self.headers.is_empty()
    }
    pub fn base_height(&self) -> u32 {
        self.params.anchor.height
    }
    pub fn tip_height(&self) -> Option<u32> {
        (!self.headers.is_empty()).then(|| self.base_height() + self.headers.len() as u32 - 1)
    }
    pub fn tip_hash(&self) -> Option<[u8; 32]> {
        self.hashes.last().copied()
    }
    fn idx(&self, height: u32) -> Option<usize> {
        let i = height.checked_sub(self.base_height())? as usize;
        (i < self.headers.len()).then_some(i)
    }
    pub fn hash_at(&self, height: u32) -> Option<[u8; 32]> {
        self.idx(height).map(|i| self.hashes[i])
    }
    pub fn header_at(&self, height: u32) -> Option<&ForkHeader> {
        self.idx(height).map(|i| &self.headers[i])
    }

    pub fn set_anchor(&mut self, raw: &[u8]) -> Result<(), EngineError> {
        let h = ForkHeader::parse(raw)?;
        let hash = h.block_hash();
        if hash != self.params.anchor.hash {
            return Err(EngineError::WrongChain);
        }
        self.headers = vec![h];
        self.hashes = vec![hash];
        Ok(())
    }

    pub fn extend(&mut self, start_height: u32, raws: &[Vec<u8>]) -> Result<(), EngineError> {
        let tip = self.tip_height().ok_or(EngineError::Chain("no anchor"))?;
        if start_height != tip + 1 {
            return Err(EngineError::Chain("headers do not start at tip + 1"));
        }
        for (i, raw) in raws.iter().enumerate() {
            let height = start_height + i as u32;
            let h = ForkHeader::parse(raw)?;
            let hash = h.block_hash();
            self.check_next(&h, &hash, height)?;
            self.headers.push(h);
            self.hashes.push(hash);
        }
        Ok(())
    }

    fn check_next(&self, h: &ForkHeader, hash: &[u8; 32], height: u32) -> Result<(), EngineError> {
        let p = &self.params;
        let prev = self.header_at(height - 1).ok_or(EngineError::Chain("missing previous header"))?;
        if h.prev_hash != self.hash_at(height - 1).unwrap() {
            return Err(EngineError::Chain("header does not link to previous"));
        }
        let should_be_v2 = height >= p.blake2b_height;
        if h.is_v2() != should_be_v2 {
            return Err(if height == p.blake2b_height { EngineError::WrongChain } else { EngineError::Chain("wrong header version for height") });
        }
        if let Some(v) = &h.v2 {
            if v.flags & 0xc0 != 0 {
                return Err(EngineError::Chain("reserved header flags set"));
            }
            if v.height as i64 != height as i64 {
                return Err(EngineError::Chain("header height field mismatch"));
            }
        }
        if p.enforce_difficulty {
            let period_first = if height % crate::pow::RETARGET_INTERVAL == 0 {
                Some(self.header_at(height - crate::pow::RETARGET_INTERVAL).ok_or(EngineError::Chain("missing retarget period start"))?)
            } else {
                None
            };
            let expected = expected_bits_for(height, prev, period_first, p.blake2b_height, false)
                .ok_or(EngineError::Chain("cannot compute difficulty"))?;
            if h.bits != expected {
                return Err(EngineError::Chain("unexpected difficulty"));
            }
        }
        if !check_pow_limit(hash, h.bits, p.pow_limit) {
            return Err(EngineError::Chain("insufficient proof of work"));
        }
        let n = self.headers.len();
        let mut times: Vec<u32> = self.headers[n.saturating_sub(MTP_WINDOW)..].iter().map(|x| x.time()).collect();
        times.sort_unstable();
        if h.time() <= times[times.len() / 2] {
            return Err(EngineError::Chain("time not after median time past"));
        }
        if let Some(cp) = p.checkpoints.iter().find(|c| c.height == height) {
            if cp.hash != *hash {
                return Err(EngineError::WrongChain);
            }
        }
        Ok(())
    }

    pub fn truncate_above(&mut self, height: u32) {
        let keep = (height.max(self.base_height()) - self.base_height() + 1) as usize;
        self.headers.truncate(keep);
        self.hashes.truncate(keep);
    }

    pub fn to_bytes(&self) -> Vec<u8> {
        let mut out = MAGIC.to_vec();
        out.extend_from_slice(&self.base_height().to_le_bytes());
        for h in &self.headers {
            out.extend_from_slice(&h.serialize());
        }
        out
    }

    pub fn from_bytes(params: ChainParams, bytes: &[u8]) -> Result<Self, EngineError> {
        if bytes.len() < 12 || &bytes[..8] != MAGIC {
            return Err(EngineError::Chain("bad header store"));
        }
        let base = u32::from_le_bytes(bytes[8..12].try_into().unwrap());
        if base != params.anchor.height {
            return Err(EngineError::Chain("header store anchor mismatch"));
        }
        let mut raws = Vec::new();
        let mut i = 12;
        while i < bytes.len() {
            if bytes.len() - i < 4 {
                return Err(EngineError::Chain("truncated header store"));
            }
            let v = u32::from_le_bytes(bytes[i..i + 4].try_into().unwrap());
            let n = ForkHeader::len_for_version(v);
            if bytes.len() - i < n {
                return Err(EngineError::Chain("truncated header store"));
            }
            raws.push(bytes[i..i + n].to_vec());
            i += n;
        }
        let mut c = HeaderChain::new(params);
        let (first, rest) = raws.split_first().ok_or(EngineError::Chain("empty header store"))?;
        c.set_anchor(first)?;
        c.extend(base + 1, rest)?;
        Ok(c)
    }
}
```
Add `pub mod params; pub mod headerchain;` to `lib.rs`.

- [ ] **Step 6: Run tests**

Run: `cargo test -p wren-engine headerchain`
Expected: PASS (8 tests).

- [ ] **Step 7: Commit**

```bash
git add core/wren-engine/src/params.rs core/wren-engine/src/headerchain.rs core/wren-engine/src/lib.rs core/scripts/export_headers.py core/wren-engine/tests/fixtures
git commit -m "feat(engine): verified header chain from pinned anchor with wrong-chain guard"
```

---

### Task 7: The SIGHASH_UNIFIED digest

**Files:**
- Create: `core/wren-engine/src/sighash.rs`
- Modify: `core/wren-engine/src/lib.rs` (add `pub mod sighash;`)
- Test: inline tests in `sighash.rs`

**Interfaces:**
- Consumes: `hashes::{sha256, tagged_sha256}`, `EngineError`.
- Produces:
```rust
pub const SIGHASH_ALL: u32 = 0x01;
pub const SIGHASH_NONE: u32 = 0x02;
pub const SIGHASH_SINGLE: u32 = 0x03;
pub const SIGHASH_ANYONECANPAY: u32 = 0x80;
pub const SIGHASH_UNIFIED: u32 = 0x20;
pub const SIGHASH_ALL_UNIFIED: u32 = 0x21;
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum ScriptType { Bare = 0, WitnessV0 = 1, Taproot = 2, Tapscript = 3 }
pub struct TapscriptContext { pub leaf_hash: [u8; 32], pub codesep_pos: u32 }
pub fn tapleaf_hash(script: &Script) -> [u8; 32];
/// Returns the 32-byte message to sign, in the byte order passed to secp256k1 (= vector order).
pub fn unified_sighash(tx: &Transaction, input_index: usize, spent: &[TxOut], hash_type: u32,
    script_type: ScriptType, script_code: &Script, tapscript: Option<&TapscriptContext>) -> Result<[u8; 32], EngineError>;
```

Port of Knots `SignatureHashUnified` (`src/script/interpreter.cpp:1635-1754`). Verified during planning with a throwaway reference script: all 166 Knots vectors match. Wren never attaches an annex (RDTS forbids it), so the taproot annex byte is always `0`.

- [ ] **Step 1: Write the failing test**

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use bdk_wallet::bitcoin::{consensus::deserialize, Amount, ScriptBuf, Transaction, TxOut};

    #[test]
    fn all_166_knots_vectors_match() {
        let path = concat!(env!("CARGO_MANIFEST_DIR"), "/tests/vectors/unified_sighash.json");
        let v: serde_json::Value = serde_json::from_str(&std::fs::read_to_string(path).unwrap()).unwrap();
        let rows = &v.as_array().unwrap()[1..]; // row 0 is the column header
        assert_eq!(rows.len(), 166);
        for (n, row) in rows.iter().enumerate() {
            let script_code = ScriptBuf::from_bytes(hex::decode(row[0].as_str().unwrap()).unwrap());
            let tx: Transaction = deserialize(&hex::decode(row[1].as_str().unwrap()).unwrap()).unwrap();
            let index = row[2].as_u64().unwrap() as usize;
            let hash_type = row[3].as_i64().unwrap() as i32 as u32;
            let st = match row[4].as_u64().unwrap() {
                0 => ScriptType::Bare,
                1 => ScriptType::WitnessV0,
                2 => ScriptType::Taproot,
                _ => ScriptType::Tapscript,
            };
            let spent: Vec<TxOut> = row[5].as_array().unwrap().iter().map(|u| TxOut {
                value: Amount::from_sat(u[0].as_i64().unwrap() as u64),
                script_pubkey: ScriptBuf::from_bytes(hex::decode(u[1].as_str().unwrap()).unwrap()),
            }).collect();
            let ctx = TapscriptContext { leaf_hash: tapleaf_hash(&script_code), codesep_pos: 0xffff_ffff };
            let got = unified_sighash(&tx, index, &spent, hash_type, st, &script_code,
                (st == ScriptType::Tapscript).then_some(&ctx)).unwrap();
            assert_eq!(hex::encode(got), row[6].as_str().unwrap(), "vector {n}");
        }
    }

    #[test]
    fn refuses_hash_type_without_unified_bit() {
        let tx = Transaction {
            version: bdk_wallet::bitcoin::transaction::Version::TWO,
            lock_time: bdk_wallet::bitcoin::absolute::LockTime::ZERO,
            input: vec![Default::default()],
            output: vec![],
        };
        let spent = vec![TxOut { value: Amount::from_sat(1), script_pubkey: ScriptBuf::new() }];
        assert!(unified_sighash(&tx, 0, &spent, SIGHASH_ALL, ScriptType::WitnessV0, &ScriptBuf::new(), None).is_err());
    }

    #[test]
    fn refuses_missing_spent_outputs() {
        let tx = Transaction {
            version: bdk_wallet::bitcoin::transaction::Version::TWO,
            lock_time: bdk_wallet::bitcoin::absolute::LockTime::ZERO,
            input: vec![Default::default(), Default::default()],
            output: vec![],
        };
        let spent = vec![TxOut { value: Amount::from_sat(1), script_pubkey: ScriptBuf::new() }];
        assert!(unified_sighash(&tx, 0, &spent, SIGHASH_ALL_UNIFIED, ScriptType::WitnessV0, &ScriptBuf::new(), None).is_err());
    }
}
```

- [ ] **Step 2: Run to verify failure**

Run: `cargo test -p wren-engine sighash`
Expected: FAIL to compile — module missing.

- [ ] **Step 3: Implement**

```rust
//! SIGHASH_UNIFIED signature message. Port of Knots v29.4.2 SignatureHashUnified.
use crate::error::EngineError;
use crate::hashes::{sha256, tagged_sha256};
use bdk_wallet::bitcoin::{consensus::encode::serialize, Script, Transaction, TxOut};

pub const SIGHASH_ALL: u32 = 0x01;
pub const SIGHASH_NONE: u32 = 0x02;
pub const SIGHASH_SINGLE: u32 = 0x03;
pub const SIGHASH_ANYONECANPAY: u32 = 0x80;
pub const SIGHASH_UNIFIED: u32 = 0x20;
pub const SIGHASH_ALL_UNIFIED: u32 = 0x21;

#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum ScriptType {
    Bare = 0,
    WitnessV0 = 1,
    Taproot = 2,
    Tapscript = 3,
}

pub struct TapscriptContext {
    pub leaf_hash: [u8; 32],
    pub codesep_pos: u32,
}

pub fn tapleaf_hash(script: &Script) -> [u8; 32] {
    let mut m = vec![0xc0u8];
    m.extend(serialize(&script.to_owned()));
    tagged_sha256("TapLeaf", &m)
}

pub fn unified_sighash(
    tx: &Transaction,
    input_index: usize,
    spent: &[TxOut],
    hash_type: u32,
    script_type: ScriptType,
    script_code: &Script,
    tapscript: Option<&TapscriptContext>,
) -> Result<[u8; 32], EngineError> {
    if input_index >= tx.input.len() {
        return Err(EngineError::Signing("input index out of range"));
    }
    if spent.len() != tx.input.len() {
        return Err(EngineError::Signing("every input's spent output is required"));
    }
    if hash_type & SIGHASH_UNIFIED == 0 {
        return Err(EngineError::Signing("hash type lacks the UNIFIED bit"));
    }
    let taproot = matches!(script_type, ScriptType::Taproot | ScriptType::Tapscript);
    let output_type = hash_type & 0x1f;
    if taproot {
        if hash_type & !(0x1f | SIGHASH_ANYONECANPAY | SIGHASH_UNIFIED) != 0 {
            return Err(EngineError::Signing("undefined taproot hash type"));
        }
        if !(output_type == SIGHASH_ALL || output_type == SIGHASH_NONE || output_type == SIGHASH_SINGLE) {
            return Err(EngineError::Signing("undefined taproot hash type"));
        }
    }
    let anyonecanpay = hash_type & SIGHASH_ANYONECANPAY != 0;

    let mut m = Vec::with_capacity(256);
    m.push(0); // epoch
    m.push(hash_type as u8);
    m.extend(tx.version.0.to_le_bytes());
    m.extend(tx.lock_time.to_consensus_u32().to_le_bytes());
    m.push(0); // fifth locktime byte
    if !anyonecanpay {
        let mut prevouts = Vec::new();
        let mut amounts = Vec::new();
        let mut scripts = Vec::new();
        let mut sequences = Vec::new();
        for (txin, out) in tx.input.iter().zip(spent) {
            prevouts.extend(serialize(&txin.previous_output));
            amounts.extend(out.value.to_sat().to_le_bytes());
            scripts.extend(serialize(&out.script_pubkey));
            sequences.extend(txin.sequence.0.to_le_bytes());
        }
        m.extend(sha256(&prevouts));
        m.extend(sha256(&amounts));
        m.extend(sha256(&scripts));
        m.extend(sha256(&sequences));
    }
    if output_type != SIGHASH_NONE && output_type != SIGHASH_SINGLE {
        let mut outs = Vec::new();
        for o in &tx.output {
            outs.extend(serialize(o));
        }
        m.extend(sha256(&outs));
    }
    m.push(script_type as u8);
    if anyonecanpay {
        m.extend(serialize(&tx.input[input_index].previous_output));
        m.extend(serialize(&spent[input_index]));
        m.extend(tx.input[input_index].sequence.0.to_le_bytes());
    } else {
        m.extend((input_index as u32).to_le_bytes());
    }
    if !taproot {
        m.extend(serialize(&script_code.to_owned()));
    } else {
        m.push(0); // no annex
    }
    if output_type == SIGHASH_SINGLE {
        let o = tx.output.get(input_index).ok_or(EngineError::Signing("SINGLE without a matching output"))?;
        m.extend(sha256(&serialize(o)));
    }
    if script_type == ScriptType::Tapscript {
        let t = tapscript.ok_or(EngineError::Signing("tapscript context missing"))?;
        m.extend(t.leaf_hash);
        m.push(0); // key version
        m.extend(t.codesep_pos.to_le_bytes());
    }
    Ok(tagged_sha256("UnifiedSighash", &m))
}
```
Add `pub mod sighash;` to `lib.rs`.

- [ ] **Step 4: Run tests**

Run: `cargo test -p wren-engine sighash`
Expected: PASS (3 tests).

- [ ] **Step 5: Commit**

```bash
git add core/wren-engine/src/sighash.rs core/wren-engine/src/lib.rs
git commit -m "feat(engine): SIGHASH_UNIFIED digest matching all 166 Knots vectors"
```

---

### Task 8: Hot-wallet signing and signature verification

**Files:**
- Create: `core/wren-engine/src/sign.rs`
- Modify: `core/wren-engine/src/lib.rs` (add `pub mod sign;`)
- Test: inline tests in `sign.rs`

**Interfaces:**
- Consumes: `sighash::{unified_sighash, ScriptType, SIGHASH_ALL_UNIFIED}` (Task 7).
- Produces:
```rust
pub fn p2wpkh_script_code(pk: &PublicKey) -> ScriptBuf;
pub fn sign_p2wpkh_unified<C: Signing + Verification>(secp: &Secp256k1<C>, tx: &mut Transaction, index: usize, spent: &[TxOut], sk: &SecretKey) -> Result<(), EngineError>;
pub fn verify_ecdsa_unified<C: Verification>(secp: &Secp256k1<C>, tx: &Transaction, index: usize, spent: &[TxOut], script_type: ScriptType, script_code: &Script, sig_with_type: &[u8], pk: &PublicKey) -> Result<(), EngineError>;
pub fn verify_schnorr_unified<C: Verification>(secp: &Secp256k1<C>, tx: &Transaction, index: usize, spent: &[TxOut], sig: &[u8], output_key: &XOnlyPublicKey) -> Result<(), EngineError>;
```
Signatures use RFC6979 with low-R grinding (`sign_ecdsa_low_r`), the same nonce policy as Knots, so Task 22 can require byte equality with `signrawtransactionwithkey … "ALL|UNIFIED"`.

- [ ] **Step 1: Write the failing tests**

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use bdk_wallet::bitcoin::{absolute::LockTime, transaction::Version, Amount, OutPoint, Sequence, TxIn, Txid};
    use bdk_wallet::bitcoin::hashes::Hash;

    fn fixture() -> (Secp256k1<bdk_wallet::bitcoin::secp256k1::All>, SecretKey, Transaction, Vec<TxOut>) {
        let secp = Secp256k1::new();
        let sk = SecretKey::from_slice(&[7u8; 32]).unwrap();
        let pk = PublicKey::from_secret_key(&secp, &sk);
        let ours = ScriptBuf::new_p2wpkh(&CompressedPublicKey(pk).wpubkey_hash());
        let theirs = ScriptBuf::new_p2wpkh(&CompressedPublicKey(PublicKey::from_secret_key(&secp, &SecretKey::from_slice(&[9u8; 32]).unwrap())).wpubkey_hash());
        let tx = Transaction {
            version: Version::TWO,
            lock_time: LockTime::ZERO,
            input: vec![
                TxIn { previous_output: OutPoint::new(Txid::from_byte_array([1; 32]), 0), sequence: Sequence::ENABLE_RBF_NO_LOCKTIME, ..Default::default() },
                TxIn { previous_output: OutPoint::new(Txid::from_byte_array([2; 32]), 1), sequence: Sequence::ENABLE_RBF_NO_LOCKTIME, ..Default::default() },
            ],
            output: vec![TxOut { value: Amount::from_sat(40_000), script_pubkey: theirs.clone() }],
        };
        let spent = vec![
            TxOut { value: Amount::from_sat(30_000), script_pubkey: ours },
            TxOut { value: Amount::from_sat(20_000), script_pubkey: theirs },
        ];
        (secp, sk, tx, spent)
    }

    #[test]
    fn signs_with_0x21_low_r_and_verifies() {
        let (secp, sk, mut tx, spent) = fixture();
        sign_p2wpkh_unified(&secp, &mut tx, 0, &spent, &sk).unwrap();
        let w: Vec<&[u8]> = tx.input[0].witness.iter().collect();
        assert_eq!(w.len(), 2);
        assert_eq!(*w[0].last().unwrap(), 0x21);
        assert!(w[0].len() <= 71, "low-R signature is at most 70 DER bytes + type");
        let pk = PublicKey::from_slice(w[1]).unwrap();
        verify_ecdsa_unified(&secp, &tx, 0, &spent, ScriptType::WitnessV0, &p2wpkh_script_code(&pk), w[0], &pk).unwrap();
    }

    #[test]
    fn signature_commits_to_every_input_amount() {
        let (secp, sk, mut tx, mut spent) = fixture();
        sign_p2wpkh_unified(&secp, &mut tx, 0, &spent, &sk).unwrap();
        let w: Vec<Vec<u8>> = tx.input[0].witness.iter().map(|x| x.to_vec()).collect();
        let pk = PublicKey::from_slice(&w[1]).unwrap();
        spent[1].value = Amount::from_sat(20_001); // the OTHER input's amount
        assert!(verify_ecdsa_unified(&secp, &tx, 0, &spent, ScriptType::WitnessV0, &p2wpkh_script_code(&pk), &w[0], &pk).is_err());
    }

    #[test]
    fn refuses_key_that_does_not_own_the_input() {
        let (secp, sk, mut tx, spent) = fixture();
        assert!(sign_p2wpkh_unified(&secp, &mut tx, 1, &spent, &sk).is_err());
    }

    #[test]
    fn verify_rejects_standard_sighash_byte() {
        let (secp, sk, mut tx, spent) = fixture();
        sign_p2wpkh_unified(&secp, &mut tx, 0, &spent, &sk).unwrap();
        let mut w: Vec<Vec<u8>> = tx.input[0].witness.iter().map(|x| x.to_vec()).collect();
        *w[0].last_mut().unwrap() = 0x01;
        let pk = PublicKey::from_slice(&w[1]).unwrap();
        assert!(verify_ecdsa_unified(&secp, &tx, 0, &spent, ScriptType::WitnessV0, &p2wpkh_script_code(&pk), &w[0], &pk).is_err());
    }
}
```

- [ ] **Step 2: Run to verify failure**

Run: `cargo test -p wren-engine sign::`
Expected: FAIL to compile — module missing.

- [ ] **Step 3: Implement**

```rust
//! Hot-wallet signing (P2WPKH only — Wren's own wallet) and UNIFIED signature verification
//! (all script types — used for payments and Claim).
use crate::error::EngineError;
use crate::sighash::{unified_sighash, ScriptType, SIGHASH_ALL_UNIFIED};
use bdk_wallet::bitcoin::secp256k1::{ecdsa, schnorr, Message, PublicKey, Secp256k1, SecretKey, Signing, Verification, XOnlyPublicKey};
use bdk_wallet::bitcoin::{CompressedPublicKey, Script, ScriptBuf, Transaction, TxOut, Witness};

pub fn p2wpkh_script_code(pk: &PublicKey) -> ScriptBuf {
    ScriptBuf::new_p2pkh(&CompressedPublicKey(*pk).pubkey_hash())
}

pub fn sign_p2wpkh_unified<C: Signing + Verification>(
    secp: &Secp256k1<C>,
    tx: &mut Transaction,
    index: usize,
    spent: &[TxOut],
    sk: &SecretKey,
) -> Result<(), EngineError> {
    let pk = PublicKey::from_secret_key(secp, sk);
    let prev = spent.get(index).ok_or(EngineError::Signing("missing spent output"))?;
    if prev.script_pubkey != ScriptBuf::new_p2wpkh(&CompressedPublicKey(pk).wpubkey_hash()) {
        return Err(EngineError::Signing("key does not own this input"));
    }
    let digest = unified_sighash(tx, index, spent, SIGHASH_ALL_UNIFIED, ScriptType::WitnessV0, &p2wpkh_script_code(&pk), None)?;
    let sig = secp.sign_ecdsa_low_r(&Message::from_digest(digest), sk);
    let mut sig_bytes = sig.serialize_der().to_vec();
    sig_bytes.push(SIGHASH_ALL_UNIFIED as u8);
    tx.input[index].witness = Witness::from_slice(&[sig_bytes, pk.serialize().to_vec()]);
    Ok(())
}

#[allow(clippy::too_many_arguments)]
pub fn verify_ecdsa_unified<C: Verification>(
    secp: &Secp256k1<C>,
    tx: &Transaction,
    index: usize,
    spent: &[TxOut],
    script_type: ScriptType,
    script_code: &Script,
    sig_with_type: &[u8],
    pk: &PublicKey,
) -> Result<(), EngineError> {
    let (&last, der) = sig_with_type.split_last().ok_or(EngineError::Signing("empty signature"))?;
    if last as u32 != SIGHASH_ALL_UNIFIED {
        return Err(EngineError::Signing("signature is not SIGHASH_ALL|UNIFIED (0x21)"));
    }
    let sig = ecdsa::Signature::from_der(der).map_err(|_| EngineError::Signing("malformed signature"))?;
    let mut normalized = sig;
    normalized.normalize_s();
    if normalized != sig {
        return Err(EngineError::Signing("high-S signature"));
    }
    let digest = unified_sighash(tx, index, spent, SIGHASH_ALL_UNIFIED, script_type, script_code, None)?;
    secp.verify_ecdsa(&Message::from_digest(digest), &sig, pk)
        .map_err(|_| EngineError::Signing("signature does not verify"))
}

pub fn verify_schnorr_unified<C: Verification>(
    secp: &Secp256k1<C>,
    tx: &Transaction,
    index: usize,
    spent: &[TxOut],
    sig: &[u8],
    output_key: &XOnlyPublicKey,
) -> Result<(), EngineError> {
    if sig.len() != 65 || sig[64] as u32 != SIGHASH_ALL_UNIFIED {
        return Err(EngineError::Signing("taproot signature must be 65 bytes ending in 0x21"));
    }
    let s = schnorr::Signature::from_slice(&sig[..64]).map_err(|_| EngineError::Signing("malformed signature"))?;
    let digest = unified_sighash(tx, index, spent, SIGHASH_ALL_UNIFIED, ScriptType::Taproot, Script::new(), None)?;
    secp.verify_schnorr(&s, &Message::from_digest(digest), output_key)
        .map_err(|_| EngineError::Signing("signature does not verify"))
}
```
Add `pub mod sign;` to `lib.rs`.

- [ ] **Step 4: Run tests**

Run: `cargo test -p wren-engine sign::`
Expected: PASS (4 tests).

- [ ] **Step 5: Commit**

```bash
git add core/wren-engine/src/sign.rs core/wren-engine/src/lib.rs
git commit -m "feat(engine): P2WPKH UNIFIED signing (low-R) and UNIFIED signature verification"
```

---

### Task 9: Electrum JSON-RPC client

**Files:**
- Create: `core/wren-engine/src/electrum/mod.rs`, `core/wren-engine/src/electrum/rpc.rs`, `core/wren-engine/src/electrum/mock.rs` (`#[cfg(test)]`)
- Modify: `core/wren-engine/src/lib.rs` (add `pub mod electrum;`)
- Test: inline tests in `electrum/mod.rs`

**Interfaces:**
- Consumes: `hashes::{sha256, sha256d}`, `header::ForkHeader::len_for_version`, `EngineError`.
- Produces:
```rust
// electrum/rpc.rs
pub trait Stream: AsyncRead + AsyncWrite + Unpin + Send {}
pub type BoxStream = Box<dyn Stream>;
pub struct Rpc; // new(BoxStream), call(&str, Value), batch(Vec<(&str, Value)>)
// electrum/mod.rs
pub struct ElectrumClient;
#[derive(Clone, Debug)] pub struct HistoryItem { pub txid: Txid, pub height: i64 } // <=0 = unconfirmed
#[derive(Clone, Debug)] pub struct MerkleProof { pub block_height: u32, pub pos: u32, pub merkle: Vec<[u8; 32]> } // internal order
pub fn script_hash(spk: &Script) -> String;
pub fn split_concatenated_headers(b: &[u8]) -> Result<Vec<Vec<u8>>, EngineError>;
pub fn verify_merkle(txid: &Txid, proof: &MerkleProof, merkle_root_internal: &[u8; 32]) -> bool;
impl ElectrumClient {
    pub async fn connect(stream: BoxStream) -> Result<Self, EngineError>;       // negotiates ["1.4","1.8"]
    pub fn protocol(&self) -> &str;
    pub async fn tip(&self) -> Result<(u32, Vec<u8>), EngineError>;
    pub async fn header(&self, height: u32) -> Result<Vec<u8>, EngineError>;
    pub async fn headers(&self, start: u32, count: u32) -> Result<Vec<Vec<u8>>, EngineError>;
    pub async fn histories(&self, scripts: &[ScriptBuf]) -> Result<Vec<Vec<HistoryItem>>, EngineError>;
    pub async fn transactions(&self, txids: &[Txid]) -> Result<Vec<Transaction>, EngineError>; // txid-checked
    pub async fn merkle_proofs(&self, items: &[(Txid, u32)]) -> Result<Vec<MerkleProof>, EngineError>;
    pub async fn estimate_fee(&self, blocks: u32) -> Result<Option<FeeRate>, EngineError>;
    pub async fn relay_fee(&self) -> Result<FeeRate, EngineError>;
    pub async fn broadcast(&self, tx: &Transaction) -> Result<Txid, EngineError>;
}
// electrum/mock.rs (tests only)
pub fn spawn(handler: impl Fn(&str, &Value) -> Result<Value, String> + Send + Sync + 'static) -> BoxStream;
```

- [ ] **Step 1: Write the mock server (test support)**

`src/electrum/mock.rs`:
```rust
//! In-process Electrum server for tests. Every request line gets one response line; before each
//! response it also emits an unsolicited notification, so clients must skip lines without our id.
use super::rpc::BoxStream;
use serde_json::{json, Value};
use tokio::io::{AsyncBufReadExt, AsyncWriteExt, BufReader};

pub fn spawn(handler: impl Fn(&str, &Value) -> Result<Value, String> + Send + Sync + 'static) -> BoxStream {
    let (client, server) = tokio::io::duplex(1 << 22);
    tokio::spawn(async move {
        let (r, mut w) = tokio::io::split(server);
        let mut lines = BufReader::new(r).lines();
        while let Ok(Some(line)) = lines.next_line().await {
            let req: Value = serde_json::from_str(&line).unwrap();
            let method = req["method"].as_str().unwrap().to_string();
            let note = json!({"jsonrpc": "2.0", "method": "blockchain.headers.subscribe", "params": [{"height": 1, "hex": "00"}]});
            let resp = match handler(&method, &req["params"]) {
                Ok(v) => json!({"jsonrpc": "2.0", "id": req["id"], "result": v}),
                Err(e) => json!({"jsonrpc": "2.0", "id": req["id"], "error": {"code": 1, "message": e}}),
            };
            if w.write_all(format!("{note}\n{resp}\n").as_bytes()).await.is_err() {
                break;
            }
        }
    });
    Box::new(client)
}
```

- [ ] **Step 2: Write the failing tests**

In `src/electrum/mod.rs`:
```rust
#[cfg(test)]
mod tests {
    use super::*;
    use bdk_wallet::bitcoin::{absolute::LockTime, consensus::encode::serialize, transaction::Version, Amount, TxOut};
    use serde_json::json;

    fn activation() -> Vec<Vec<u8>> {
        let p = concat!(env!("CARGO_MANIFEST_DIR"), "/tests/vectors/mainnet-activation.txt");
        std::fs::read_to_string(p).unwrap().split_whitespace().map(|l| hex::decode(l).unwrap()).collect()
    }
    fn tx(v: u64) -> Transaction {
        Transaction { version: Version::TWO, lock_time: LockTime::ZERO, input: vec![Default::default()],
            output: vec![TxOut { value: Amount::from_sat(v), script_pubkey: ScriptBuf::new() }] }
    }

    #[tokio::test]
    async fn negotiates_protocol_range() {
        let s = mock::spawn(|m, p| match m {
            "server.version" => {
                assert_eq!(p, &json!(["Wren", ["1.4", "1.8"]]));
                Ok(json!(["Fulcrum 2.1.2", "1.6"]))
            }
            _ => Err("unexpected".into()),
        });
        let c = ElectrumClient::connect(s).await.unwrap();
        assert_eq!(c.protocol(), "1.6");
    }

    #[tokio::test]
    async fn headers_accepts_list_and_variable_width_hex() {
        let raws = activation()[6..10].to_vec(); // two 80-byte then two 164-byte headers
        let list: Vec<String> = raws.iter().map(hex::encode).collect();
        let concat = hex::encode(raws.concat());
        let s1 = mock::spawn(move |m, _| match m {
            "server.version" => Ok(json!(["x", "1.6"])),
            _ => Ok(json!({"count": 4, "headers": list.clone(), "max": 2016})),
        });
        let s2 = mock::spawn(move |m, _| match m {
            "server.version" => Ok(json!(["x", "1.4"])),
            _ => Ok(json!({"count": 4, "hex": concat.clone(), "max": 2016})),
        });
        assert_eq!(ElectrumClient::connect(s1).await.unwrap().headers(961_638, 4).await.unwrap(), raws);
        assert_eq!(ElectrumClient::connect(s2).await.unwrap().headers(961_638, 4).await.unwrap(), raws);
    }

    #[tokio::test]
    async fn server_errors_surface_as_server_error() {
        let s = mock::spawn(|m, _| match m {
            "server.version" => Ok(json!(["x", "1.4"])),
            _ => Err("history too large".into()),
        });
        let c = ElectrumClient::connect(s).await.unwrap();
        assert!(matches!(c.histories(&[ScriptBuf::new()]).await, Err(EngineError::Server(_))));
    }

    #[tokio::test]
    async fn rejects_transaction_that_does_not_match_requested_txid() {
        let real = tx(1000);
        let other = tx(2000);
        let other_hex = hex::encode(serialize(&other));
        let s = mock::spawn(move |m, _| match m {
            "server.version" => Ok(json!(["x", "1.4"])),
            _ => Ok(json!(other_hex.clone())),
        });
        let c = ElectrumClient::connect(s).await.unwrap();
        assert!(c.transactions(&[real.compute_txid()]).await.is_err());
    }

    #[test]
    fn merkle_verification() {
        let a = tx(1).compute_txid();
        let b = tx(2).compute_txid();
        let c = tx(3).compute_txid();
        // Tree for [a, b, c]: c is duplicated at the leaf level.
        let pair = |x: [u8; 32], y: [u8; 32]| { let mut v = x.to_vec(); v.extend(y); crate::hashes::sha256d(&v) };
        let ab = pair(a.to_byte_array(), b.to_byte_array());
        let cc = pair(c.to_byte_array(), c.to_byte_array());
        let root = pair(ab, cc);
        let proof_c = MerkleProof { block_height: 1, pos: 2, merkle: vec![c.to_byte_array(), ab] };
        assert!(verify_merkle(&c, &proof_c, &root));
        let wrong_pos = MerkleProof { pos: 3, ..proof_c.clone() };
        assert!(!verify_merkle(&c, &wrong_pos, &root));
    }

    #[test]
    fn script_hash_is_reversed_sha256() {
        // Electrum protocol docs example: P2PKH 1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa
        let spk = ScriptBuf::from_hex("76a91462e907b15cbf27d5425399ebf6f0fb50ebb88f1888ac").unwrap();
        assert_eq!(script_hash(&spk), "8b01df4e368ea28f8dc0423bcf7a4923e3a12d307c875e47a0cfbf90b5c39161");
    }
}
```

- [ ] **Step 3: Run to verify failure**

Run: `cargo test -p wren-engine electrum`
Expected: FAIL to compile — module missing.

- [ ] **Step 4: Implement `rpc.rs`**

```rust
//! Newline-delimited JSON-RPC over any byte stream (TCP, TLS, or a Tor stream).
use crate::error::EngineError;
use serde_json::{json, Value};
use std::time::Duration;
use tokio::io::{AsyncBufReadExt, AsyncRead, AsyncWrite, AsyncWriteExt, BufReader, ReadHalf, WriteHalf};

pub trait Stream: AsyncRead + AsyncWrite + Unpin + Send {}
impl<T: AsyncRead + AsyncWrite + Unpin + Send> Stream for T {}
pub type BoxStream = Box<dyn Stream>;

const MAX_LINE: usize = 16 * 1024 * 1024;
const TIMEOUT: Duration = Duration::from_secs(60);

pub struct Rpc {
    reader: BufReader<ReadHalf<BoxStream>>,
    writer: WriteHalf<BoxStream>,
    next_id: u64,
}

fn net<E: std::fmt::Display>(e: E) -> EngineError {
    EngineError::Network(e.to_string())
}

impl Rpc {
    pub fn new(stream: BoxStream) -> Self {
        let (r, w) = tokio::io::split(stream);
        Rpc { reader: BufReader::new(r), writer: w, next_id: 1 }
    }

    pub async fn call(&mut self, method: &str, params: Value) -> Result<Value, EngineError> {
        Ok(self.batch(vec![(method, params)]).await?.remove(0))
    }

    pub async fn batch(&mut self, calls: Vec<(&str, Value)>) -> Result<Vec<Value>, EngineError> {
        if calls.is_empty() {
            return Ok(vec![]);
        }
        let first = self.next_id;
        let n = calls.len() as u64;
        self.next_id += n;
        let mut out = String::new();
        for (i, (m, p)) in calls.iter().enumerate() {
            out.push_str(&json!({"jsonrpc": "2.0", "id": first + i as u64, "method": m, "params": p}).to_string());
            out.push('\n');
        }
        tokio::time::timeout(TIMEOUT, self.writer.write_all(out.as_bytes())).await.map_err(net)?.map_err(net)?;
        tokio::time::timeout(TIMEOUT, self.writer.flush()).await.map_err(net)?.map_err(net)?;

        let mut results: Vec<Option<Value>> = vec![None; calls.len()];
        let mut remaining = calls.len();
        while remaining > 0 {
            let line = tokio::time::timeout(TIMEOUT, self.read_line()).await.map_err(net)??;
            let v: Value = serde_json::from_slice(&line).map_err(|_| EngineError::Server("malformed response".into()))?;
            let Some(id) = v.get("id").and_then(Value::as_u64) else { continue };
            if id < first || id >= first + n {
                continue;
            }
            let slot = &mut results[(id - first) as usize];
            if slot.is_some() {
                continue;
            }
            if let Some(e) = v.get("error").filter(|e| !e.is_null()) {
                let msg = e.get("message").and_then(Value::as_str).unwrap_or("error");
                return Err(EngineError::Server(msg.chars().take(200).collect()));
            }
            *slot = Some(v.get("result").cloned().unwrap_or(Value::Null));
            remaining -= 1;
        }
        Ok(results.into_iter().map(|x| x.expect("filled")).collect())
    }

    async fn read_line(&mut self) -> Result<Vec<u8>, EngineError> {
        let mut buf = Vec::new();
        loop {
            let avail = self.reader.fill_buf().await.map_err(net)?;
            if avail.is_empty() {
                return Err(EngineError::Network("connection closed".into()));
            }
            if let Some(pos) = avail.iter().position(|&b| b == b'\n') {
                buf.extend_from_slice(&avail[..pos]);
                self.reader.consume(pos + 1);
                return Ok(buf);
            }
            let len = avail.len();
            buf.extend_from_slice(avail);
            self.reader.consume(len);
            if buf.len() > MAX_LINE {
                return Err(EngineError::Server("response too large".into()));
            }
        }
    }
}
```

- [ ] **Step 5: Implement `mod.rs`**

```rust
//! Typed Electrum client. Headers are always handled as raw bytes (80 or 164) — never decoded with
//! rust-bitcoin's 80-byte Header.
pub mod rpc;
#[cfg(test)]
pub(crate) mod mock;

use crate::error::EngineError;
use crate::hashes::{sha256, sha256d};
use crate::header::ForkHeader;
use bdk_wallet::bitcoin::{consensus::encode::{deserialize, serialize}, hashes::Hash, FeeRate, Script, ScriptBuf, Transaction, Txid};
use rpc::{BoxStream, Rpc};
use serde_json::{json, Value};
use std::str::FromStr;
use tokio::sync::Mutex;

pub struct ElectrumClient {
    rpc: Mutex<Rpc>,
    protocol: String,
}

#[derive(Clone, Debug)]
pub struct HistoryItem {
    pub txid: Txid,
    pub height: i64,
}

#[derive(Clone, Debug)]
pub struct MerkleProof {
    pub block_height: u32,
    pub pos: u32,
    pub merkle: Vec<[u8; 32]>,
}

fn bad(what: &str) -> EngineError {
    EngineError::Server(format!("unexpected {what} from server"))
}

fn hex_bytes(v: &Value) -> Result<Vec<u8>, EngineError> {
    hex::decode(v.as_str().ok_or_else(|| bad("value"))?).map_err(|_| bad("hex"))
}

pub fn script_hash(spk: &Script) -> String {
    let mut h = sha256(spk.as_bytes());
    h.reverse();
    hex::encode(h)
}

pub fn split_concatenated_headers(b: &[u8]) -> Result<Vec<Vec<u8>>, EngineError> {
    let mut out = Vec::new();
    let mut i = 0;
    while i < b.len() {
        if b.len() - i < 4 {
            return Err(bad("header data"));
        }
        let n = ForkHeader::len_for_version(u32::from_le_bytes(b[i..i + 4].try_into().unwrap()));
        if b.len() - i < n {
            return Err(bad("header data"));
        }
        out.push(b[i..i + n].to_vec());
        i += n;
    }
    Ok(out)
}

pub fn verify_merkle(txid: &Txid, proof: &MerkleProof, merkle_root_internal: &[u8; 32]) -> bool {
    let mut h = txid.to_byte_array();
    let mut pos = proof.pos;
    for sib in &proof.merkle {
        let mut buf = [0u8; 64];
        if pos & 1 == 1 {
            buf[..32].copy_from_slice(sib);
            buf[32..].copy_from_slice(&h);
        } else {
            buf[..32].copy_from_slice(&h);
            buf[32..].copy_from_slice(sib);
        }
        h = sha256d(&buf);
        pos >>= 1;
    }
    pos == 0 && h == *merkle_root_internal
}

/// BTC/kvB (Electrum) -> FeeRate, rounded up, floored at 1 sat/vB.
fn fee_from_btc_per_kvb(btc: f64) -> FeeRate {
    let sat_per_kvb = (btc * 100_000_000.0).max(1000.0);
    FeeRate::from_sat_per_kwu((sat_per_kvb / 4.0).ceil() as u64)
}

impl ElectrumClient {
    pub async fn connect(stream: BoxStream) -> Result<Self, EngineError> {
        let mut rpc = Rpc::new(stream);
        let r = rpc.call("server.version", json!(["Wren", ["1.4", "1.8"]])).await?;
        let protocol = r.get(1).and_then(Value::as_str).ok_or_else(|| bad("version"))?.to_string();
        Ok(ElectrumClient { rpc: Mutex::new(rpc), protocol })
    }

    pub fn protocol(&self) -> &str {
        &self.protocol
    }

    async fn call(&self, m: &str, p: Value) -> Result<Value, EngineError> {
        self.rpc.lock().await.call(m, p).await
    }

    async fn batch(&self, calls: Vec<(&str, Value)>) -> Result<Vec<Value>, EngineError> {
        let mut out = Vec::with_capacity(calls.len());
        for chunk in calls.chunks(50) {
            out.extend(self.rpc.lock().await.batch(chunk.to_vec()).await?);
        }
        Ok(out)
    }

    pub async fn tip(&self) -> Result<(u32, Vec<u8>), EngineError> {
        let r = self.call("blockchain.headers.subscribe", json!([])).await?;
        let h = r.get("height").and_then(Value::as_u64).ok_or_else(|| bad("tip"))? as u32;
        Ok((h, hex_bytes(r.get("hex").ok_or_else(|| bad("tip"))?)?))
    }

    pub async fn header(&self, height: u32) -> Result<Vec<u8>, EngineError> {
        hex_bytes(&self.call("blockchain.block.header", json!([height])).await?)
    }

    pub async fn headers(&self, start: u32, count: u32) -> Result<Vec<Vec<u8>>, EngineError> {
        let r = self.call("blockchain.block.headers", json!([start, count])).await?;
        let out = if let Some(list) = r.get("headers").and_then(Value::as_array) {
            list.iter().map(hex_bytes).collect::<Result<Vec<_>, _>>()?
        } else if let Some(h) = r.get("hex") {
            split_concatenated_headers(&hex_bytes(h)?)?
        } else {
            return Err(bad("headers"));
        };
        if out.len() > count as usize {
            return Err(bad("header count"));
        }
        Ok(out)
    }

    pub async fn histories(&self, scripts: &[ScriptBuf]) -> Result<Vec<Vec<HistoryItem>>, EngineError> {
        let calls = scripts.iter().map(|s| ("blockchain.scripthash.get_history", json!([script_hash(s)]))).collect();
        self.batch(calls).await?.into_iter().map(|v| {
            v.as_array().ok_or_else(|| bad("history"))?.iter().map(|i| {
                Ok(HistoryItem {
                    txid: Txid::from_str(i.get("tx_hash").and_then(Value::as_str).ok_or_else(|| bad("history"))?).map_err(|_| bad("txid"))?,
                    height: i.get("height").and_then(Value::as_i64).ok_or_else(|| bad("history"))?,
                })
            }).collect()
        }).collect()
    }

    pub async fn transactions(&self, txids: &[Txid]) -> Result<Vec<Transaction>, EngineError> {
        let calls = txids.iter().map(|t| ("blockchain.transaction.get", json!([t.to_string()]))).collect();
        let raws = self.batch(calls).await?;
        txids.iter().zip(raws).map(|(want, v)| {
            let tx: Transaction = deserialize(&hex_bytes(&v)?).map_err(|_| bad("transaction"))?;
            if tx.compute_txid() != *want {
                return Err(EngineError::Server("server sent a transaction that does not match its id".into()));
            }
            Ok(tx)
        }).collect()
    }

    pub async fn merkle_proofs(&self, items: &[(Txid, u32)]) -> Result<Vec<MerkleProof>, EngineError> {
        let calls = items.iter().map(|(t, h)| ("blockchain.transaction.get_merkle", json!([t.to_string(), h]))).collect();
        self.batch(calls).await?.into_iter().map(|v| {
            let merkle = v.get("merkle").and_then(Value::as_array).ok_or_else(|| bad("merkle"))?.iter().map(|m| {
                let mut b: [u8; 32] = hex_bytes(m)?.try_into().map_err(|_| bad("merkle"))?;
                b.reverse();
                Ok(b)
            }).collect::<Result<Vec<_>, EngineError>>()?;
            Ok(MerkleProof {
                block_height: v.get("block_height").and_then(Value::as_u64).ok_or_else(|| bad("merkle"))? as u32,
                pos: v.get("pos").and_then(Value::as_u64).ok_or_else(|| bad("merkle"))? as u32,
                merkle,
            })
        }).collect()
    }

    pub async fn estimate_fee(&self, blocks: u32) -> Result<Option<FeeRate>, EngineError> {
        let v = self.call("blockchain.estimatefee", json!([blocks])).await?;
        let btc = v.as_f64().ok_or_else(|| bad("fee"))?;
        Ok((btc > 0.0).then(|| fee_from_btc_per_kvb(btc)))
    }

    pub async fn relay_fee(&self) -> Result<FeeRate, EngineError> {
        let v = self.call("blockchain.relayfee", json!([])).await?;
        Ok(fee_from_btc_per_kvb(v.as_f64().ok_or_else(|| bad("fee"))?))
    }

    pub async fn broadcast(&self, tx: &Transaction) -> Result<Txid, EngineError> {
        let v = self.call("blockchain.transaction.broadcast", json!([hex::encode(serialize(tx))])).await?;
        let got = Txid::from_str(v.as_str().ok_or_else(|| bad("broadcast"))?).map_err(|_| bad("txid"))?;
        if got != tx.compute_txid() {
            return Err(bad("broadcast txid"));
        }
        Ok(got)
    }
}
```
Add `pub mod electrum;` to `lib.rs`.

- [ ] **Step 6: Run tests**

Run: `cargo test -p wren-engine electrum`
Expected: PASS (6 tests).

- [ ] **Step 7: Commit**

```bash
git add core/wren-engine/src/electrum core/wren-engine/src/lib.rs
git commit -m "feat(engine): Electrum JSON-RPC client with raw 80/164-byte headers"
```

---

### Task 10: Transports — TCP, TLS with web PKI, TLS with a pinned self-signed certificate

**Files:**
- Create: `core/wren-engine/src/electrum/transport.rs`
- Modify: `core/wren-engine/src/electrum/mod.rs` (add `pub mod transport;`), `core/wren-engine/Cargo.toml` (dev-dependency `rcgen`)
- Test: inline tests in `transport.rs`

**Interfaces:**
- Consumes: `rpc::BoxStream`, `hashes::sha256`.
- Produces:
```rust
#[derive(Clone, Debug, PartialEq, Eq)]
pub enum ServerAddress {
    Tcp { host: String, port: u16 },
    Tls { host: String, port: u16, pinned_sha256: Option<[u8; 32]> }, // None = web PKI roots
    Onion { host: String, port: u16, tls: bool },                      // implemented in Task 21
}
pub async fn connect(addr: &ServerAddress) -> Result<BoxStream, EngineError>;
/// Trust-on-first-use: handshake far enough to read the server certificate, abort, return its sha256.
/// The caller shows it to the user; nothing is sent to the server.
pub async fn fetch_certificate_fingerprint(host: &str, port: u16) -> Result<[u8; 32], EngineError>;
```
Self-signed pinning exists because personal nodes (Start9, Umbrel) serve self-signed certificates. A pinned connection accepts exactly one certificate; there is no "accept any certificate" mode.

- [ ] **Step 1: Add the dev-dependency**

Run: `cd core/wren-engine && cargo add --dev rcgen@0.13`

- [ ] **Step 2: Write the failing tests**

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use rustls::pki_types::{CertificateDer, PrivateKeyDer, PrivatePkcs8KeyDer};
    use std::sync::Arc;
    use tokio::io::{AsyncBufReadExt, AsyncWriteExt, BufReader};
    use tokio::net::TcpListener;

    /// Starts a TLS echo server with a fresh self-signed cert; returns (port, sha256 of the cert).
    async fn tls_echo_server() -> (u16, [u8; 32]) {
        let ck = rcgen::generate_simple_self_signed(vec!["localhost".into()]).unwrap();
        let cert_der = CertificateDer::from(ck.cert.der().to_vec());
        let fp = crate::hashes::sha256(cert_der.as_ref());
        let key = PrivateKeyDer::Pkcs8(PrivatePkcs8KeyDer::from(ck.key_pair.serialize_der()));
        let cfg = rustls::ServerConfig::builder_with_provider(Arc::new(rustls::crypto::ring::default_provider()))
            .with_safe_default_protocol_versions().unwrap()
            .with_no_client_auth()
            .with_single_cert(vec![cert_der], key).unwrap();
        let acceptor = tokio_rustls::TlsAcceptor::from(Arc::new(cfg));
        let listener = TcpListener::bind("127.0.0.1:0").await.unwrap();
        let port = listener.local_addr().unwrap().port();
        tokio::spawn(async move {
            loop {
                let (tcp, _) = listener.accept().await.unwrap();
                let acceptor = acceptor.clone();
                tokio::spawn(async move {
                    if let Ok(tls) = acceptor.accept(tcp).await {
                        let (r, mut w) = tokio::io::split(tls);
                        let mut lines = BufReader::new(r).lines();
                        while let Ok(Some(l)) = lines.next_line().await {
                            let _ = w.write_all(format!("{l}\n").as_bytes()).await;
                        }
                    }
                });
            }
        });
        (port, fp)
    }

    async fn roundtrip(mut s: BoxStream) -> String {
        s.write_all(b"ping\n").await.unwrap();
        let mut line = String::new();
        BufReader::new(s).read_line(&mut line).await.unwrap();
        line
    }

    #[tokio::test]
    async fn pinned_self_signed_cert_connects() {
        let (port, fp) = tls_echo_server().await;
        let s = connect(&ServerAddress::Tls { host: "localhost".into(), port, pinned_sha256: Some(fp) }).await.unwrap();
        assert_eq!(roundtrip(s).await, "ping\n");
    }

    #[tokio::test]
    async fn wrong_pin_is_refused() {
        let (port, _) = tls_echo_server().await;
        let r = connect(&ServerAddress::Tls { host: "localhost".into(), port, pinned_sha256: Some([0u8; 32]) }).await;
        assert!(r.is_err());
    }

    #[tokio::test]
    async fn self_signed_without_pin_is_refused_by_web_pki() {
        let (port, _) = tls_echo_server().await;
        let r = connect(&ServerAddress::Tls { host: "localhost".into(), port, pinned_sha256: None }).await;
        assert!(r.is_err());
    }

    #[tokio::test]
    async fn fingerprint_capture_matches_server_cert() {
        let (port, fp) = tls_echo_server().await;
        assert_eq!(fetch_certificate_fingerprint("localhost", port).await.unwrap(), fp);
    }
}
```

- [ ] **Step 3: Run to verify failure**

Run: `cargo test -p wren-engine transport`
Expected: FAIL to compile — module missing.

- [ ] **Step 4: Implement**

```rust
//! Byte-stream transports for the Electrum client.
use super::rpc::BoxStream;
use crate::error::EngineError;
use crate::hashes::sha256;
use rustls::client::danger::{HandshakeSignatureValid, ServerCertVerified, ServerCertVerifier};
use rustls::crypto::{verify_tls12_signature, verify_tls13_signature, CryptoProvider};
use rustls::pki_types::{CertificateDer, ServerName, UnixTime};
use rustls::{ClientConfig, DigitallySignedStruct, RootCertStore, SignatureScheme};
use std::sync::{Arc, Mutex};
use tokio::net::TcpStream;
use tokio_rustls::TlsConnector;

#[derive(Clone, Debug, PartialEq, Eq)]
pub enum ServerAddress {
    Tcp { host: String, port: u16 },
    Tls { host: String, port: u16, pinned_sha256: Option<[u8; 32]> },
    Onion { host: String, port: u16, tls: bool },
}

fn net<E: std::fmt::Display>(e: E) -> EngineError {
    EngineError::Network(e.to_string())
}

fn provider() -> Arc<CryptoProvider> {
    Arc::new(rustls::crypto::ring::default_provider())
}

/// Accepts exactly one certificate (by sha256 of its DER), still verifying handshake signatures.
#[derive(Debug)]
struct PinnedCert {
    pin: [u8; 32],
    provider: Arc<CryptoProvider>,
}

impl ServerCertVerifier for PinnedCert {
    fn verify_server_cert(&self, end_entity: &CertificateDer<'_>, _: &[CertificateDer<'_>], _: &ServerName<'_>, _: &[u8], _: UnixTime) -> Result<ServerCertVerified, rustls::Error> {
        if sha256(end_entity.as_ref()) == self.pin {
            Ok(ServerCertVerified::assertion())
        } else {
            Err(rustls::Error::General("server certificate does not match the pinned fingerprint".into()))
        }
    }
    fn verify_tls12_signature(&self, m: &[u8], c: &CertificateDer<'_>, d: &DigitallySignedStruct) -> Result<HandshakeSignatureValid, rustls::Error> {
        verify_tls12_signature(m, c, d, &self.provider.signature_verification_algorithms)
    }
    fn verify_tls13_signature(&self, m: &[u8], c: &CertificateDer<'_>, d: &DigitallySignedStruct) -> Result<HandshakeSignatureValid, rustls::Error> {
        verify_tls13_signature(m, c, d, &self.provider.signature_verification_algorithms)
    }
    fn supported_verify_schemes(&self) -> Vec<SignatureScheme> {
        self.provider.signature_verification_algorithms.supported_schemes()
    }
}

/// Records the certificate and then fails the handshake, so no application data is ever exchanged.
#[derive(Debug)]
struct CaptureCert {
    seen: Mutex<Option<[u8; 32]>>,
    provider: Arc<CryptoProvider>,
}

impl ServerCertVerifier for CaptureCert {
    fn verify_server_cert(&self, end_entity: &CertificateDer<'_>, _: &[CertificateDer<'_>], _: &ServerName<'_>, _: &[u8], _: UnixTime) -> Result<ServerCertVerified, rustls::Error> {
        *self.seen.lock().unwrap() = Some(sha256(end_entity.as_ref()));
        Err(rustls::Error::General("fingerprint captured; connection intentionally aborted".into()))
    }
    fn verify_tls12_signature(&self, m: &[u8], c: &CertificateDer<'_>, d: &DigitallySignedStruct) -> Result<HandshakeSignatureValid, rustls::Error> {
        verify_tls12_signature(m, c, d, &self.provider.signature_verification_algorithms)
    }
    fn verify_tls13_signature(&self, m: &[u8], c: &CertificateDer<'_>, d: &DigitallySignedStruct) -> Result<HandshakeSignatureValid, rustls::Error> {
        verify_tls13_signature(m, c, d, &self.provider.signature_verification_algorithms)
    }
    fn supported_verify_schemes(&self) -> Vec<SignatureScheme> {
        self.provider.signature_verification_algorithms.supported_schemes()
    }
}

fn server_name(host: &str) -> Result<ServerName<'static>, EngineError> {
    ServerName::try_from(host.to_string()).map_err(|_| EngineError::Input("invalid server host name"))
}

async fn tls_connect(host: &str, port: u16, config: ClientConfig) -> Result<BoxStream, EngineError> {
    let tcp = TcpStream::connect((host, port)).await.map_err(net)?;
    let tls = TlsConnector::from(Arc::new(config)).connect(server_name(host)?, tcp).await.map_err(net)?;
    Ok(Box::new(tls))
}

pub async fn connect(addr: &ServerAddress) -> Result<BoxStream, EngineError> {
    match addr {
        ServerAddress::Tcp { host, port } => Ok(Box::new(TcpStream::connect((host.as_str(), *port)).await.map_err(net)?)),
        ServerAddress::Tls { host, port, pinned_sha256 } => {
            let p = provider();
            let builder = ClientConfig::builder_with_provider(p.clone()).with_safe_default_protocol_versions().map_err(net)?;
            let config = match pinned_sha256 {
                Some(pin) => builder.dangerous().with_custom_certificate_verifier(Arc::new(PinnedCert { pin: *pin, provider: p })).with_no_client_auth(),
                None => {
                    let mut roots = RootCertStore::empty();
                    roots.extend(webpki_roots::TLS_SERVER_ROOTS.iter().cloned());
                    builder.with_root_certificates(roots).with_no_client_auth()
                }
            };
            tls_connect(host, *port, config).await
        }
        ServerAddress::Onion { .. } => Err(EngineError::Network("Tor transport not built in (Task 21)".into())),
    }
}

pub async fn fetch_certificate_fingerprint(host: &str, port: u16) -> Result<[u8; 32], EngineError> {
    let p = provider();
    let capture = Arc::new(CaptureCert { seen: Mutex::new(None), provider: p.clone() });
    let config = ClientConfig::builder_with_provider(p).with_safe_default_protocol_versions().map_err(net)?
        .dangerous().with_custom_certificate_verifier(capture.clone()).with_no_client_auth();
    let _ = tls_connect(host, port, config).await; // always fails by design
    let seen = *capture.seen.lock().unwrap();
    seen.ok_or(EngineError::Network("server did not present a certificate".into()))
}
```
Add `pub mod transport;` to `electrum/mod.rs`.

- [ ] **Step 5: Run tests**

Run: `cargo test -p wren-engine transport`
Expected: PASS (4 tests).

- [ ] **Step 6: Commit**

```bash
git add core/wren-engine/src/electrum core/wren-engine/Cargo.toml core/Cargo.lock
git commit -m "feat(engine): TCP and TLS transports with web PKI or pinned self-signed certs"
```

---

### Task 11: Header sync (with reorg handling) and test chain utilities

**Files:**
- Create: `core/wren-engine/src/sync.rs` (header part), `core/wren-engine/src/testutil.rs` (`#[cfg(test)]`)
- Modify: `core/wren-engine/src/lib.rs` (add `pub mod sync;` and `#[cfg(test)] pub(crate) mod testutil;`)
- Test: inline tests in `sync.rs`

**Interfaces:**
- Consumes: `ElectrumClient` (Task 9), `HeaderChain`, `ChainParams`, `Checkpoint` (Task 6), `ForkHeader` (Tasks 3–4), `pow::{check_pow_limit, regtest_pow_limit}`.
- Produces:
```rust
// sync.rs
pub const MAX_REORG_DEPTH: u32 = 144;
pub const MAX_SERVER_LAG: u32 = 6;
pub struct HeaderSyncOutcome { pub tip_height: u32, pub reorg_from: Option<u32> }
pub async fn sync_headers(client: &ElectrumClient, chain: &mut HeaderChain) -> Result<HeaderSyncOutcome, EngineError>;
pub fn save_chain(chain: &HeaderChain, path: &Path) -> Result<(), EngineError>;          // atomic write
pub fn load_chain(params: ChainParams, path: &Path) -> Result<Option<HeaderChain>, EngineError>;
// testutil.rs (tests only)
pub fn mine(prev: [u8; 32], height: u32, blake2b_height: u32, time: u32, salt: u8, merkle_root: [u8; 32]) -> ForkHeader;
pub struct TestChain { pub headers: Vec<ForkHeader>, pub anchor_height: u32 }
impl TestChain {
    pub fn new(anchor_height: u32, len: u32, blake2b_height: u32, salt: u8) -> TestChain;
    pub fn fork_at(&self, height: u32, new_len: u32, salt: u8) -> TestChain; // shares headers below `height`
    pub fn params(&self) -> ChainParams;                                     // regtest params anchored at headers[0]
    pub fn set_merkle_root(&mut self, height: u32, root: [u8; 32]);          // re-mines height and above
    pub fn raw(&self, height: u32) -> Vec<u8>;
    pub fn tip(&self) -> u32;
}
pub struct MockWorld; // shared, mutable server state: chain + script histories + txs + merkle proofs
impl MockWorld {
    pub fn new(chain: TestChain) -> Arc<Mutex<MockWorld>>;
}
pub fn serve(world: Arc<Mutex<MockWorld>>) -> BoxStream; // Electrum mock over MockWorld
```

- [ ] **Step 1: Write `testutil.rs`**

```rust
//! Synthetic regtest chains and a mock Electrum server over them. Tests only.
use crate::electrum::{mock, rpc::BoxStream, script_hash};
use crate::header::{ForkHeader, HeaderV2, VERSION_HEADER_V2_FLAG};
use crate::params::{ChainParams, Checkpoint};
use crate::pow::{check_pow_limit, regtest_pow_limit};
use bdk_wallet::bitcoin::{consensus::encode::serialize, ScriptBuf, Transaction, Txid};
use serde_json::{json, Value};
use std::collections::HashMap;
use std::sync::{Arc, Mutex};

pub fn mine(prev: [u8; 32], height: u32, blake2b_height: u32, time: u32, salt: u8, merkle_root: [u8; 32]) -> ForkHeader {
    let v2 = height >= blake2b_height;
    let mut h = ForkHeader {
        version: if v2 { 0x2000_0000 | VERSION_HEADER_V2_FLAG } else { 0x2000_0000 },
        prev_hash: prev,
        merkle_root,
        time_on_wire: time,
        bits: 0x207f_ffff,
        nonce: 0,
        v2: v2.then(|| HeaderV2 {
            nonce2: 0, nonce3: salt as u32, extranonce: [0; 16], time_offset: 0, txcount: 1, flags: 0,
            xor_key_mask_clear_bits: 0, xor_key: [0; 16], height: height as i32, mm_rhs: [0; 32],
        }),
    };
    if !v2 {
        h.merkle_root[31] ^= salt; // make v1 headers of different branches differ too
    }
    while !check_pow_limit(&h.block_hash(), h.bits, regtest_pow_limit()) {
        h.nonce += 1;
    }
    h
}

#[derive(Clone)]
pub struct TestChain {
    pub headers: Vec<ForkHeader>,
    pub anchor_height: u32,
    pub blake2b_height: u32,
    roots: HashMap<u32, [u8; 32]>,
}

impl TestChain {
    pub fn new(anchor_height: u32, len: u32, blake2b_height: u32, salt: u8) -> TestChain {
        let mut c = TestChain { headers: vec![], anchor_height, blake2b_height, roots: HashMap::new() };
        c.extend_from(0, len, salt, [0; 32]);
        c
    }
    fn extend_from(&mut self, keep: usize, total: u32, salt: u8, genesis_prev: [u8; 32]) {
        self.headers.truncate(keep);
        for i in keep as u32..total {
            let height = self.anchor_height + i;
            let prev = if i == 0 { genesis_prev } else { self.headers[i as usize - 1].block_hash() };
            let root = *self.roots.get(&height).unwrap_or(&[height as u8; 32]);
            self.headers.push(mine(prev, height, self.blake2b_height, 1_700_000_000 + i * 600, salt, root));
        }
    }
    pub fn fork_at(&self, height: u32, new_len: u32, salt: u8) -> TestChain {
        let mut c = self.clone();
        c.extend_from((height - self.anchor_height) as usize, new_len, salt, [0; 32]);
        c
    }
    pub fn set_merkle_root(&mut self, height: u32, root: [u8; 32]) {
        self.roots.insert(height, root);
        let len = self.headers.len() as u32;
        let salt = self.headers[(height - self.anchor_height) as usize].v2.as_ref().map(|v| v.nonce3 as u8).unwrap_or(0);
        self.extend_from((height - self.anchor_height) as usize, len, salt, [0; 32]);
    }
    pub fn params(&self) -> ChainParams {
        ChainParams::regtest(self.blake2b_height).with_anchor(Checkpoint { height: self.anchor_height, hash: self.headers[0].block_hash() })
    }
    pub fn raw(&self, height: u32) -> Vec<u8> {
        self.headers[(height - self.anchor_height) as usize].serialize()
    }
    pub fn tip(&self) -> u32 {
        self.anchor_height + self.headers.len() as u32 - 1
    }
}

pub struct MockWorld {
    pub chain: TestChain,
    pub history: HashMap<String, Vec<(Txid, i64)>>, // script hash -> [(txid, height)]
    pub txs: HashMap<Txid, Transaction>,
    pub tx_override: HashMap<Txid, Transaction>,    // serve a different tx for this id (attack tests)
    pub merkle: HashMap<Txid, (u32, u32, Vec<[u8; 32]>)>,
    pub broadcasts: Vec<Transaction>,
}

impl MockWorld {
    pub fn new(chain: TestChain) -> Arc<Mutex<MockWorld>> {
        Arc::new(Mutex::new(MockWorld { chain, history: HashMap::new(), txs: HashMap::new(), tx_override: HashMap::new(), merkle: HashMap::new(), broadcasts: vec![] }))
    }
    pub fn add_history(&mut self, spk: &ScriptBuf, txid: Txid, height: i64) {
        self.history.entry(script_hash(spk)).or_default().push((txid, height));
    }
}

pub fn serve(world: Arc<Mutex<MockWorld>>) -> BoxStream {
    mock::spawn(move |m, p| {
        let mut w = world.lock().unwrap();
        let u = |i: usize| p[i].as_u64().unwrap() as u32;
        match m {
            "server.version" => Ok(json!(["mock", "1.6"])),
            "blockchain.headers.subscribe" => Ok(json!({"height": w.chain.tip(), "hex": hex::encode(w.chain.raw(w.chain.tip()))})),
            "blockchain.block.header" => {
                let h = u(0);
                if h < w.chain.anchor_height || h > w.chain.tip() { return Err("height out of range".into()) }
                Ok(json!(hex::encode(w.chain.raw(h))))
            }
            "blockchain.block.headers" => {
                let (s, c) = (u(0), u(1));
                let end = (s + c).min(w.chain.tip() + 1);
                let list: Vec<String> = (s..end).map(|h| hex::encode(w.chain.raw(h))).collect();
                Ok(json!({"count": list.len(), "headers": list, "max": 2016}))
            }
            "blockchain.scripthash.get_history" => {
                let items: Vec<Value> = w.history.get(p[0].as_str().unwrap()).cloned().unwrap_or_default()
                    .into_iter().map(|(t, h)| json!({"tx_hash": t.to_string(), "height": h})).collect();
                Ok(json!(items))
            }
            "blockchain.transaction.get" => {
                let id: Txid = p[0].as_str().unwrap().parse().unwrap();
                let tx = w.tx_override.get(&id).or_else(|| w.txs.get(&id)).ok_or("unknown tx")?;
                Ok(json!(hex::encode(serialize(tx))))
            }
            "blockchain.transaction.get_merkle" => {
                let id: Txid = p[0].as_str().unwrap().parse().unwrap();
                let (bh, pos, path) = w.merkle.get(&id).cloned().ok_or("no proof")?;
                let path: Vec<String> = path.into_iter().map(|mut b| { b.reverse(); hex::encode(b) }).collect();
                Ok(json!({"block_height": bh, "pos": pos, "merkle": path}))
            }
            "blockchain.estimatefee" => Ok(json!(0.00002)),
            "blockchain.relayfee" => Ok(json!(0.00001)),
            "blockchain.transaction.broadcast" => {
                let tx: Transaction = bdk_wallet::bitcoin::consensus::deserialize(&hex::decode(p[0].as_str().unwrap()).unwrap()).unwrap();
                let id = tx.compute_txid();
                w.broadcasts.push(tx);
                Ok(json!(id.to_string()))
            }
            _ => Err(format!("unhandled {m}")),
        }
    })
}
```

- [ ] **Step 2: Write the failing tests** (in `sync.rs`)

```rust
#[cfg(test)]
mod header_sync_tests {
    use super::*;
    use crate::electrum::ElectrumClient;
    use crate::testutil::{serve, MockWorld, TestChain};

    async fn client(world: &std::sync::Arc<std::sync::Mutex<MockWorld>>) -> ElectrumClient {
        ElectrumClient::connect(serve(world.clone())).await.unwrap()
    }

    #[tokio::test]
    async fn syncs_from_empty_then_incrementally() {
        let a = TestChain::new(100, 30, 105, 1);
        let world = MockWorld::new(a.clone());
        let mut chain = HeaderChain::new(a.params());
        let out = sync_headers(&client(&world).await, &mut chain).await.unwrap();
        assert_eq!(out.tip_height, 129);
        world.lock().unwrap().chain = a.fork_at(130, 40, 1); // same branch, 10 more blocks
        let out = sync_headers(&client(&world).await, &mut chain).await.unwrap();
        assert_eq!((out.tip_height, out.reorg_from), (139, None));
    }

    #[tokio::test]
    async fn sync_handles_reorg_within_window() {
        let a = TestChain::new(100, 21, 105, 1);
        let world = MockWorld::new(a.clone());
        let mut chain = HeaderChain::new(a.params());
        sync_headers(&client(&world).await, &mut chain).await.unwrap();
        let b = a.fork_at(116, 23, 2); // diverges at 116, longer
        world.lock().unwrap().chain = b.clone();
        let out = sync_headers(&client(&world).await, &mut chain).await.unwrap();
        assert_eq!(out.reorg_from, Some(116));
        assert_eq!(chain.tip_hash(), Some(b.headers.last().unwrap().block_hash()));
    }

    #[tokio::test]
    async fn sync_refuses_reorg_deeper_than_window() {
        let a = TestChain::new(100, 200, 105, 1);
        let world = MockWorld::new(a.clone());
        let mut chain = HeaderChain::new(a.params());
        sync_headers(&client(&world).await, &mut chain).await.unwrap();
        world.lock().unwrap().chain = a.fork_at(101, 210, 2); // diverges 198 blocks below our tip
        assert!(matches!(sync_headers(&client(&world).await, &mut chain).await, Err(EngineError::Chain(_))));
    }

    #[tokio::test]
    async fn wrong_anchor_is_wrong_chain() {
        let a = TestChain::new(100, 10, 105, 1);
        let other = TestChain::new(100, 10, 105, 9);
        let world = MockWorld::new(other);
        let mut chain = HeaderChain::new(a.params());
        assert!(matches!(sync_headers(&client(&world).await, &mut chain).await, Err(EngineError::WrongChain)));
    }

    #[tokio::test]
    async fn save_and_load_round_trip() {
        let a = TestChain::new(100, 30, 105, 1);
        let world = MockWorld::new(a.clone());
        let mut chain = HeaderChain::new(a.params());
        sync_headers(&client(&world).await, &mut chain).await.unwrap();
        let dir = tempfile::tempdir().unwrap();
        let p = dir.path().join("headers.bin");
        save_chain(&chain, &p).unwrap();
        let loaded = load_chain(a.params(), &p).unwrap().unwrap();
        assert_eq!(loaded.tip_hash(), chain.tip_hash());
        assert!(load_chain(a.params(), &dir.path().join("missing.bin")).unwrap().is_none());
    }
}
```

- [ ] **Step 3: Run to verify failure**

Run: `cargo test -p wren-engine header_sync_tests`
Expected: FAIL to compile — `sync_headers` missing.

- [ ] **Step 4: Implement the header half of `sync.rs`**

```rust
//! Chain source: header sync (this task) and wallet script sync (Task 13).
use crate::electrum::ElectrumClient;
use crate::error::EngineError;
use crate::header::ForkHeader;
use crate::headerchain::HeaderChain;
use crate::params::ChainParams;
use std::path::Path;

pub const MAX_REORG_DEPTH: u32 = 144;
pub const MAX_SERVER_LAG: u32 = 6;

pub struct HeaderSyncOutcome {
    pub tip_height: u32,
    pub reorg_from: Option<u32>,
}

async fn server_hash(client: &ElectrumClient, height: u32) -> Result<[u8; 32], EngineError> {
    Ok(ForkHeader::parse(&client.header(height).await?)?.block_hash())
}

pub async fn sync_headers(client: &ElectrumClient, chain: &mut HeaderChain) -> Result<HeaderSyncOutcome, EngineError> {
    if chain.is_empty() {
        let raw = client.header(chain.base_height()).await?;
        chain.set_anchor(&raw)?;
    }
    let (server_tip, _) = client.tip().await?;
    let our_tip = chain.tip_height().expect("anchored");
    if our_tip > server_tip + MAX_SERVER_LAG {
        return Err(EngineError::Server("server is behind the chain".into()));
    }
    let mut reorg_from = None;
    let check = our_tip.min(server_tip);
    if check > chain.base_height() && server_hash(client, check).await? != chain.hash_at(check).unwrap() {
        let mut agree = None;
        for d in 1..=MAX_REORG_DEPTH {
            let Some(h) = check.checked_sub(d) else { break };
            if h < chain.base_height() {
                break;
            }
            if server_hash(client, h).await? == chain.hash_at(h).unwrap() {
                agree = Some(h);
                break;
            }
        }
        let agree = agree.ok_or(EngineError::Chain("reorganization deeper than 144 blocks"))?;
        chain.truncate_above(agree);
        reorg_from = Some(agree + 1);
    }
    let mut next = chain.tip_height().unwrap() + 1;
    while next <= server_tip {
        let count = (server_tip - next + 1).min(2016);
        let raws = client.headers(next, count).await?;
        if raws.is_empty() {
            return Err(EngineError::Server("server returned no headers".into()));
        }
        chain.extend(next, &raws)?;
        next += raws.len() as u32;
    }
    Ok(HeaderSyncOutcome { tip_height: chain.tip_height().unwrap(), reorg_from })
}

pub fn save_chain(chain: &HeaderChain, path: &Path) -> Result<(), EngineError> {
    let tmp = path.with_extension("tmp");
    std::fs::write(&tmp, chain.to_bytes()).map_err(|e| EngineError::Wallet(e.to_string()))?;
    std::fs::rename(&tmp, path).map_err(|e| EngineError::Wallet(e.to_string()))
}

pub fn load_chain(params: ChainParams, path: &Path) -> Result<Option<HeaderChain>, EngineError> {
    match std::fs::read(path) {
        Ok(b) => Ok(Some(HeaderChain::from_bytes(params, &b)?)),
        Err(e) if e.kind() == std::io::ErrorKind::NotFound => Ok(None),
        Err(e) => Err(EngineError::Wallet(e.to_string())),
    }
}
```
Add to `lib.rs`: `pub mod sync;` and `#[cfg(test)] pub(crate) mod testutil;`.

- [ ] **Step 5: Run tests**

Run: `cargo test -p wren-engine header_sync_tests`
Expected: PASS (5 tests).

- [ ] **Step 6: Commit**

```bash
git add core/wren-engine/src/sync.rs core/wren-engine/src/testutil.rs core/wren-engine/src/lib.rs
git commit -m "feat(engine): header sync with reorg window, persistence, synthetic test chains"
```

---

### Task 12: Wallet core (create, restore, open, addresses, balance, history)

**Files:**
- Create: `core/wren-engine/src/wallet.rs`
- Modify: `core/wren-engine/src/lib.rs` (add `pub mod wallet;`)
- Test: inline tests in `wallet.rs`

**Interfaces:**
- Consumes: `EngineError`.
- Produces:
```rust
pub const ACCOUNT_PATH: &str = "m/84'/0'/0'";
pub fn generate_words() -> Result<Vec<String>, EngineError>;  // 12 words from OsRng entropy
#[derive(Clone, Debug)]
pub struct TxSummary { pub txid: Txid, pub received: Amount, pub sent: Amount, pub fee: Option<Amount>,
    pub confirmation_height: Option<u32>, pub confirmation_time: Option<u64> }
pub struct WrenWallet;
impl WrenWallet {
    pub fn create(db_path: &Path, network: Network, words: &str, passphrase: &str) -> Result<Self, EngineError>; // fails if a wallet exists
    pub fn open(db_path: &Path, network: Network, words: &str, passphrase: &str) -> Result<Self, EngineError>;   // fails on mismatch
    pub fn next_receive_address(&mut self) -> Result<Address, EngineError>;
    pub fn balance(&self) -> Balance;
    pub fn transactions(&self) -> Vec<TxSummary>;        // newest first
    pub fn persist(&mut self) -> Result<(), EngineError>;
    pub(crate) fn bdk(&self) -> &Wallet;
    pub(crate) fn bdk_mut(&mut self) -> &mut Wallet;
    pub(crate) fn secret_key_for(&self, input: &psbt::Input) -> Result<SecretKey, EngineError>;
}
```
Keys: the mnemonic is parsed, turned into a seed (held in `Zeroizing`), turned into the master `Xpriv`, then dropped. BDK only ever sees **public** descriptors with key origin `[fingerprint/84'/0'/0']`, so the SQLite file never contains private keys. The master key's secret bytes and chain code are overwritten on drop.

- [ ] **Step 1: Write the failing tests**

```rust
#[cfg(test)]
mod tests {
    use super::*;
    const ABANDON: &str = "abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon about";

    #[test]
    fn bip84_official_vector_first_address() {
        let dir = tempfile::tempdir().unwrap();
        let mut w = WrenWallet::create(&dir.path().join("w.sqlite"), Network::Bitcoin, ABANDON, "").unwrap();
        assert_eq!(w.next_receive_address().unwrap().to_string(), "bc1qcr8te4kr609gcawutmrza0j4xv80jy8z306fyu");
    }

    #[test]
    fn reopen_with_same_words_and_refuse_wrong_passphrase() {
        let dir = tempfile::tempdir().unwrap();
        let p = dir.path().join("w.sqlite");
        drop(WrenWallet::create(&p, Network::Bitcoin, ABANDON, "tree").unwrap());
        assert!(WrenWallet::open(&p, Network::Bitcoin, ABANDON, "tree").is_ok());
        assert!(WrenWallet::open(&p, Network::Bitcoin, ABANDON, "").is_err());
    }

    #[test]
    fn create_refuses_to_overwrite() {
        let dir = tempfile::tempdir().unwrap();
        let p = dir.path().join("w.sqlite");
        drop(WrenWallet::create(&p, Network::Bitcoin, ABANDON, "").unwrap());
        assert!(WrenWallet::create(&p, Network::Bitcoin, ABANDON, "").is_err());
    }

    #[test]
    fn generated_words_are_12_valid_bip39_words() {
        let words = generate_words().unwrap();
        assert_eq!(words.len(), 12);
        let dir = tempfile::tempdir().unwrap();
        assert!(WrenWallet::create(&dir.path().join("w.sqlite"), Network::Bitcoin, &words.join(" "), "").is_ok());
    }

    #[test]
    fn rejects_invalid_and_24_word_phrases() {
        let dir = tempfile::tempdir().unwrap();
        let p = dir.path().join("w.sqlite");
        assert!(WrenWallet::create(&p, Network::Bitcoin, "abandon abandon", "").is_err());
        let w24 = "abandon ".repeat(23) + "art";
        assert!(WrenWallet::create(&p, Network::Bitcoin, &w24, "").is_err());
    }
}
```

- [ ] **Step 2: Run to verify failure**

Run: `cargo test -p wren-engine wallet::`
Expected: FAIL to compile — module missing.

- [ ] **Step 3: Implement**

```rust
//! Wren's own wallet: BDK bookkeeping over public descriptors + an in-memory signing key.
use crate::error::EngineError;
use bdk_wallet::bitcoin::bip32::{ChainCode, DerivationPath, Xpriv, Xpub};
use bdk_wallet::bitcoin::secp256k1::rand::{rngs::OsRng, RngCore};
use bdk_wallet::bitcoin::secp256k1::{All, PublicKey, Secp256k1, SecretKey};
use bdk_wallet::bitcoin::{psbt, Address, Amount, Network, Txid};
use bdk_wallet::chain::ChainPosition;
use bdk_wallet::keys::bip39::{Language, Mnemonic};
use bdk_wallet::rusqlite::Connection;
use bdk_wallet::{Balance, KeychainKind, PersistedWallet, Wallet};
use std::path::Path;
use std::str::FromStr;
use zeroize::Zeroizing;

pub const ACCOUNT_PATH: &str = "m/84'/0'/0'";

fn werr<E: std::fmt::Display>(e: E) -> EngineError {
    EngineError::Wallet(e.to_string())
}

pub fn generate_words() -> Result<Vec<String>, EngineError> {
    let mut entropy = Zeroizing::new([0u8; 16]);
    OsRng.fill_bytes(&mut *entropy);
    let m = Mnemonic::from_entropy(&*entropy).map_err(werr)?;
    Ok(m.words().map(String::from).collect())
}

struct SigningKeys {
    master: Xpriv,
    secp: Secp256k1<All>,
}

impl Drop for SigningKeys {
    fn drop(&mut self) {
        self.master.private_key.non_secure_erase();
        self.master.chain_code = ChainCode::from([0u8; 32]);
    }
}

fn keys_and_descriptors(words: &str, passphrase: &str, network: Network) -> Result<(SigningKeys, String, String), EngineError> {
    let m = Mnemonic::parse_in_normalized(Language::English, words).map_err(|_| EngineError::Input("recovery phrase is not valid"))?;
    if m.word_count() != 12 {
        return Err(EngineError::Input("recovery phrase must be 12 words"));
    }
    let seed = Zeroizing::new(m.to_seed(passphrase));
    drop(m);
    let secp = Secp256k1::new();
    let master = Xpriv::new_master(network, &*seed).map_err(werr)?;
    let fp = master.fingerprint(&secp);
    let account = master.derive_priv(&secp, &DerivationPath::from_str(ACCOUNT_PATH).unwrap()).map_err(werr)?;
    let xpub = Xpub::from_priv(&secp, &account);
    let ext = format!("wpkh([{fp}/84'/0'/0']{xpub}/0/*)");
    let int = format!("wpkh([{fp}/84'/0'/0']{xpub}/1/*)");
    Ok((SigningKeys { master, secp }, ext, int))
}

#[derive(Clone, Debug)]
pub struct TxSummary {
    pub txid: Txid,
    pub received: Amount,
    pub sent: Amount,
    pub fee: Option<Amount>,
    pub confirmation_height: Option<u32>,
    pub confirmation_time: Option<u64>,
}

pub struct WrenWallet {
    wallet: PersistedWallet<Connection>,
    db: Connection,
    keys: SigningKeys,
}

impl WrenWallet {
    pub fn create(db_path: &Path, network: Network, words: &str, passphrase: &str) -> Result<Self, EngineError> {
        let (keys, ext, int) = keys_and_descriptors(words, passphrase, network)?;
        let mut db = Connection::open(db_path).map_err(werr)?;
        if Wallet::load().load_wallet(&mut db).map_err(werr)?.is_some() {
            return Err(EngineError::Wallet("a wallet already exists on this device".into()));
        }
        let wallet = Wallet::create(ext, int).network(network).create_wallet(&mut db).map_err(werr)?;
        Ok(WrenWallet { wallet, db, keys })
    }

    pub fn open(db_path: &Path, network: Network, words: &str, passphrase: &str) -> Result<Self, EngineError> {
        let (keys, ext, int) = keys_and_descriptors(words, passphrase, network)?;
        let mut db = Connection::open(db_path).map_err(werr)?;
        let wallet = Wallet::load()
            .descriptor(KeychainKind::External, Some(ext))
            .descriptor(KeychainKind::Internal, Some(int))
            .check_network(network)
            .load_wallet(&mut db)
            .map_err(|_| EngineError::Input("recovery phrase or passphrase does not match this wallet"))?
            .ok_or(EngineError::Wallet("no wallet on this device".into()))?;
        Ok(WrenWallet { wallet, db, keys })
    }

    pub fn next_receive_address(&mut self) -> Result<Address, EngineError> {
        let a = self.wallet.reveal_next_address(KeychainKind::External).address;
        self.persist()?;
        Ok(a)
    }

    pub fn balance(&self) -> Balance {
        self.wallet.balance()
    }

    pub fn transactions(&self) -> Vec<TxSummary> {
        let mut out: Vec<TxSummary> = self.wallet.transactions().map(|wtx| {
            let tx = &wtx.tx_node.tx;
            let (sent, received) = self.wallet.sent_and_received(tx);
            let (h, t) = match &wtx.chain_position {
                ChainPosition::Confirmed { anchor, .. } => (Some(anchor.block_id.height), Some(anchor.confirmation_time)),
                _ => (None, None),
            };
            TxSummary { txid: tx.compute_txid(), received, sent, fee: self.wallet.calculate_fee(tx).ok(), confirmation_height: h, confirmation_time: t }
        }).collect();
        out.sort_by(|a, b| b.confirmation_height.unwrap_or(u32::MAX).cmp(&a.confirmation_height.unwrap_or(u32::MAX)));
        out
    }

    pub fn persist(&mut self) -> Result<(), EngineError> {
        self.wallet.persist(&mut self.db).map_err(werr)?;
        Ok(())
    }

    pub(crate) fn bdk(&self) -> &Wallet {
        &self.wallet
    }

    pub(crate) fn bdk_mut(&mut self) -> &mut Wallet {
        &mut self.wallet
    }

    pub(crate) fn secret_key_for(&self, input: &psbt::Input) -> Result<SecretKey, EngineError> {
        let fp = self.keys.master.fingerprint(&self.keys.secp);
        for (pk, (origin_fp, path)) in &input.bip32_derivation {
            if *origin_fp != fp {
                continue;
            }
            let k = self.keys.master.derive_priv(&self.keys.secp, path).map_err(werr)?;
            if PublicKey::from_secret_key(&self.keys.secp, &k.private_key) == *pk {
                return Ok(k.private_key);
            }
        }
        Err(EngineError::Signing("no key in this wallet for an input"))
    }
}
```
Add `pub mod wallet;` to `lib.rs`.

- [ ] **Step 4: Run tests**

Run: `cargo test -p wren-engine wallet::`
Expected: PASS (5 tests).

- [ ] **Step 5: Commit**

```bash
git add core/wren-engine/src/wallet.rs core/wren-engine/src/lib.rs
git commit -m "feat(engine): wallet core over public BIP84 descriptors with in-memory signing key"
```

---

### Task 13: Script sync — Electrum history into a verified BDK update

**Files:**
- Modify: `core/wren-engine/src/sync.rs` (add wallet sync)
- Test: inline tests in `sync.rs`

**Interfaces:**
- Consumes: `ElectrumClient::{histories, transactions, merkle_proofs, header}` + `verify_merkle` (Task 9), `HeaderChain` (Task 6), `bdk_wallet::Wallet`.
- Produces:
```rust
pub const STOP_GAP: u32 = 20;
pub struct KeychainScan { pub keychain: KeychainKind, pub last_active: Option<u32> }
/// Scans both keychains of `wallet` (gap limit STOP_GAP past the last used/revealed index), fetches
/// and txid-checks every transaction, verifies each confirmation with a merkle proof against the
/// verified header chain (or, for heights below the anchor, against the server's header — see note),
/// and applies one bdk_wallet::Update. Caller persists.
pub async fn sync_wallet(client: &ElectrumClient, chain: &HeaderChain, wallet: &mut Wallet, now: u64) -> Result<(), EngineError>;
/// Shared by Claim (Task 17).
pub(crate) async fn scan_keychain(client: &ElectrumClient, wallet: &Wallet, keychain: KeychainKind, stop_gap: u32,
    found: &mut HashMap<Txid, i64>) -> Result<Option<u32>, EngineError>;
pub(crate) async fn fetch_confirmed_update(client: &ElectrumClient, chain: &HeaderChain, genesis: BlockId,
    existing_chain: &[BlockId], found: &HashMap<Txid, i64>, have: &dyn Fn(&Txid) -> bool, now: u64) -> Result<Update, EngineError>;
```

**Trust note (for Gate A):** confirmations at or above the anchor (961,632) are proven against headers the engine verified itself. Coins confirmed **before** the split are proven against the server's own header at that height, which the engine cannot verify without the whole pre-split chain. A lying server can therefore invent or hide a pre-split coin, but it cannot make Wren lose money: amounts are taken only from parent transactions whose txid the engine checks, and SIGHASH_UNIFIED commits to every input's amount and script, so any transaction built on a fake coin is simply rejected by the network.

- [ ] **Step 1: Write the failing tests**

```rust
#[cfg(test)]
mod wallet_sync_tests {
    use super::*;
    use crate::electrum::ElectrumClient;
    use crate::testutil::{serve, MockWorld, TestChain};
    use crate::wallet::WrenWallet;
    use bdk_wallet::bitcoin::{absolute::LockTime, hashes::Hash, transaction::Version, Amount, Network, OutPoint, Transaction, TxIn, TxOut, Txid};
    use bdk_wallet::KeychainKind;

    const ABANDON: &str = "abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon about";

    fn pay(spk: bdk_wallet::bitcoin::ScriptBuf, sats: u64, salt: u8) -> Transaction {
        Transaction { version: Version::TWO, lock_time: LockTime::ZERO,
            input: vec![TxIn { previous_output: OutPoint::new(Txid::from_byte_array([salt; 32]), 0), ..Default::default() }],
            output: vec![TxOut { value: Amount::from_sat(sats), script_pubkey: spk }] }
    }

    async fn setup() -> (tempfile::TempDir, WrenWallet, TestChain, std::sync::Arc<std::sync::Mutex<MockWorld>>, HeaderChain, Transaction) {
        let dir = tempfile::tempdir().unwrap();
        let mut w = WrenWallet::create(&dir.path().join("w.sqlite"), Network::Regtest, ABANDON, "").unwrap();
        let spk = w.bdk_mut().peek_address(KeychainKind::External, 3).script_pubkey(); // index 3: tests the gap scan
        let funding = pay(spk.clone(), 50_000, 1);
        let mut tc = TestChain::new(100, 30, 105, 1);
        tc.set_merkle_root(110, funding.compute_txid().to_byte_array()); // single-tx block: root = txid
        let world = MockWorld::new(tc.clone());
        {
            let mut wd = world.lock().unwrap();
            wd.txs.insert(funding.compute_txid(), funding.clone());
            wd.add_history(&spk, funding.compute_txid(), 110);
            wd.merkle.insert(funding.compute_txid(), (110, 0, vec![]));
        }
        let mut chain = HeaderChain::new(tc.params());
        sync_headers(&ElectrumClient::connect(serve(world.clone())).await.unwrap(), &mut chain).await.unwrap();
        (dir, w, tc, world, chain, funding)
    }

    #[tokio::test]
    async fn confirmed_payment_is_found_past_unused_addresses() {
        let (_d, mut w, _tc, world, chain, _f) = setup().await;
        let c = ElectrumClient::connect(serve(world)).await.unwrap();
        sync_wallet(&c, &chain, w.bdk_mut(), 1_700_000_000).await.unwrap();
        assert_eq!(w.balance().confirmed, Amount::from_sat(50_000));
        assert_eq!(w.transactions()[0].confirmation_height, Some(110));
    }

    #[tokio::test]
    async fn unconfirmed_payment_is_pending() {
        let (_d, mut w, _tc, world, chain, _f) = setup().await;
        let spk = w.bdk_mut().peek_address(KeychainKind::External, 0).script_pubkey();
        let tx = pay(spk.clone(), 7_000, 2);
        {
            let mut wd = world.lock().unwrap();
            wd.txs.insert(tx.compute_txid(), tx.clone());
            wd.add_history(&spk, tx.compute_txid(), 0);
        }
        let c = ElectrumClient::connect(serve(world)).await.unwrap();
        sync_wallet(&c, &chain, w.bdk_mut(), 1_700_000_000).await.unwrap();
        assert_eq!(w.balance().untrusted_pending, Amount::from_sat(7_000));
    }

    #[tokio::test]
    async fn sync_rejects_tx_whose_hash_mismatches() {
        let (_d, mut w, _tc, world, chain, f) = setup().await;
        world.lock().unwrap().tx_override.insert(f.compute_txid(), pay(f.output[0].script_pubkey.clone(), 9_999_999, 9));
        let c = ElectrumClient::connect(serve(world)).await.unwrap();
        assert!(sync_wallet(&c, &chain, w.bdk_mut(), 1_700_000_000).await.is_err());
        assert_eq!(w.balance().total(), Amount::ZERO);
    }

    #[tokio::test]
    async fn sync_rejects_bad_merkle_proof() {
        let (_d, mut w, _tc, world, chain, f) = setup().await;
        world.lock().unwrap().merkle.insert(f.compute_txid(), (110, 1, vec![[7u8; 32]]));
        let c = ElectrumClient::connect(serve(world)).await.unwrap();
        assert!(matches!(sync_wallet(&c, &chain, w.bdk_mut(), 1_700_000_000).await, Err(EngineError::Chain(_))));
    }
}
```

- [ ] **Step 2: Run to verify failure**

Run: `cargo test -p wren-engine wallet_sync_tests`
Expected: FAIL to compile — `sync_wallet` missing.

- [ ] **Step 3: Implement** (append to `sync.rs`)

```rust
use crate::electrum::verify_merkle;
use bdk_wallet::bitcoin::{hashes::Hash, BlockHash, ScriptBuf, Txid};
use bdk_wallet::chain::{BlockId, CheckPoint, ConfirmationBlockTime, TxUpdate};
use bdk_wallet::{KeychainKind, Update, Wallet};
use std::collections::{BTreeMap, HashMap};
use std::sync::Arc;

pub const STOP_GAP: u32 = 20;

/// Gap-scans one keychain against the server, collecting (txid -> height). Returns the last used index.
pub(crate) async fn scan_keychain(client: &ElectrumClient, wallet: &Wallet, keychain: KeychainKind, stop_gap: u32, found: &mut HashMap<Txid, i64>) -> Result<Option<u32>, EngineError> {
    let last_revealed = wallet.spk_index().last_revealed_index(keychain);
    let mut last_active: Option<u32> = None;
    let mut index = 0u32;
    loop {
        let batch: Vec<ScriptBuf> = (index..index + STOP_GAP).map(|i| wallet.peek_address(keychain, i).script_pubkey()).collect();
        let histories = client.histories(&batch).await?;
        for (i, hist) in histories.iter().enumerate() {
            if !hist.is_empty() {
                last_active = Some(index + i as u32);
            }
            for item in hist {
                found.insert(item.txid, item.height);
            }
        }
        index += STOP_GAP;
        let frontier = last_active.max(last_revealed).map(|x| x + 1).unwrap_or(0);
        if index >= frontier + stop_gap {
            return Ok(last_active);
        }
    }
}

pub(crate) async fn fetch_confirmed_update(
    client: &ElectrumClient,
    chain: &HeaderChain,
    genesis: BlockId,
    existing_chain: &[BlockId],
    found: &HashMap<Txid, i64>,
    have: &dyn Fn(&Txid) -> bool,
    now: u64,
) -> Result<Update, EngineError> {
    let mut tx_update = TxUpdate::<ConfirmationBlockTime>::default();
    let missing: Vec<Txid> = found.keys().filter(|t| !have(t)).copied().collect();
    for tx in client.transactions(&missing).await? {
        tx_update.txs.push(Arc::new(tx));
    }

    let tip = chain.tip_height().ok_or(EngineError::NotVerified)?;
    let base = chain.base_height();
    let mut blocks: BTreeMap<u32, BlockHash> = BTreeMap::new();
    blocks.insert(genesis.height, genesis.hash);

    let confirmed: Vec<(Txid, u32)> = found.iter().filter(|(_, h)| **h > 0 && (**h as u32) <= tip).map(|(t, h)| (*t, *h as u32)).collect();
    let proofs = client.merkle_proofs(&confirmed).await?;
    let mut server_headers: HashMap<u32, crate::header::ForkHeader> = HashMap::new();
    for ((txid, height), proof) in confirmed.iter().zip(proofs) {
        if proof.block_height != *height {
            return Err(EngineError::Chain("merkle proof is for a different block"));
        }
        let (hash, header) = if *height >= base {
            (chain.hash_at(*height).unwrap(), chain.header_at(*height).unwrap().clone())
        } else {
            if !server_headers.contains_key(height) {
                let h = crate::header::ForkHeader::parse(&client.header(*height).await?)?;
                server_headers.insert(*height, h);
            }
            let h = server_headers[height].clone();
            (h.block_hash(), h)
        };
        if !verify_merkle(txid, &proof, &header.merkle_root) {
            return Err(EngineError::Chain("merkle proof does not match block"));
        }
        let block_id = BlockId { height: *height, hash: BlockHash::from_byte_array(hash) };
        blocks.insert(*height, block_id.hash);
        tx_update.anchors.insert((ConfirmationBlockTime { block_id, confirmation_time: header.time() as u64 }, *txid));
    }
    for (txid, h) in found {
        if *h <= 0 || (*h as u32) > tip {
            tx_update.seen_ats.insert((*txid, now));
        }
    }
    for b in existing_chain {
        if b.height >= base {
            if let Some(h) = chain.hash_at(b.height) {
                blocks.insert(b.height, BlockHash::from_byte_array(h));
            }
        } else {
            blocks.entry(b.height).or_insert(b.hash);
        }
    }
    blocks.insert(tip, BlockHash::from_byte_array(chain.tip_hash().unwrap()));
    let cp = CheckPoint::from_block_ids(blocks.into_iter().map(|(height, hash)| BlockId { height, hash }))
        .map_err(|_| EngineError::Chain("could not build checkpoint"))?;
    Ok(Update { tx_update, chain: Some(cp), ..Default::default() })
}

pub async fn sync_wallet(client: &ElectrumClient, chain: &HeaderChain, wallet: &mut Wallet, now: u64) -> Result<(), EngineError> {
    let mut found: HashMap<Txid, i64> = HashMap::new();
    let ext = scan_keychain(client, wallet, KeychainKind::External, STOP_GAP, &mut found).await?;
    let int = scan_keychain(client, wallet, KeychainKind::Internal, STOP_GAP, &mut found).await?;
    let genesis = BlockId { height: 0, hash: wallet.local_chain().genesis_hash() };
    let existing: Vec<BlockId> = wallet.local_chain().iter_checkpoints().map(|cp| cp.block_id()).collect();
    let graph = wallet.tx_graph();
    let have = |t: &Txid| graph.get_tx(*t).is_some();
    let mut update = fetch_confirmed_update(client, chain, genesis, &existing, &found, &have, now).await?;
    if let Some(i) = ext {
        update.last_active_indices.insert(KeychainKind::External, i);
    }
    if let Some(i) = int {
        update.last_active_indices.insert(KeychainKind::Internal, i);
    }
    wallet.apply_update(update).map_err(|_| EngineError::Chain("update does not connect to wallet chain"))
}
```

- [ ] **Step 4: Run tests**

Run: `cargo test -p wren-engine wallet_sync_tests`
Expected: PASS (4 tests).

- [ ] **Step 5: Commit**

```bash
git add core/wren-engine/src/sync.rs
git commit -m "feat(engine): gap-limit script sync with txid and merkle verification into BDK"
```

---

### Task 14: Payments — fee estimates, build and review, sign and broadcast, speed up

**Files:**
- Create: `core/wren-engine/src/payment.rs`
- Modify: `core/wren-engine/src/lib.rs` (add `pub mod payment;`), `core/wren-engine/src/testutil.rs` (add `funded_wallet`)
- Test: inline tests in `payment.rs`

**Interfaces:**
- Consumes: `WrenWallet::{bdk, bdk_mut, secret_key_for, persist}` (Task 12), `sign::{sign_p2wpkh_unified, verify_ecdsa_unified, p2wpkh_script_code}` (Task 8), `ElectrumClient::{estimate_fee, relay_fee, broadcast}` (Task 9), `ChainParams` (Task 6).
- Produces:
```rust
pub struct FeeEstimates { pub slow: FeeRate, pub normal: FeeRate, pub fast: FeeRate }
pub enum PaymentAmount { Exact(Amount), SendAll }
#[derive(Clone, Debug)]
pub struct ReviewSummary { pub review_id: u64, pub recipient: String, pub amount: Amount, pub fee: Amount,
    pub fee_rate: FeeRate, pub change: Amount, pub inputs: usize, pub replay_safe: bool }
#[derive(Default)] pub struct PendingPayments; // review_id -> Psbt, in memory only
pub async fn fee_estimates(client: &ElectrumClient) -> Result<FeeEstimates, EngineError>;
pub fn build_payment(w: &mut WrenWallet, params: &ChainParams, tip: u32, pending: &mut PendingPayments,
    recipient: &Address, amount: PaymentAmount, fee_rate: FeeRate) -> Result<ReviewSummary, EngineError>;
pub fn bump_fee(w: &mut WrenWallet, pending: &mut PendingPayments, txid: Txid, fee_rate: FeeRate) -> Result<ReviewSummary, EngineError>;
pub(crate) fn sign_pending(w: &WrenWallet, psbt: &Psbt) -> Result<Transaction, EngineError>;
pub async fn sign_and_broadcast(w: &mut WrenWallet, pending: &mut PendingPayments, review_id: u64,
    client: &ElectrumClient, now: u64) -> Result<Txid, EngineError>;
// testutil.rs addition (tests only)
pub async fn funded_wallet(params_tweak: impl FnOnce(&mut ChainParams), coins: &[(u64, u32, bool)]) -> Funded;
pub struct Funded { pub dir: tempfile::TempDir, pub wallet: WrenWallet, pub world: Arc<Mutex<MockWorld>>, pub chain: HeaderChain, pub params: ChainParams }
```
`coins` entries are `(sats, confirmation_height, is_coinbase)`, each paid to a fresh external address of the `ABANDON` wallet on a 250-block synthetic chain (heights 100..349, BLAKE2b at 105).

- [ ] **Step 1: Add `funded_wallet` to `testutil.rs`**

```rust
use crate::headerchain::HeaderChain;
use crate::params::ChainParams;
use crate::wallet::WrenWallet;
use bdk_wallet::bitcoin::{absolute::LockTime, hashes::Hash, transaction::Version, Amount, Network, OutPoint, Sequence, TxIn, TxOut};
use bdk_wallet::KeychainKind;

pub const ABANDON: &str = "abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon about";

pub struct Funded {
    pub dir: tempfile::TempDir,
    pub wallet: WrenWallet,
    pub world: Arc<Mutex<MockWorld>>,
    pub chain: HeaderChain,
    pub params: ChainParams,
}

pub async fn funded_wallet(params_tweak: impl FnOnce(&mut ChainParams), coins: &[(u64, u32, bool)]) -> Funded {
    let dir = tempfile::tempdir().unwrap();
    let mut wallet = WrenWallet::create(&dir.path().join("w.sqlite"), Network::Regtest, ABANDON, "").unwrap();
    let mut tc = TestChain::new(100, 250, 105, 1);
    let mut txs = Vec::new();
    for (i, (sats, height, coinbase)) in coins.iter().enumerate() {
        let spk = wallet.bdk_mut().reveal_next_address(KeychainKind::External).script_pubkey();
        let input = if *coinbase {
            TxIn { previous_output: OutPoint::null(), script_sig: bdk_wallet::bitcoin::ScriptBuf::from_bytes(vec![3, *height as u8, (*height >> 8) as u8, (*height >> 16) as u8]), sequence: Sequence::MAX, ..Default::default() }
        } else {
            TxIn { previous_output: OutPoint::new(bdk_wallet::bitcoin::Txid::from_byte_array([i as u8 + 1; 32]), 0), ..Default::default() }
        };
        let tx = Transaction { version: Version::TWO, lock_time: LockTime::ZERO, input: vec![input], output: vec![TxOut { value: Amount::from_sat(*sats), script_pubkey: spk.clone() }] };
        tc.set_merkle_root(*height, tx.compute_txid().to_byte_array());
        txs.push((tx, spk, *height));
    }
    wallet.persist().unwrap();
    let world = MockWorld::new(tc.clone());
    {
        let mut w = world.lock().unwrap();
        for (tx, spk, h) in &txs {
            w.txs.insert(tx.compute_txid(), tx.clone());
            w.add_history(spk, tx.compute_txid(), *h as i64);
            w.merkle.insert(tx.compute_txid(), (*h, 0, vec![]));
        }
    }
    let mut params = tc.params();
    params_tweak(&mut params);
    let mut chain = HeaderChain::new(params.clone());
    let client = crate::electrum::ElectrumClient::connect(serve(world.clone())).await.unwrap();
    crate::sync::sync_headers(&client, &mut chain).await.unwrap();
    crate::sync::sync_wallet(&client, &chain, wallet.bdk_mut(), 1_700_000_000).await.unwrap();
    wallet.persist().unwrap();
    Funded { dir, wallet, world, chain, params }
}
```
Each coin must be at a distinct height (one transaction per synthetic block).

- [ ] **Step 2: Write the failing tests**

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::electrum::ElectrumClient;
    use crate::testutil::{funded_wallet, serve};
    use bdk_wallet::bitcoin::{Address, Network};
    use std::str::FromStr;

    fn dest() -> Address {
        Address::from_str("bcrt1qw508d6qejxtdg4y5r3zarvary0c5xw7kygt080").unwrap().require_network(Network::Regtest).unwrap()
    }
    fn rate(sat_vb: u64) -> FeeRate {
        FeeRate::from_sat_per_vb(sat_vb).unwrap()
    }

    #[tokio::test]
    async fn every_input_is_marked_0x21_and_rbf() {
        let mut f = funded_wallet(|_| {}, &[(50_000, 110, false), (30_000, 111, false)]).await;
        let mut pending = PendingPayments::default();
        let r = build_payment(&mut f.wallet, &f.params, 349, &mut pending, &dest(), PaymentAmount::Exact(Amount::from_sat(60_000)), rate(2)).unwrap();
        assert!(r.replay_safe);
        let psbt = pending.peek(r.review_id).unwrap();
        for (inp, txin) in psbt.inputs.iter().zip(&psbt.unsigned_tx.input) {
            assert_eq!(inp.sighash_type.unwrap().to_u32(), 0x21);
            assert_eq!(txin.sequence, Sequence::ENABLE_RBF_NO_LOCKTIME);
        }
    }

    #[tokio::test]
    async fn signed_payment_verifies_and_is_broadcast() {
        let mut f = funded_wallet(|_| {}, &[(50_000, 110, false)]).await;
        let mut pending = PendingPayments::default();
        let r = build_payment(&mut f.wallet, &f.params, 349, &mut pending, &dest(), PaymentAmount::Exact(Amount::from_sat(20_000)), rate(2)).unwrap();
        let client = ElectrumClient::connect(serve(f.world.clone())).await.unwrap();
        let txid = sign_and_broadcast(&mut f.wallet, &mut pending, r.review_id, &client, 1_700_000_100).await.unwrap();
        let sent = f.world.lock().unwrap().broadcasts[0].clone();
        assert_eq!(sent.compute_txid(), txid);
        for txin in &sent.input {
            assert_eq!(*txin.witness.iter().next().unwrap().last().unwrap(), 0x21);
        }
        assert!(pending.peek(r.review_id).is_none(), "a review is single-use");
    }

    #[tokio::test]
    async fn send_all_has_no_change() {
        let mut f = funded_wallet(|_| {}, &[(50_000, 110, false), (30_000, 111, false)]).await;
        let mut pending = PendingPayments::default();
        let r = build_payment(&mut f.wallet, &f.params, 349, &mut pending, &dest(), PaymentAmount::SendAll, rate(1)).unwrap();
        assert_eq!(r.change, Amount::ZERO);
        assert_eq!(pending.peek(r.review_id).unwrap().unsigned_tx.output.len(), 1);
        assert_eq!(r.amount + r.fee, Amount::from_sat(80_000));
    }

    #[tokio::test]
    async fn refuses_fee_below_floor_and_insufficient_funds() {
        let mut f = funded_wallet(|_| {}, &[(10_000, 110, false)]).await;
        let mut pending = PendingPayments::default();
        assert!(build_payment(&mut f.wallet, &f.params, 349, &mut pending, &dest(), PaymentAmount::Exact(Amount::from_sat(1_000)), FeeRate::from_sat_per_kwu(200)).is_err());
        assert!(build_payment(&mut f.wallet, &f.params, 349, &mut pending, &dest(), PaymentAmount::Exact(Amount::from_sat(20_000)), rate(1)).is_err());
    }

    #[tokio::test]
    async fn long_maturity_coinbase_is_not_spendable() {
        // Coinbase at 110, tip 349: 240 confirmations — mature under the 100 rule, immature under 6,480.
        let mut f = funded_wallet(|p| { p.long_maturity_start = 105; p.long_maturity_end = 1_000_000; }, &[(50_000, 110, true)]).await;
        let mut pending = PendingPayments::default();
        assert!(build_payment(&mut f.wallet, &f.params, 349, &mut pending, &dest(), PaymentAmount::Exact(Amount::from_sat(10_000)), rate(1)).is_err());
        // Same coin with the rule disabled is spendable (proves the exclusion is the long-maturity rule).
        let mut g = funded_wallet(|_| {}, &[(50_000, 110, true)]).await;
        assert!(build_payment(&mut g.wallet, &g.params, 349, &mut pending, &dest(), PaymentAmount::Exact(Amount::from_sat(10_000)), rate(1)).is_ok());
    }

    #[tokio::test]
    async fn bump_fee_raises_fee_and_keeps_0x21() {
        let mut f = funded_wallet(|_| {}, &[(50_000, 110, false)]).await;
        let mut pending = PendingPayments::default();
        let r = build_payment(&mut f.wallet, &f.params, 349, &mut pending, &dest(), PaymentAmount::Exact(Amount::from_sat(20_000)), rate(1)).unwrap();
        let client = ElectrumClient::connect(serve(f.world.clone())).await.unwrap();
        let txid = sign_and_broadcast(&mut f.wallet, &mut pending, r.review_id, &client, 1_700_000_100).await.unwrap();
        let b = bump_fee(&mut f.wallet, &mut pending, txid, rate(5)).unwrap();
        assert!(b.fee > r.fee);
        for inp in &pending.peek(b.review_id).unwrap().inputs {
            assert_eq!(inp.sighash_type.unwrap().to_u32(), 0x21);
        }
    }
}
```

- [ ] **Step 3: Run to verify failure**

Run: `cargo test -p wren-engine payment::`
Expected: FAIL to compile — module missing.

- [ ] **Step 4: Implement**

```rust
//! Payments from Wren's own wallet. Every input is marked and signed SIGHASH_ALL|UNIFIED.
use crate::electrum::ElectrumClient;
use crate::error::EngineError;
use crate::params::ChainParams;
use crate::sighash::{ScriptType, SIGHASH_ALL_UNIFIED};
use crate::sign::{p2wpkh_script_code, sign_p2wpkh_unified, verify_ecdsa_unified};
use crate::wallet::WrenWallet;
use bdk_wallet::bitcoin::psbt::PsbtSighashType;
use bdk_wallet::bitcoin::secp256k1::{PublicKey, Secp256k1};
use bdk_wallet::bitcoin::{Address, Amount, FeeRate, OutPoint, Psbt, Sequence, Transaction, TxOut, Txid};
use bdk_wallet::chain::ChainPosition;
use std::collections::HashMap;

pub struct FeeEstimates {
    pub slow: FeeRate,
    pub normal: FeeRate,
    pub fast: FeeRate,
}

pub enum PaymentAmount {
    Exact(Amount),
    SendAll,
}

#[derive(Clone, Debug)]
pub struct ReviewSummary {
    pub review_id: u64,
    pub recipient: String,
    pub amount: Amount,
    pub fee: Amount,
    pub fee_rate: FeeRate,
    pub change: Amount,
    pub inputs: usize,
    pub replay_safe: bool,
}

#[derive(Default)]
pub struct PendingPayments {
    next_id: u64,
    map: HashMap<u64, Psbt>,
}

impl PendingPayments {
    fn insert(&mut self, psbt: Psbt) -> u64 {
        self.next_id += 1;
        self.map.clear(); // only the latest review is valid
        self.map.insert(self.next_id, psbt);
        self.next_id
    }
    pub fn peek(&self, id: u64) -> Option<&Psbt> {
        self.map.get(&id)
    }
    fn take(&mut self, id: u64) -> Option<Psbt> {
        self.map.remove(&id)
    }
}

fn floor() -> FeeRate {
    FeeRate::from_sat_per_vb_unchecked(1)
}

pub async fn fee_estimates(client: &ElectrumClient) -> Result<FeeEstimates, EngineError> {
    let min = client.relay_fee().await?.max(floor());
    let slow = client.estimate_fee(12).await?.unwrap_or(min).max(min);
    let normal = client.estimate_fee(6).await?.unwrap_or(slow).max(slow);
    let fast = client.estimate_fee(2).await?.unwrap_or(normal).max(normal);
    Ok(FeeEstimates { slow, normal, fast })
}

/// Coinbase outputs still immature under the fork's long-maturity rule (Knots tx_verify.cpp:199).
fn immature_coinbase(w: &WrenWallet, params: &ChainParams, tip: u32) -> Vec<OutPoint> {
    let spend_height = tip + 1;
    w.bdk().list_unspent().filter_map(|u| {
        let wtx = w.bdk().get_tx(u.outpoint.txid)?;
        if !wtx.tx_node.tx.is_coinbase() {
            return None;
        }
        let ChainPosition::Confirmed { anchor, .. } = &wtx.chain_position else { return Some(u.outpoint) };
        let h = anchor.block_id.height;
        let required = if h >= params.long_maturity_start && spend_height < params.long_maturity_end { 6_480 } else { 100 };
        (spend_height.saturating_sub(h) < required).then_some(u.outpoint)
    }).collect()
}

fn summarize(w: &WrenWallet, psbt: &Psbt, id: u64, fee_rate: FeeRate) -> Result<ReviewSummary, EngineError> {
    let network = w.bdk().network();
    let mut recipient = String::new();
    let (mut amount, mut change) = (Amount::ZERO, Amount::ZERO);
    for o in &psbt.unsigned_tx.output {
        if w.bdk().is_mine(o.script_pubkey.clone()) {
            change += o.value;
        } else {
            amount += o.value;
            recipient = Address::from_script(&o.script_pubkey, network).map(|a| a.to_string()).unwrap_or_default();
        }
    }
    Ok(ReviewSummary {
        review_id: id,
        recipient,
        amount,
        fee: psbt.fee().map_err(|e| EngineError::Wallet(e.to_string()))?,
        fee_rate,
        change,
        inputs: psbt.inputs.len(),
        replay_safe: psbt.inputs.iter().all(|i| i.sighash_type.map(|s| s.to_u32()) == Some(SIGHASH_ALL_UNIFIED)),
    })
}

pub fn build_payment(w: &mut WrenWallet, params: &ChainParams, tip: u32, pending: &mut PendingPayments, recipient: &Address, amount: PaymentAmount, fee_rate: FeeRate) -> Result<ReviewSummary, EngineError> {
    if fee_rate < floor() {
        return Err(EngineError::Input("fee rate is below 1 sat/vB"));
    }
    let exclude = immature_coinbase(w, params, tip);
    let mut b = w.bdk_mut().build_tx();
    b.fee_rate(fee_rate)
        .sighash(PsbtSighashType::from_u32(SIGHASH_ALL_UNIFIED))
        .set_exact_sequence(Sequence::ENABLE_RBF_NO_LOCKTIME)
        .unspendable(exclude);
    match amount {
        PaymentAmount::Exact(a) => {
            b.add_recipient(recipient.script_pubkey(), a);
        }
        PaymentAmount::SendAll => {
            b.drain_wallet().drain_to(recipient.script_pubkey());
        }
    }
    let psbt = b.finish().map_err(|e| EngineError::Wallet(e.to_string()))?;
    let id = pending.insert(psbt);
    summarize(w, pending.peek(id).unwrap(), id, fee_rate)
}

pub fn bump_fee(w: &mut WrenWallet, pending: &mut PendingPayments, txid: Txid, fee_rate: FeeRate) -> Result<ReviewSummary, EngineError> {
    let mut b = w.bdk_mut().build_fee_bump(txid).map_err(|e| EngineError::Wallet(e.to_string()))?;
    b.fee_rate(fee_rate)
        .sighash(PsbtSighashType::from_u32(SIGHASH_ALL_UNIFIED))
        .set_exact_sequence(Sequence::ENABLE_RBF_NO_LOCKTIME);
    let psbt = b.finish().map_err(|e| EngineError::Wallet(e.to_string()))?;
    let id = pending.insert(psbt);
    summarize(w, pending.peek(id).unwrap(), id, fee_rate)
}

pub(crate) fn sign_pending(w: &WrenWallet, psbt: &Psbt) -> Result<Transaction, EngineError> {
    let secp = Secp256k1::new();
    let spent: Vec<TxOut> = psbt.inputs.iter()
        .map(|i| i.witness_utxo.clone().ok_or(EngineError::Signing("missing previous output")))
        .collect::<Result<_, _>>()?;
    if psbt.inputs.iter().any(|i| i.sighash_type.map(|s| s.to_u32()) != Some(SIGHASH_ALL_UNIFIED)) {
        return Err(EngineError::Signing("an input is not marked replay-safe"));
    }
    let mut tx = psbt.unsigned_tx.clone();
    for i in 0..tx.input.len() {
        let sk = w.secret_key_for(&psbt.inputs[i])?;
        sign_p2wpkh_unified(&secp, &mut tx, i, &spent, &sk)?;
    }
    for i in 0..tx.input.len() {
        let wit: Vec<Vec<u8>> = tx.input[i].witness.iter().map(|x| x.to_vec()).collect();
        let pk = PublicKey::from_slice(&wit[1]).map_err(|_| EngineError::Signing("bad public key"))?;
        verify_ecdsa_unified(&secp, &tx, i, &spent, ScriptType::WitnessV0, &p2wpkh_script_code(&pk), &wit[0], &pk)?;
    }
    Ok(tx)
}

pub async fn sign_and_broadcast(w: &mut WrenWallet, pending: &mut PendingPayments, review_id: u64, client: &ElectrumClient, now: u64) -> Result<Txid, EngineError> {
    let psbt = pending.take(review_id).ok_or(EngineError::Input("this payment review has expired; review it again"))?;
    let tx = sign_pending(w, &psbt)?;
    let txid = client.broadcast(&tx).await?;
    w.bdk_mut().apply_unconfirmed_txs([(tx, now)]);
    w.persist()?;
    Ok(txid)
}
```
Add `pub mod payment;` to `lib.rs`.

- [ ] **Step 5: Run tests**

Run: `cargo test -p wren-engine payment::`
Expected: PASS (6 tests).

- [ ] **Step 6: Commit**

```bash
git add core/wren-engine/src/payment.rs core/wren-engine/src/testutil.rs core/wren-engine/src/lib.rs
git commit -m "feat(engine): payments with 0x21 on every input, RBF, send-all, long-maturity guard"
```

---

### Task 15: `xbt:` payment links

**Files:**
- Create: `core/wren-engine/src/uri.rs`
- Modify: `core/wren-engine/src/lib.rs` (add `pub mod uri;`)
- Test: inline tests in `uri.rs`

**Interfaces:**
- Consumes: `EngineError`.
- Produces:
```rust
#[derive(Clone, Debug, PartialEq, Eq)]
pub struct PaymentLink { pub address: Address, pub amount: Option<Amount>, pub label: Option<String> }
pub fn parse_payment_link(s: &str, network: Network) -> Result<PaymentLink, EngineError>; // accepts xbt: links and bare addresses
pub fn make_payment_link(address: &Address, amount: Option<Amount>) -> String;
pub fn format_xbt(a: Amount) -> String;  // "0.001", "1", "21.5"
```

- [ ] **Step 1: Write the failing tests**

```rust
#[cfg(test)]
mod tests {
    use super::*;
    const ADDR: &str = "bc1qcr8te4kr609gcawutmrza0j4xv80jy8z306fyu";

    #[test]
    fn rejects_bitcoin_links_in_any_case() {
        for s in [format!("bitcoin:{ADDR}"), format!("BITCOIN:{ADDR}?amount=1")] {
            match parse_payment_link(&s, Network::Bitcoin) {
                Err(EngineError::Input(m)) => assert!(m.contains("not an XBT link")),
                other => panic!("expected rejection, got {other:?}"),
            }
        }
    }

    #[test]
    fn parses_xbt_links_bare_addresses_and_qr_uppercase() {
        let l = parse_payment_link(&format!("xbt:{ADDR}?amount=0.0015&label=Coffee%20beans"), Network::Bitcoin).unwrap();
        assert_eq!(l.amount, Some(Amount::from_sat(150_000)));
        assert_eq!(l.label.as_deref(), Some("Coffee beans"));
        assert_eq!(parse_payment_link(ADDR, Network::Bitcoin).unwrap().amount, None);
        let upper = format!("XBT:{}", ADDR.to_uppercase());
        assert_eq!(parse_payment_link(&upper, Network::Bitcoin).unwrap().address.to_string(), ADDR);
    }

    #[test]
    fn rejects_wrong_network_bad_amounts_and_unknown_required_params() {
        assert!(parse_payment_link("xbt:tb1qw508d6qejxtdg4y5r3zarvary0c5xw7kxpjzsx", Network::Bitcoin).is_err());
        assert!(parse_payment_link(&format!("xbt:{ADDR}?amount=-1"), Network::Bitcoin).is_err());
        assert!(parse_payment_link(&format!("xbt:{ADDR}?amount=0.000000001"), Network::Bitcoin).is_err());
        assert!(parse_payment_link(&format!("xbt:{ADDR}?req-escrow=1"), Network::Bitcoin).is_err());
    }

    #[test]
    fn make_and_parse_round_trip() {
        let a = Address::from_str(ADDR).unwrap().require_network(Network::Bitcoin).unwrap();
        let s = make_payment_link(&a, Some(Amount::from_sat(2_150_000_000)));
        assert_eq!(s, format!("xbt:{ADDR}?amount=21.5"));
        assert_eq!(parse_payment_link(&s, Network::Bitcoin).unwrap().amount, Some(Amount::from_sat(2_150_000_000)));
        assert_eq!(format_xbt(Amount::from_sat(100_000)), "0.001");
        assert_eq!(format_xbt(Amount::from_sat(100_000_000)), "1");
    }
}
```

- [ ] **Step 2: Run to verify failure**

Run: `cargo test -p wren-engine uri::`
Expected: FAIL to compile — module missing.

- [ ] **Step 3: Implement**

```rust
//! `xbt:` payment links. `bitcoin:` links are refused so a legacy-chain request can never be paid here
//! (and Wren never emits one that a legacy wallet could open).
use crate::error::EngineError;
use bdk_wallet::bitcoin::{Address, Amount, Denomination, Network};
use std::str::FromStr;

#[derive(Clone, Debug, PartialEq, Eq)]
pub struct PaymentLink {
    pub address: Address,
    pub amount: Option<Amount>,
    pub label: Option<String>,
}

fn has_prefix_ci(s: &str, p: &str) -> bool {
    s.len() >= p.len() && s[..p.len()].eq_ignore_ascii_case(p)
}

fn percent_decode(s: &str) -> Result<String, EngineError> {
    let b = s.as_bytes();
    let mut out = Vec::with_capacity(b.len());
    let mut i = 0;
    while i < b.len() {
        if b[i] == b'%' {
            let h = s.get(i + 1..i + 3).ok_or(EngineError::Input("bad escape in link"))?;
            out.push(u8::from_str_radix(h, 16).map_err(|_| EngineError::Input("bad escape in link"))?);
            i += 3;
        } else {
            out.push(if b[i] == b'+' { b' ' } else { b[i] });
            i += 1;
        }
    }
    String::from_utf8(out).map_err(|_| EngineError::Input("bad text in link"))
}

pub fn parse_payment_link(s: &str, network: Network) -> Result<PaymentLink, EngineError> {
    let t = s.trim();
    if has_prefix_ci(t, "bitcoin:") {
        return Err(EngineError::Input("this is a bitcoin: link, not an XBT link"));
    }
    let body = if has_prefix_ci(t, "xbt:") { &t[4..] } else { t };
    let (addr, query) = match body.split_once('?') {
        Some((a, q)) => (a, Some(q)),
        None => (body, None),
    };
    let address = Address::from_str(addr)
        .map_err(|_| EngineError::Input("not a valid address"))?
        .require_network(network)
        .map_err(|_| EngineError::Input("address is for a different network"))?;
    let (mut amount, mut label) = (None, None);
    for pair in query.unwrap_or("").split('&').filter(|p| !p.is_empty()) {
        let (k, v) = pair.split_once('=').unwrap_or((pair, ""));
        match k {
            "amount" => amount = Some(Amount::from_str_in(v, Denomination::Bitcoin).map_err(|_| EngineError::Input("invalid amount in link"))?),
            "label" => label = Some(percent_decode(v)?),
            k if k.starts_with("req-") => return Err(EngineError::Input("this payment request needs a feature Wren does not support")),
            _ => {}
        }
    }
    Ok(PaymentLink { address, amount, label })
}

pub fn format_xbt(a: Amount) -> String {
    let s = a.to_sat();
    let (whole, frac) = (s / 100_000_000, s % 100_000_000);
    if frac == 0 {
        whole.to_string()
    } else {
        format!("{whole}.{frac:08}").trim_end_matches('0').to_string()
    }
}

pub fn make_payment_link(address: &Address, amount: Option<Amount>) -> String {
    match amount {
        Some(a) => format!("xbt:{address}?amount={}", format_xbt(a)),
        None => format!("xbt:{address}"),
    }
}
```
Add `pub mod uri;` to `lib.rs`.

- [ ] **Step 4: Run tests**

Run: `cargo test -p wren-engine uri::`
Expected: PASS (4 tests).

- [ ] **Step 5: Commit**

```bash
git add core/wren-engine/src/uri.rs core/wren-engine/src/lib.rs
git commit -m "feat(engine): xbt: payment links; refuse bitcoin: links"
```

---

### Task 16: Animated QR (BC-UR) and claim xpub import

**Files:**
- Create: `core/wren-engine/src/ur.rs`, `core/wren-engine/src/xpub_import.rs`
- Modify: `core/wren-engine/src/lib.rs` (add `pub mod ur; pub mod xpub_import;`)
- Test: inline tests in both files

**Interfaces:**
- Consumes: `EngineError`, crates `ur` 0.5 (`ur::Encoder::new(msg, max_len, type)`, `next_part()`, `fragment_count()`, `ur::decode`, `ur::Decoder::{receive, complete, message}`) and `ciborium`.
- Produces:
```rust
// ur.rs
pub const QR_FRAGMENT_LEN: usize = 200;
pub fn encode_psbt_frames(psbt: &[u8]) -> Result<Vec<String>, EngineError>; // uppercase UR:CRYPTO-PSBT frames, 2x fountain parts
pub struct QrCollector;
impl QrCollector {
    pub fn new() -> Self;
    pub fn receive(&mut self, frame: &str) -> Result<bool, EngineError>; // true when complete
    pub fn finish(self) -> Result<ScannedPayload, EngineError>;
}
#[derive(Clone, Debug, PartialEq, Eq)]
pub enum ScannedPayload { Text(String), Ur { ur_type: String, cbor: Vec<u8> } }
pub fn psbt_from_payload(p: &ScannedPayload) -> Result<Vec<u8>, EngineError>; // crypto-psbt UR or base64 text
// xpub_import.rs
#[derive(Clone, Copy, Debug, PartialEq, Eq, Hash)]
pub enum ClaimScriptType { P2wpkh, P2shP2wpkh, P2pkh, P2trKeyPath }
#[derive(Clone, Debug, PartialEq, Eq)]
pub struct ImportedAccount { pub script_type: ClaimScriptType, pub fingerprint: Fingerprint, pub path: DerivationPath, pub xpub: Xpub }
pub fn import_accounts(p: &ScannedPayload, network: Network) -> Result<Vec<ImportedAccount>, EngineError>;
```
Accepted xpub formats (research: SeedSigner `settings_definition.py`): UR `crypto-account` (animated default), Specter `[fingerprint/derivation]xpub|ypub|zpub`, and single-key output descriptors (`wpkh(…)`, `sh(wpkh(…))`, `pkh(…)`, `tr(…)`). A key **without** origin information is refused, because the signer finds its keys in the PSBT by master fingerprint and path.

- [ ] **Step 1: Write the failing tests**

`ur.rs` tests:
```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn psbt_round_trips_through_animated_frames_out_of_order() {
        let psbt: Vec<u8> = b"psbt\xff".iter().copied().chain((0..3000u32).map(|i| (i % 251) as u8)).collect();
        let frames = encode_psbt_frames(&psbt).unwrap();
        assert!(frames.len() > 10 && frames[0].starts_with("UR:CRYPTO-PSBT/"));
        let mut c = QrCollector::new();
        let mut done = false;
        for f in frames.iter().rev() {
            done = c.receive(f).unwrap();
            if done { break; }
        }
        assert!(done);
        assert_eq!(psbt_from_payload(&c.finish().unwrap()).unwrap(), psbt);
    }

    #[test]
    fn accepts_base64_psbt_text_and_rejects_non_psbt() {
        use base64::Engine;
        let psbt = b"psbt\xff\x01\x02".to_vec();
        let text = base64::engine::general_purpose::STANDARD.encode(&psbt);
        let mut c = QrCollector::new();
        assert!(c.receive(&text).unwrap());
        assert_eq!(psbt_from_payload(&c.finish().unwrap()).unwrap(), psbt);
        assert!(psbt_from_payload(&ScannedPayload::Text("hello".into())).is_err());
    }
}
```

`xpub_import.rs` tests (the `abandon…about` wallet's BIP84 account, from the BIP84 test vectors: master fingerprint `73c5da0a`):
```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::ur::ScannedPayload;
    use ciborium::value::{Integer, Value};
    use std::str::FromStr;

    const ZPUB: &str = "zpub6rFR7y4Q2AijBEqTUquhVz398htDFrtymD9xYYfG1m4wAcvPhXNfE3EfH1r1ADqtfSdVCToUG868RvUUkgDKf31mGDtKsAYz2oz2AGutZYs";

    fn expected_xpub() -> Xpub {
        let data = bdk_wallet::bitcoin::base58::decode_check(ZPUB).unwrap();
        let mut d = data.clone();
        d[..4].copy_from_slice(&[0x04, 0x88, 0xb2, 0x1e]);
        Xpub::decode(&d).unwrap()
    }

    #[test]
    fn specter_format_with_zpub() {
        let a = import_accounts(&ScannedPayload::Text(format!("[73c5da0a/84h/0h/0h]{ZPUB}")), Network::Bitcoin).unwrap();
        assert_eq!(a.len(), 1);
        assert_eq!(a[0].script_type, ClaimScriptType::P2wpkh);
        assert_eq!(a[0].fingerprint.to_string(), "73c5da0a");
        assert_eq!(a[0].path.to_string(), "84'/0'/0'");
        assert_eq!(a[0].xpub, expected_xpub());
    }

    #[test]
    fn descriptor_formats_pick_script_type() {
        let x = expected_xpub();
        for (d, t) in [
            (format!("wpkh([73c5da0a/84'/0'/0']{x}/0/*)"), ClaimScriptType::P2wpkh),
            (format!("sh(wpkh([73c5da0a/49'/0'/0']{x}/0/*))"), ClaimScriptType::P2shP2wpkh),
            (format!("pkh([73c5da0a/44'/0'/0']{x}/<0;1>/*)#abcdefgh"), ClaimScriptType::P2pkh),
            (format!("tr([73c5da0a/86'/0'/0']{x}/0/*)"), ClaimScriptType::P2trKeyPath),
        ] {
            assert_eq!(import_accounts(&ScannedPayload::Text(d), Network::Bitcoin).unwrap()[0].script_type, t);
        }
    }

    #[test]
    fn refuses_keys_without_origin_and_testnet_keys() {
        assert!(import_accounts(&ScannedPayload::Text(ZPUB.into()), Network::Bitcoin).is_err());
        use bdk_wallet::bitcoin::bip32::Xpriv;
        use bdk_wallet::keys::bip39::{Language, Mnemonic};
        let secp = bdk_wallet::bitcoin::secp256k1::Secp256k1::new();
        let seed = Mnemonic::parse_in(Language::English, crate::testutil::ABANDON).unwrap().to_seed("");
        let m = Xpriv::new_master(Network::Testnet, &seed).unwrap();
        let tpub = Xpub::from_priv(&secp, &m.derive_priv(&secp, &DerivationPath::from_str("m/84'/1'/0'").unwrap()).unwrap());
        let text = format!("[73c5da0a/84h/1h/0h]{tpub}");
        assert!(import_accounts(&ScannedPayload::Text(text), Network::Bitcoin).is_err());
    }

    #[test]
    fn ur_crypto_account_bcr_2020_015() {
        let x = expected_xpub();
        let path = |p: u64| Value::Array(vec![
            Value::Integer(Integer::from(p)), Value::Bool(true),
            Value::Integer(Integer::from(0u64)), Value::Bool(true),
            Value::Integer(Integer::from(0u64)), Value::Bool(true)]);
        let hdkey = |p: u64| Value::Tag(303, Box::new(Value::Map(vec![
            (Value::Integer(3.into()), Value::Bytes(x.public_key.serialize().to_vec())),
            (Value::Integer(4.into()), Value::Bytes(x.chain_code.as_bytes().to_vec())),
            (Value::Integer(6.into()), Value::Tag(304, Box::new(Value::Map(vec![
                (Value::Integer(1.into()), path(p)),
                (Value::Integer(2.into()), Value::Integer(Integer::from(0x73c5da0au64))),
            ])))),
            (Value::Integer(8.into()), Value::Integer(Integer::from(u32::from_be_bytes(*x.parent_fingerprint.as_bytes()) as u64))),
        ])));
        let account = Value::Map(vec![
            (Value::Integer(1.into()), Value::Integer(Integer::from(0x73c5da0au64))),
            (Value::Integer(2.into()), Value::Array(vec![
                Value::Tag(404, Box::new(hdkey(84))),
                Value::Tag(400, Box::new(Value::Tag(404, Box::new(hdkey(49))))),
            ])),
        ]);
        let mut cbor = Vec::new();
        ciborium::into_writer(&account, &mut cbor).unwrap();
        let a = import_accounts(&ScannedPayload::Ur { ur_type: "crypto-account".into(), cbor }, Network::Bitcoin).unwrap();
        assert_eq!(a.len(), 2);
        assert_eq!((a[0].script_type, a[0].path.to_string()), (ClaimScriptType::P2wpkh, "84'/0'/0'".to_string()));
        assert_eq!(a[0].xpub, x);
        assert_eq!(a[1].script_type, ClaimScriptType::P2shP2wpkh);
    }
}
```
The UR test builds the payload by hand from BCR-2020-015. A crypto-account captured from a real SeedSigner export is added as a fixture during the app plan's real-device test.

- [ ] **Step 2: Run to verify failure**

Run: `cargo test -p wren-engine ur:: xpub_import`
Expected: FAIL to compile — modules missing.

- [ ] **Step 3: Implement `ur.rs`**

```rust
//! BC-UR animated QR codes (SeedSigner speaks UR v2 `crypto-psbt` and `crypto-account`).
use crate::error::EngineError;
use base64::Engine;
use ciborium::value::Value;

pub const QR_FRAGMENT_LEN: usize = 200;

#[derive(Clone, Debug, PartialEq, Eq)]
pub enum ScannedPayload {
    Text(String),
    Ur { ur_type: String, cbor: Vec<u8> },
}

fn qr(_: impl std::fmt::Debug) -> EngineError {
    EngineError::Input("could not read this QR code")
}

pub fn encode_psbt_frames(psbt: &[u8]) -> Result<Vec<String>, EngineError> {
    let mut cbor = Vec::new();
    ciborium::into_writer(&Value::Bytes(psbt.to_vec()), &mut cbor).map_err(qr)?;
    let mut enc = ur::Encoder::new(&cbor, QR_FRAGMENT_LEN, "crypto-psbt").map_err(qr)?;
    let n = enc.fragment_count() * 2; // extra fountain parts help cameras that miss frames
    (0..n).map(|_| enc.next_part().map(|p| p.to_uppercase()).map_err(qr)).collect()
}

pub struct QrCollector {
    decoder: ur::Decoder,
    single: Option<ScannedPayload>,
    ur_type: Option<String>,
}

impl Default for QrCollector {
    fn default() -> Self {
        Self::new()
    }
}

impl QrCollector {
    pub fn new() -> Self {
        QrCollector { decoder: ur::Decoder::default(), single: None, ur_type: None }
    }

    pub fn receive(&mut self, frame: &str) -> Result<bool, EngineError> {
        let f = frame.trim();
        let lower = f.to_lowercase();
        if !lower.starts_with("ur:") {
            self.single = Some(ScannedPayload::Text(f.to_string()));
            return Ok(true);
        }
        let ty = lower[3..].split('/').next().unwrap_or("").to_string();
        if lower.matches('/').count() >= 2 {
            self.decoder.receive(&lower).map_err(qr)?;
            self.ur_type = Some(ty);
            Ok(self.decoder.complete())
        } else {
            let (_, cbor) = ur::decode(&lower).map_err(qr)?;
            self.single = Some(ScannedPayload::Ur { ur_type: ty, cbor });
            Ok(true)
        }
    }

    pub fn finish(self) -> Result<ScannedPayload, EngineError> {
        if let Some(s) = self.single {
            return Ok(s);
        }
        let cbor = self.decoder.message().map_err(qr)?.ok_or(EngineError::Input("QR code not fully scanned yet"))?;
        Ok(ScannedPayload::Ur { ur_type: self.ur_type.unwrap_or_default(), cbor })
    }
}

pub fn psbt_from_payload(p: &ScannedPayload) -> Result<Vec<u8>, EngineError> {
    let bytes = match p {
        ScannedPayload::Ur { ur_type, cbor } if ur_type == "crypto-psbt" || ur_type == "psbt" => {
            match ciborium::from_reader::<Value, _>(cbor.as_slice()).map_err(qr)? {
                Value::Bytes(b) => b,
                Value::Tag(310, inner) => match *inner {
                    Value::Bytes(b) => b,
                    _ => return Err(EngineError::Input("QR code is not a PSBT")),
                },
                _ => return Err(EngineError::Input("QR code is not a PSBT")),
            }
        }
        ScannedPayload::Text(t) => base64::engine::general_purpose::STANDARD.decode(t.trim()).map_err(|_| EngineError::Input("QR code is not a PSBT"))?,
        _ => return Err(EngineError::Input("QR code is not a PSBT")),
    };
    if !bytes.starts_with(b"psbt\xff") {
        return Err(EngineError::Input("QR code is not a PSBT"));
    }
    Ok(bytes)
}
```

- [ ] **Step 4: Implement `xpub_import.rs`**

```rust
//! Claim input: account xpubs from a hardware signer. Never seeds.
use crate::error::EngineError;
use crate::ur::ScannedPayload;
use bdk_wallet::bitcoin::base58;
use bdk_wallet::bitcoin::bip32::{ChainCode, ChildNumber, DerivationPath, Fingerprint, Xpub};
use bdk_wallet::bitcoin::secp256k1::PublicKey;
use bdk_wallet::bitcoin::{Network, NetworkKind};
use ciborium::value::Value;
use std::str::FromStr;

#[derive(Clone, Copy, Debug, PartialEq, Eq, Hash)]
pub enum ClaimScriptType {
    P2wpkh,
    P2shP2wpkh,
    P2pkh,
    P2trKeyPath,
}

#[derive(Clone, Debug, PartialEq, Eq)]
pub struct ImportedAccount {
    pub script_type: ClaimScriptType,
    pub fingerprint: Fingerprint,
    pub path: DerivationPath,
    pub xpub: Xpub,
}

fn bad() -> EngineError {
    EngineError::Input("this is not an account key Wren can use")
}

/// xpub/ypub/zpub (and testnet equivalents) -> Xpub, plus the script type implied by the prefix.
fn parse_key(s: &str) -> Result<(Xpub, Option<ClaimScriptType>), EngineError> {
    let data = base58::decode_check(s).map_err(|_| bad())?;
    if data.len() != 78 {
        return Err(bad());
    }
    let (version, implied) = match &data[..4] {
        [0x04, 0x88, 0xb2, 0x1e] => ([0x04, 0x88, 0xb2, 0x1e], None),
        [0x04, 0x9d, 0x7c, 0xb2] => ([0x04, 0x88, 0xb2, 0x1e], Some(ClaimScriptType::P2shP2wpkh)),
        [0x04, 0xb2, 0x47, 0x46] => ([0x04, 0x88, 0xb2, 0x1e], Some(ClaimScriptType::P2wpkh)),
        [0x04, 0x35, 0x87, 0xcf] | [0x04, 0x4a, 0x52, 0x62] | [0x04, 0x5f, 0x1c, 0xf6] => ([0x04, 0x35, 0x87, 0xcf], None),
        _ => return Err(bad()),
    };
    let mut d = data.clone();
    d[..4].copy_from_slice(&version);
    Ok((Xpub::decode(&d).map_err(|_| bad())?, implied))
}

fn type_from_purpose(path: &DerivationPath) -> Option<ClaimScriptType> {
    let purpose: &[ChildNumber] = path.as_ref();
    match purpose.first()? {
        ChildNumber::Hardened { index: 84 } => Some(ClaimScriptType::P2wpkh),
        ChildNumber::Hardened { index: 49 } => Some(ClaimScriptType::P2shP2wpkh),
        ChildNumber::Hardened { index: 44 } => Some(ClaimScriptType::P2pkh),
        ChildNumber::Hardened { index: 86 } => Some(ClaimScriptType::P2trKeyPath),
        _ => None,
    }
}

fn check_network(x: &Xpub, network: Network) -> Result<(), EngineError> {
    let want = NetworkKind::from(network);
    if x.network != want {
        return Err(EngineError::Input("this key is for a different network"));
    }
    Ok(())
}

fn from_text(t: &str, network: Network) -> Result<Vec<ImportedAccount>, EngineError> {
    let t = t.trim();
    let t = t.split('#').next().unwrap_or(t);
    let (wrapper_type, inner) = if let Some(r) = t.strip_prefix("sh(wpkh(") {
        (Some(ClaimScriptType::P2shP2wpkh), r.trim_end_matches(')'))
    } else if let Some(r) = t.strip_prefix("wpkh(") {
        (Some(ClaimScriptType::P2wpkh), r.trim_end_matches(')'))
    } else if let Some(r) = t.strip_prefix("pkh(") {
        (Some(ClaimScriptType::P2pkh), r.trim_end_matches(')'))
    } else if let Some(r) = t.strip_prefix("tr(") {
        (Some(ClaimScriptType::P2trKeyPath), r.trim_end_matches(')'))
    } else {
        (None, t)
    };
    let rest = inner.strip_prefix('[').ok_or(EngineError::Input("export the key with its fingerprint and path (Specter or UR format)"))?;
    let (origin, key_and_suffix) = rest.split_once(']').ok_or_else(bad)?;
    let (fp, path) = origin.split_once('/').ok_or_else(bad)?;
    let fingerprint = Fingerprint::from_str(fp).map_err(|_| bad())?;
    let path = DerivationPath::from_str(&format!("m/{}", path.replace('h', "'"))).map_err(|_| bad())?;
    let key = key_and_suffix.split('/').next().unwrap_or("");
    let (xpub, implied) = parse_key(key)?;
    check_network(&xpub, network)?;
    let script_type = wrapper_type.or(implied).or_else(|| type_from_purpose(&path)).ok_or_else(bad)?;
    Ok(vec![ImportedAccount { script_type, fingerprint, path, xpub }])
}

fn map_get(m: &[(Value, Value)], k: u64) -> Option<&Value> {
    m.iter().find(|(key, _)| matches!(key, Value::Integer(i) if u64::try_from(*i).ok() == Some(k))).map(|(_, v)| v)
}
fn as_u64(v: &Value) -> Option<u64> {
    match v { Value::Integer(i) => u64::try_from(*i).ok(), _ => None }
}

fn hdkey_to_account(v: &Value, master: Fingerprint, script_type: ClaimScriptType, network: Network) -> Result<ImportedAccount, EngineError> {
    let Value::Tag(303, inner) = v else { return Err(bad()) };
    let Value::Map(m) = inner.as_ref() else { return Err(bad()) };
    let key = match map_get(m, 3) { Some(Value::Bytes(b)) => PublicKey::from_slice(b).map_err(|_| bad())?, _ => return Err(bad()) };
    let chain_code = match map_get(m, 4) { Some(Value::Bytes(b)) if b.len() == 32 => ChainCode::from(<[u8; 32]>::try_from(b.as_slice()).unwrap()), _ => return Err(bad()) };
    let Some(Value::Tag(304, origin)) = map_get(m, 6) else { return Err(bad()) };
    let Value::Map(om) = origin.as_ref() else { return Err(bad()) };
    let Some(Value::Array(comps)) = map_get(om, 1) else { return Err(bad()) };
    let mut children = Vec::new();
    for pair in comps.chunks(2) {
        let idx = as_u64(&pair[0]).ok_or_else(bad)? as u32;
        let hardened = matches!(pair.get(1), Some(Value::Bool(true)));
        children.push(if hardened { ChildNumber::from_hardened_idx(idx) } else { ChildNumber::from_normal_idx(idx) }.map_err(|_| bad())?);
    }
    let parent = map_get(m, 8).and_then(as_u64).unwrap_or(0) as u32;
    let path = DerivationPath::from(children.clone());
    let xpub = Xpub {
        network: NetworkKind::from(network),
        depth: children.len() as u8,
        parent_fingerprint: Fingerprint::from(parent.to_be_bytes()),
        child_number: *children.last().ok_or_else(bad)?,
        public_key: key,
        chain_code,
    };
    Ok(ImportedAccount { script_type, fingerprint: master, path, xpub })
}

fn from_crypto_account(cbor: &[u8], network: Network) -> Result<Vec<ImportedAccount>, EngineError> {
    let v: Value = ciborium::from_reader(cbor).map_err(|_| bad())?;
    let Value::Map(m) = v else { return Err(bad()) };
    let master = Fingerprint::from((map_get(&m, 1).and_then(as_u64).ok_or_else(bad)? as u32).to_be_bytes());
    let Some(Value::Array(outputs)) = map_get(&m, 2) else { return Err(bad()) };
    let mut out = Vec::new();
    for o in outputs {
        let parsed = match o {
            Value::Tag(404, k) => hdkey_to_account(k, master, ClaimScriptType::P2wpkh, network),
            Value::Tag(403, k) => hdkey_to_account(k, master, ClaimScriptType::P2pkh, network),
            Value::Tag(409, k) => hdkey_to_account(k, master, ClaimScriptType::P2trKeyPath, network),
            Value::Tag(400, inner) => match inner.as_ref() {
                Value::Tag(404, k) => hdkey_to_account(k, master, ClaimScriptType::P2shP2wpkh, network),
                _ => continue, // sh(multi) etc.: multisig claims are out of scope for v1
            },
            _ => continue,
        };
        out.push(parsed?);
    }
    if out.is_empty() {
        return Err(bad());
    }
    Ok(out)
}

pub fn import_accounts(p: &ScannedPayload, network: Network) -> Result<Vec<ImportedAccount>, EngineError> {
    match p {
        ScannedPayload::Text(t) => from_text(t, network),
        ScannedPayload::Ur { ur_type, cbor } if ur_type == "crypto-account" => from_crypto_account(cbor, network),
        _ => Err(bad()),
    }
}
```
Add `pub mod ur; pub mod xpub_import;` to `lib.rs`.

- [ ] **Step 5: Run tests**

Run: `cargo test -p wren-engine ur:: xpub_import`
Expected: PASS (6 tests).

- [ ] **Step 6: Commit**

```bash
git add core/wren-engine/src/ur.rs core/wren-engine/src/xpub_import.rs core/wren-engine/src/lib.rs
git commit -m "feat(engine): BC-UR animated PSBT frames and claim xpub import (UR, Specter, descriptors)"
```

---

### Task 17: Claim session and scan

**Files:**
- Create: `core/wren-engine/src/claim/mod.rs`, `core/wren-engine/src/claim/scan.rs`
- Modify: `core/wren-engine/src/lib.rs` (add `pub mod claim;`)
- Test: inline tests in `claim/scan.rs`

**Interfaces:**
- Consumes: `ImportedAccount`, `ClaimScriptType` (Task 16), `sync::{scan_keychain, fetch_confirmed_update, STOP_GAP}` (Task 13), `ElectrumClient`, `HeaderChain`.
- Produces:
```rust
// claim/mod.rs
pub struct ClaimSession { /* in-memory only: accounts, ephemeral watch-only wallets, last plan */ }
#[derive(Clone, Debug, Default)]
pub struct FoundSummary { pub total: Amount, pub coins: usize, pub by_type: Vec<(ClaimScriptType, Amount, usize)>, pub unconfirmed_ignored: usize }
impl ClaimSession {
    pub fn start(accounts: Vec<ImportedAccount>, network: Network) -> Result<Self, EngineError>;
    pub async fn scan(&mut self, client: &ElectrumClient, chain: &HeaderChain, search_further: bool, now: u64) -> Result<FoundSummary, EngineError>;
}
pub fn claim_supported_types() -> Vec<ClaimScriptType>;
// claim/scan.rs
pub(crate) fn descriptors(a: &ImportedAccount) -> (String, String);
```
The ephemeral wallets are `bdk_wallet::Wallet`s created with `create_wallet_no_persist()` from public descriptors; nothing touches disk. Dropping the `ClaimSession` drops every xpub-derived structure (finish, cancel, or engine drop). "Search further" raises the gap limit from 20 to 200 on both chains.

- [ ] **Step 1: Write the failing tests** (in `claim/scan.rs`)

```rust
#[cfg(test)]
mod tests {
    use super::super::*;
    use crate::electrum::ElectrumClient;
    use crate::testutil::{serve, MockWorld, TestChain};
    use crate::xpub_import::{import_accounts, ClaimScriptType};
    use crate::ur::ScannedPayload;
    use bdk_wallet::bitcoin::{absolute::LockTime, hashes::Hash, transaction::Version, Amount, Network, OutPoint, Transaction, TxIn, TxOut, Txid};
    use bdk_wallet::KeychainKind;

    // BIP84 abandon…about account, as a regtest (tpub) key so it matches regtest addresses.
    fn old_wallet_text() -> String {
        use bdk_wallet::bitcoin::bip32::{DerivationPath, Xpriv, Xpub};
        use bdk_wallet::keys::bip39::{Language, Mnemonic};
        use std::str::FromStr;
        let secp = bdk_wallet::bitcoin::secp256k1::Secp256k1::new();
        let seed = Mnemonic::parse_in(Language::English, crate::testutil::ABANDON).unwrap().to_seed("");
        let m = Xpriv::new_master(Network::Regtest, &seed).unwrap();
        let x = Xpub::from_priv(&secp, &m.derive_priv(&secp, &DerivationPath::from_str("m/84'/0'/0'").unwrap()).unwrap());
        format!("[{}/84h/0h/0h]{}", m.fingerprint(&secp), x)
    }

    fn pay(spk: bdk_wallet::bitcoin::ScriptBuf, sats: u64, salt: u8) -> Transaction {
        Transaction { version: Version::TWO, lock_time: LockTime::ZERO,
            input: vec![TxIn { previous_output: OutPoint::new(Txid::from_byte_array([salt; 32]), 0), ..Default::default() }],
            output: vec![TxOut { value: Amount::from_sat(sats), script_pubkey: spk }] }
    }

    async fn world_with_coin_at_index(index: u32) -> (std::sync::Arc<std::sync::Mutex<MockWorld>>, HeaderChain, ClaimSession) {
        let accounts = import_accounts(&ScannedPayload::Text(old_wallet_text()), Network::Regtest).unwrap();
        let session = ClaimSession::start(accounts, Network::Regtest).unwrap();
        let spk = session.wallets[0].1.peek_address(KeychainKind::External, index).script_pubkey();
        let tx = pay(spk.clone(), 70_000, 3);
        let mut tc = TestChain::new(100, 30, 105, 1);
        tc.set_merkle_root(104, tx.compute_txid().to_byte_array()); // "pre-split" in this synthetic chain
        let world = MockWorld::new(tc.clone());
        {
            let mut w = world.lock().unwrap();
            w.txs.insert(tx.compute_txid(), tx.clone());
            w.add_history(&spk, tx.compute_txid(), 104);
            w.merkle.insert(tx.compute_txid(), (104, 0, vec![]));
        }
        let mut chain = HeaderChain::new(tc.params());
        crate::sync::sync_headers(&ElectrumClient::connect(serve(world.clone())).await.unwrap(), &mut chain).await.unwrap();
        (world, chain, session)
    }

    #[tokio::test]
    async fn finds_coins_on_the_imported_account() {
        let (world, chain, mut s) = world_with_coin_at_index(0).await;
        let c = ElectrumClient::connect(serve(world)).await.unwrap();
        let f = s.scan(&c, &chain, false, 1_700_000_000).await.unwrap();
        assert_eq!((f.total, f.coins), (Amount::from_sat(70_000), 1));
        assert_eq!(f.by_type[0].0, ClaimScriptType::P2wpkh);
    }

    #[tokio::test]
    async fn nothing_found_is_zero_not_error() {
        let (world, chain, mut s) = world_with_coin_at_index(0).await;
        world.lock().unwrap().history.clear();
        let c = ElectrumClient::connect(serve(world)).await.unwrap();
        let f = s.scan(&c, &chain, false, 1_700_000_000).await.unwrap();
        assert_eq!((f.total, f.coins), (Amount::ZERO, 0));
    }

    #[tokio::test]
    async fn search_further_finds_coins_beyond_the_gap() {
        let (world, chain, mut s) = world_with_coin_at_index(60).await;
        let c = ElectrumClient::connect(serve(world)).await.unwrap();
        assert_eq!(s.scan(&c, &chain, false, 1_700_000_000).await.unwrap().coins, 0);
        assert_eq!(s.scan(&c, &chain, true, 1_700_000_000).await.unwrap().coins, 1);
    }

    #[test]
    fn descriptors_for_every_type() {
        let mut a = import_accounts(&ScannedPayload::Text(old_wallet_text()), Network::Regtest).unwrap().remove(0);
        for (t, prefix) in [(ClaimScriptType::P2wpkh, "wpkh("), (ClaimScriptType::P2shP2wpkh, "sh(wpkh("), (ClaimScriptType::P2pkh, "pkh("), (ClaimScriptType::P2trKeyPath, "tr(")] {
            a.script_type = t;
            let (e, i) = scan::descriptors(&a);
            assert!(e.starts_with(prefix) && e.contains("/0/*") && i.contains("/1/*"), "{e}");
        }
    }
}
```

- [ ] **Step 2: Run to verify failure**

Run: `cargo test -p wren-engine claim::`
Expected: FAIL to compile — module missing.

- [ ] **Step 3: Implement `claim/scan.rs`**

```rust
//! Ephemeral watch-only wallets for the account being claimed.
use crate::xpub_import::{ClaimScriptType, ImportedAccount};

pub(crate) fn descriptors(a: &ImportedAccount) -> (String, String) {
    let key = |chain: u32| format!("[{}/{}]{}/{chain}/*", a.fingerprint, a.path, a.xpub);
    let wrap = |k: String| match a.script_type {
        ClaimScriptType::P2wpkh => format!("wpkh({k})"),
        ClaimScriptType::P2shP2wpkh => format!("sh(wpkh({k}))"),
        ClaimScriptType::P2pkh => format!("pkh({k})"),
        ClaimScriptType::P2trKeyPath => format!("tr({k})"),
    };
    (wrap(key(0)), wrap(key(1)))
}
```

- [ ] **Step 4: Implement `claim/mod.rs`**

```rust
//! Claim your XBT: sweep the XBT twins of a pre-split wallet, signed on a hardware signer.
//! Everything here lives in memory and is dropped with the session.
pub mod scan;

use crate::electrum::ElectrumClient;
use crate::error::EngineError;
use crate::headerchain::HeaderChain;
use crate::sync::{fetch_confirmed_update, scan_keychain, STOP_GAP};
use crate::xpub_import::{ClaimScriptType, ImportedAccount};
use bdk_wallet::bitcoin::{Amount, Network, Txid};
use bdk_wallet::chain::{BlockId, ChainPosition};
use bdk_wallet::{KeychainKind, Wallet};
use std::collections::HashMap;

pub const SEARCH_FURTHER_GAP: u32 = 200;

#[derive(Clone, Debug, Default)]
pub struct FoundSummary {
    pub total: Amount,
    pub coins: usize,
    pub by_type: Vec<(ClaimScriptType, Amount, usize)>,
    pub unconfirmed_ignored: usize,
}

pub struct ClaimSession {
    pub(crate) network: Network,
    pub(crate) wallets: Vec<(ClaimScriptType, Wallet)>,
    pub(crate) plan: Option<build::ClaimPlan>,
}

pub fn claim_supported_types() -> Vec<ClaimScriptType> {
    vec![ClaimScriptType::P2wpkh, ClaimScriptType::P2shP2wpkh, ClaimScriptType::P2pkh, ClaimScriptType::P2trKeyPath]
}

impl ClaimSession {
    pub fn start(accounts: Vec<ImportedAccount>, network: Network) -> Result<Self, EngineError> {
        if accounts.is_empty() {
            return Err(EngineError::Claim("no account key was provided"));
        }
        let mut wallets = Vec::new();
        for a in accounts {
            let (e, i) = scan::descriptors(&a);
            let w = Wallet::create(e, i).network(network).create_wallet_no_persist()
                .map_err(|_| EngineError::Claim("could not read this account key"))?;
            wallets.push((a.script_type, w));
        }
        Ok(ClaimSession { network, wallets, plan: None })
    }

    pub async fn scan(&mut self, client: &ElectrumClient, chain: &HeaderChain, search_further: bool, now: u64) -> Result<FoundSummary, EngineError> {
        let gap = if search_further { SEARCH_FURTHER_GAP } else { STOP_GAP };
        let mut summary = FoundSummary::default();
        for (ty, wallet) in self.wallets.iter_mut() {
            let mut found: HashMap<Txid, i64> = HashMap::new();
            let ext = scan_keychain(client, wallet, KeychainKind::External, gap, &mut found).await?;
            let int = scan_keychain(client, wallet, KeychainKind::Internal, gap, &mut found).await?;
            let genesis = BlockId { height: 0, hash: wallet.local_chain().genesis_hash() };
            let existing: Vec<BlockId> = wallet.local_chain().iter_checkpoints().map(|c| c.block_id()).collect();
            let graph = wallet.tx_graph();
            let have = |t: &Txid| graph.get_tx(*t).is_some();
            let mut update = fetch_confirmed_update(client, chain, genesis, &existing, &found, &have, now).await?;
            if let Some(i) = ext { update.last_active_indices.insert(KeychainKind::External, i); }
            if let Some(i) = int { update.last_active_indices.insert(KeychainKind::Internal, i); }
            wallet.apply_update(update).map_err(|_| EngineError::Chain("update does not connect"))?;
            let (mut amt, mut n) = (Amount::ZERO, 0usize);
            for u in wallet.list_unspent() {
                if matches!(u.chain_position, ChainPosition::Confirmed { .. }) {
                    amt += u.txout.value;
                    n += 1;
                } else {
                    summary.unconfirmed_ignored += 1;
                }
            }
            if n > 0 {
                summary.by_type.push((*ty, amt, n));
                summary.total += amt;
                summary.coins += n;
            }
        }
        self.plan = None;
        Ok(summary)
    }
}

pub mod build;
pub mod psbt_sigs;
pub mod verify;
```
Create empty placeholder modules so this compiles until Tasks 18–19 fill them: `claim/build.rs` containing only `pub struct ClaimPlan;`, and empty `claim/psbt_sigs.rs` and `claim/verify.rs`. (Tasks 18 and 19 replace these files entirely.)

Add `pub mod claim;` to `lib.rs`.

- [ ] **Step 5: Run tests**

Run: `cargo test -p wren-engine claim::`
Expected: PASS (4 tests).

- [ ] **Step 6: Commit**

```bash
git add core/wren-engine/src/claim core/wren-engine/src/lib.rs
git commit -m "feat(engine): claim session with in-memory watch-only scan and search-further"
```

---

### Task 18: Claim build — sweep PSBTs marked 0x21 on every input

**Files:**
- Replace: `core/wren-engine/src/claim/build.rs`
- Modify: `core/wren-engine/src/testutil.rs` (add `claim_fixture`, `OLD_KEYS` helpers)
- Test: inline tests in `claim/build.rs`

**Interfaces:**
- Consumes: `ClaimSession` (Task 17), `ur::encode_psbt_frames` (Task 16), `sighash::SIGHASH_ALL_UNIFIED`.
- Produces:
```rust
pub const MAX_INPUTS_PER_TX: usize = 200;
#[derive(Clone, Debug)]
pub struct ClaimTx { pub script_type: ClaimScriptType, pub psbt: Psbt, pub inputs: usize, pub amount: Amount, pub fee: Amount, pub qr_frames: Vec<String> }
#[derive(Clone, Debug)]
pub struct ClaimPlan { pub destination: String, pub txs: Vec<ClaimTx>, pub total_in: Amount, pub total_fee: Amount, pub total_out: Amount }
impl ClaimSession {
    pub fn build(&mut self, destination: &Address, fee_rate: FeeRate) -> Result<&ClaimPlan, EngineError>;
    pub(crate) fn build_with_limit(&mut self, destination: &Address, fee_rate: FeeRate, max_inputs: usize) -> Result<&ClaimPlan, EngineError>;
}
pub(crate) fn check_prevouts(psbt: &Psbt, ty: ClaimScriptType) -> Result<(), EngineError>;
// testutil.rs (tests only)
pub fn old_account_text(ty: ClaimScriptType) -> String;  // "[fp/purpose'/0'/0']tpub" descriptor text for ABANDON on regtest
pub async fn claim_fixture(types: &[ClaimScriptType], coins_per_type: u32) -> (Arc<Mutex<MockWorld>>, HeaderChain, ClaimSession);
```
One sweep transaction per script type (a BDK wallet can only spend its own descriptor's coins), split further if a type has more than 200 coins. All found **confirmed** coins are swept; there is no change output. For taproot, BDK sizes Schnorr signatures at 64 bytes but a UNIFIED signature always carries the hash-type byte (65 bytes), so the fee is raised by one weight unit per input so the requested rate is still met (research: consensus facts, "budget 65").

- [ ] **Step 1: Add claim test helpers to `testutil.rs`**

```rust
use crate::claim::ClaimSession;
use crate::ur::ScannedPayload;
use crate::xpub_import::{import_accounts, ClaimScriptType};

pub fn purpose(ty: ClaimScriptType) -> u32 {
    match ty { ClaimScriptType::P2wpkh => 84, ClaimScriptType::P2shP2wpkh => 49, ClaimScriptType::P2pkh => 44, ClaimScriptType::P2trKeyPath => 86 }
}

pub fn abandon_master() -> bdk_wallet::bitcoin::bip32::Xpriv {
    use bdk_wallet::keys::bip39::{Language, Mnemonic};
    let seed = Mnemonic::parse_in(Language::English, ABANDON).unwrap().to_seed("");
    bdk_wallet::bitcoin::bip32::Xpriv::new_master(Network::Regtest, &seed).unwrap()
}

pub fn old_account_text(ty: ClaimScriptType) -> String {
    use bdk_wallet::bitcoin::bip32::{DerivationPath, Xpub};
    use std::str::FromStr;
    let secp = bdk_wallet::bitcoin::secp256k1::Secp256k1::new();
    let m = abandon_master();
    let p = purpose(ty);
    let x = Xpub::from_priv(&secp, &m.derive_priv(&secp, &DerivationPath::from_str(&format!("m/{p}'/0'/0'")).unwrap()).unwrap());
    let key = format!("[{}/{p}h/0h/0h]{x}/0/*", m.fingerprint(&secp));
    match ty {
        ClaimScriptType::P2wpkh => format!("wpkh({key})"),
        ClaimScriptType::P2shP2wpkh => format!("sh(wpkh({key}))"),
        ClaimScriptType::P2pkh => format!("pkh({key})"),
        ClaimScriptType::P2trKeyPath => format!("tr({key})"),
    }
}

/// Pays 70_000 sats to external addresses 0..coins_per_type of each type, one tx per block from height 101.
pub async fn claim_fixture(types: &[ClaimScriptType], coins_per_type: u32) -> (Arc<Mutex<MockWorld>>, HeaderChain, ClaimSession) {
    let mut accounts = Vec::new();
    for t in types {
        accounts.extend(import_accounts(&ScannedPayload::Text(old_account_text(*t)), Network::Regtest).unwrap());
    }
    let session = ClaimSession::start(accounts, Network::Regtest).unwrap();
    let mut tc = TestChain::new(100, 60, 105, 1);
    let mut entries = Vec::new();
    let mut height = 101;
    for (w_idx, _) in types.iter().enumerate() {
        for i in 0..coins_per_type {
            let spk = session.wallets[w_idx].1.peek_address(KeychainKind::External, i).script_pubkey();
            let tx = Transaction { version: Version::TWO, lock_time: LockTime::ZERO,
                input: vec![TxIn { previous_output: OutPoint::new(bdk_wallet::bitcoin::Txid::from_byte_array([height as u8; 32]), 1), ..Default::default() }],
                output: vec![TxOut { value: Amount::from_sat(70_000), script_pubkey: spk.clone() }] };
            tc.set_merkle_root(height, tx.compute_txid().to_byte_array());
            entries.push((tx, spk, height));
            height += 1;
        }
    }
    let world = MockWorld::new(tc.clone());
    {
        let mut w = world.lock().unwrap();
        for (tx, spk, h) in &entries {
            w.txs.insert(tx.compute_txid(), tx.clone());
            w.add_history(spk, tx.compute_txid(), *h as i64);
            w.merkle.insert(tx.compute_txid(), (*h, 0, vec![]));
        }
    }
    let mut chain = HeaderChain::new(tc.params());
    let client = crate::electrum::ElectrumClient::connect(serve(world.clone())).await.unwrap();
    crate::sync::sync_headers(&client, &mut chain).await.unwrap();
    let mut session = session;
    session.scan(&client, &chain, false, 1_700_000_000).await.unwrap();
    (world, chain, session)
}
```

- [ ] **Step 2: Write the failing tests** (in `claim/build.rs`)

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::testutil::claim_fixture;
    use bdk_wallet::bitcoin::{Network, Weight};
    use std::str::FromStr;

    fn dest() -> Address {
        Address::from_str("bcrt1qw508d6qejxtdg4y5r3zarvary0c5xw7kygt080").unwrap().require_network(Network::Regtest).unwrap()
    }
    const ALL: [ClaimScriptType; 4] = [ClaimScriptType::P2wpkh, ClaimScriptType::P2shP2wpkh, ClaimScriptType::P2pkh, ClaimScriptType::P2trKeyPath];

    #[tokio::test]
    async fn sweeps_every_type_with_0x21_and_no_change() {
        let (_w, _c, mut s) = claim_fixture(&ALL, 1).await;
        let plan = s.build(&dest(), FeeRate::from_sat_per_vb(2).unwrap()).unwrap().clone();
        assert_eq!(plan.txs.len(), 4);
        for t in &plan.txs {
            assert_eq!(t.psbt.unsigned_tx.output.len(), 1);
            assert_eq!(t.psbt.unsigned_tx.output[0].script_pubkey, dest().script_pubkey());
            assert!(t.psbt.inputs.iter().all(|i| i.sighash_type.map(|x| x.to_u32()) == Some(0x21)));
            assert_eq!(t.amount + t.fee, Amount::from_sat(70_000));
            assert!(!t.qr_frames.is_empty());
        }
        assert_eq!(plan.total_in, Amount::from_sat(280_000));
    }

    #[tokio::test]
    async fn claim_build_requires_every_prevout() {
        let (_w, _c, mut s) = claim_fixture(&[ClaimScriptType::P2pkh], 1).await;
        let mut psbt = s.build(&dest(), FeeRate::from_sat_per_vb(1).unwrap()).unwrap().txs[0].psbt.clone();
        assert!(check_prevouts(&psbt, ClaimScriptType::P2pkh).is_ok());
        psbt.inputs[0].non_witness_utxo = None;
        psbt.inputs[0].witness_utxo = None;
        assert!(check_prevouts(&psbt, ClaimScriptType::P2pkh).is_err());
    }

    #[tokio::test]
    async fn taproot_fee_budgets_65_byte_signatures() {
        let (_w, _c, mut s) = claim_fixture(&[ClaimScriptType::P2trKeyPath], 3).await;
        let t = s.build(&dest(), FeeRate::from_sat_per_vb(1).unwrap()).unwrap().txs[0].clone();
        // weight with signatures = unsigned weight + 2 (segwit marker/flag) + per input (1 count + 1 len + 65 sig)
        let w = t.psbt.unsigned_tx.weight() + Weight::from_wu(2 + 67 * t.inputs as u64);
        assert!(t.fee.to_sat() >= w.to_vbytes_ceil(), "fee {} < vsize {}", t.fee, w.to_vbytes_ceil());
    }

    #[tokio::test]
    async fn splits_large_claims() {
        let (_w, _c, mut s) = claim_fixture(&[ClaimScriptType::P2wpkh], 3).await;
        let plan = s.build_with_limit(&dest(), FeeRate::from_sat_per_vb(1).unwrap(), 1).unwrap();
        assert_eq!(plan.txs.len(), 3);
    }

    #[tokio::test]
    async fn refuses_before_anything_was_found_and_below_fee_floor() {
        let (_w, _c, mut s) = claim_fixture(&[ClaimScriptType::P2wpkh], 1).await;
        assert!(s.build(&dest(), FeeRate::from_sat_per_kwu(100)).is_err());
        assert!(ClaimSession::start(vec![], Network::Regtest).is_err());
    }
}
```

- [ ] **Step 3: Run to verify failure**

Run: `cargo test -p wren-engine claim::build`
Expected: FAIL to compile — `build` / `ClaimTx` missing.

- [ ] **Step 4: Implement `claim/build.rs`**

```rust
//! Claim sweep transactions. Every input is marked 0x21 so the SeedSigner fork firmware signs UNIFIED.
use super::ClaimSession;
use crate::error::EngineError;
use crate::sighash::SIGHASH_ALL_UNIFIED;
use crate::ur::encode_psbt_frames;
use crate::xpub_import::ClaimScriptType;
use bdk_wallet::bitcoin::psbt::PsbtSighashType;
use bdk_wallet::bitcoin::{Address, Amount, FeeRate, OutPoint, Psbt, Sequence};
use bdk_wallet::chain::ChainPosition;
use bdk_wallet::Wallet;

pub const MAX_INPUTS_PER_TX: usize = 200;

#[derive(Clone, Debug)]
pub struct ClaimTx {
    pub script_type: ClaimScriptType,
    pub psbt: Psbt,
    pub inputs: usize,
    pub amount: Amount,
    pub fee: Amount,
    pub qr_frames: Vec<String>,
}

#[derive(Clone, Debug)]
pub struct ClaimPlan {
    pub destination: String,
    pub txs: Vec<ClaimTx>,
    pub total_in: Amount,
    pub total_fee: Amount,
    pub total_out: Amount,
}

fn werr<E: std::fmt::Display>(e: E) -> EngineError {
    EngineError::Wallet(e.to_string())
}

pub(crate) fn check_prevouts(psbt: &Psbt, ty: ClaimScriptType) -> Result<(), EngineError> {
    for inp in &psbt.inputs {
        let ok = match ty {
            ClaimScriptType::P2pkh => inp.non_witness_utxo.is_some(),
            _ => inp.witness_utxo.is_some() || inp.non_witness_utxo.is_some(),
        };
        if !ok {
            return Err(EngineError::Claim("a coin's previous transaction is missing; scan again"));
        }
        if inp.sighash_type.map(|s| s.to_u32()) != Some(SIGHASH_ALL_UNIFIED) {
            return Err(EngineError::Claim("an input is not marked replay-safe"));
        }
    }
    Ok(())
}

fn sweep(wallet: &mut Wallet, outpoints: &[OutPoint], dest: &Address, rate: FeeRate, absolute: Option<Amount>) -> Result<Psbt, EngineError> {
    let mut b = wallet.build_tx();
    b.add_utxos(outpoints).map_err(werr)?;
    b.manually_selected_only()
        .drain_to(dest.script_pubkey())
        .sighash(PsbtSighashType::from_u32(SIGHASH_ALL_UNIFIED))
        .set_exact_sequence(Sequence::ENABLE_RBF_NO_LOCKTIME);
    match absolute {
        Some(f) => { b.fee_absolute(f); }
        None => { b.fee_rate(rate); }
    }
    b.finish().map_err(werr)
}

fn build_one(wallet: &mut Wallet, ty: ClaimScriptType, outpoints: &[OutPoint], dest: &Address, rate: FeeRate) -> Result<Psbt, EngineError> {
    let first = sweep(wallet, outpoints, dest, rate, None)?;
    if ty != ClaimScriptType::P2trKeyPath {
        return Ok(first);
    }
    let extra = (outpoints.len() as u64 * rate.to_sat_per_kwu()).div_ceil(1000);
    let fee = first.fee().map_err(werr)? + Amount::from_sat(extra);
    sweep(wallet, outpoints, dest, rate, Some(fee))
}

impl ClaimSession {
    pub fn build(&mut self, destination: &Address, fee_rate: FeeRate) -> Result<&ClaimPlan, EngineError> {
        self.build_with_limit(destination, fee_rate, MAX_INPUTS_PER_TX)
    }

    pub(crate) fn build_with_limit(&mut self, destination: &Address, fee_rate: FeeRate, max_inputs: usize) -> Result<&ClaimPlan, EngineError> {
        if fee_rate < FeeRate::from_sat_per_vb_unchecked(1) {
            return Err(EngineError::Input("fee rate is below 1 sat/vB"));
        }
        let mut txs = Vec::new();
        for (ty, wallet) in self.wallets.iter_mut() {
            let utxos: Vec<OutPoint> = wallet.list_unspent()
                .filter(|u| matches!(u.chain_position, ChainPosition::Confirmed { .. }))
                .map(|u| u.outpoint)
                .collect();
            for chunk in utxos.chunks(max_inputs) {
                let psbt = build_one(wallet, *ty, chunk, destination, fee_rate)?;
                check_prevouts(&psbt, *ty)?;
                let fee = psbt.fee().map_err(werr)?;
                let amount = psbt.unsigned_tx.output.iter().map(|o| o.value).sum();
                let qr_frames = encode_psbt_frames(&psbt.serialize())?;
                txs.push(ClaimTx { script_type: *ty, inputs: psbt.inputs.len(), psbt, amount, fee, qr_frames });
            }
        }
        if txs.is_empty() {
            return Err(EngineError::Claim("nothing to claim; scan first"));
        }
        let total_fee = txs.iter().map(|t| t.fee).sum();
        let total_out = txs.iter().map(|t| t.amount).sum();
        self.plan = Some(ClaimPlan { destination: destination.to_string(), txs, total_in: total_fee + total_out, total_fee, total_out });
        Ok(self.plan.as_ref().unwrap())
    }
}
```

- [ ] **Step 5: Run tests**

Run: `cargo test -p wren-engine claim::build`
Expected: PASS (5 tests).

- [ ] **Step 6: Commit**

```bash
git add core/wren-engine/src/claim/build.rs core/wren-engine/src/testutil.rs
git commit -m "feat(engine): claim sweep PSBTs (0x21 inputs, no change, 65-byte taproot fee budget)"
```

---

### Task 19: Read the signer's PSBT, run the five checks, finalize, broadcast

**Files:**
- Replace: `core/wren-engine/src/claim/psbt_sigs.rs`, `core/wren-engine/src/claim/verify.rs`
- Modify: `core/wren-engine/src/claim/mod.rs` (add `ClaimSession::finish`)
- Test: inline tests in `claim/psbt_sigs.rs` and `claim/verify.rs`

**Interfaces:**
- Consumes: `ClaimPlan`, `ClaimTx` (Task 18), `sign::{verify_ecdsa_unified, verify_schnorr_unified, p2wpkh_script_code}` (Task 8), `ur::{psbt_from_payload, ScannedPayload}` (Task 16).
- Produces:
```rust
// psbt_sigs.rs
#[derive(Clone, Debug, Default, PartialEq, Eq)]
pub struct SignedInput { pub partial_sigs: Vec<(Vec<u8>, Vec<u8>)>, pub tap_key_sig: Option<Vec<u8>> }
#[derive(Clone, Debug, PartialEq, Eq)]
pub struct SignedPsbt { pub unsigned_tx: Vec<u8>, pub inputs: Vec<SignedInput> }
pub fn read_signed_psbt(bytes: &[u8]) -> Result<SignedPsbt, EngineError>;
#[cfg(test)] pub(crate) fn write_minimal_signed_psbt(unsigned_tx: &[u8], inputs: &[SignedInput], n_outputs: usize) -> Vec<u8>;
// verify.rs
pub const WRONG_SIGNATURE_MESSAGE: &str;
pub fn verify_and_finalize(t: &ClaimTx, signed: &SignedPsbt) -> Result<Transaction, EngineError>;
// claim/mod.rs
impl ClaimSession {
    pub async fn finish(&mut self, signed: &[ScannedPayload], client: &ElectrumClient) -> Result<Vec<Txid>, EngineError>;
}
```

**Why a custom reader:** `rust-bitcoin` decodes `PSBT_IN_PARTIAL_SIG` values into `ecdsa::Signature`, whose hash-type field cannot hold `0x21`, so a signed claim PSBT fails to deserialize at all (research: Rust stack §2). The reader extracts only the per-input signatures (`0x02` partial signatures, `0x13` taproot key-path signature) and the signer's copy of the unsigned transaction; everything else comes from Wren's own in-memory PSBT. **The five checks** (spec §4.1): (1) every signature's hash-type byte is `0x21`; (2) every signature verifies against the UNIFIED digest; (3) the inputs are exactly the scanned coins; (4) destination and amount are exactly as reviewed; (5) the fee is exactly as chosen. Checks 3–5 are enforced together by requiring the signer's unsigned transaction to be byte-identical to Wren's (a change to any input, output, or amount changes those bytes, and the fee is inputs minus outputs), with a specific message for which part differs. **All transactions of a plan are verified before any is broadcast.**

- [ ] **Step 1: Write the failing reader tests** (in `psbt_sigs.rs`)

```rust
#[cfg(test)]
mod tests {
    use super::*;

    fn unsigned() -> Vec<u8> {
        // version 2, 1 input (null outpoint), 1 output of 1000 sats to empty script, locktime 0
        hex::decode("02000000010000000000000000000000000000000000000000000000000000000000000000ffffffff00fdffffff01e8030000000000000000000000").unwrap()
    }

    #[test]
    fn reads_partial_and_taproot_signatures() {
        let inputs = vec![SignedInput { partial_sigs: vec![(vec![2; 33], vec![0x30, 0x21])], tap_key_sig: Some(vec![9; 65]) }];
        let b = write_minimal_signed_psbt(&unsigned(), &inputs, 1);
        let s = read_signed_psbt(&b).unwrap();
        assert_eq!(s.unsigned_tx, unsigned());
        assert_eq!(s.inputs, inputs);
    }

    #[test]
    fn rejects_truncated_bad_magic_and_duplicate_keys() {
        let b = write_minimal_signed_psbt(&unsigned(), &[SignedInput::default()], 1);
        assert!(read_signed_psbt(&b[..b.len() - 1]).is_err());
        let mut bad = b.clone();
        bad[0] = b'x';
        assert!(read_signed_psbt(&bad).is_err());
        let dup = SignedInput { partial_sigs: vec![(vec![2; 33], vec![1]), (vec![2; 33], vec![2])], tap_key_sig: None };
        assert!(read_signed_psbt(&write_minimal_signed_psbt(&unsigned(), &[dup], 1)).is_err());
    }
}
```

- [ ] **Step 2: Implement `psbt_sigs.rs`**

```rust
//! Minimal BIP174 reader for signed claim PSBTs. Reads only what verification needs.
use crate::error::EngineError;
use bdk_wallet::bitcoin::{consensus::deserialize, Transaction};

#[derive(Clone, Debug, Default, PartialEq, Eq)]
pub struct SignedInput {
    pub partial_sigs: Vec<(Vec<u8>, Vec<u8>)>,
    pub tap_key_sig: Option<Vec<u8>>,
}

#[derive(Clone, Debug, PartialEq, Eq)]
pub struct SignedPsbt {
    pub unsigned_tx: Vec<u8>,
    pub inputs: Vec<SignedInput>,
}

fn bad() -> EngineError {
    EngineError::Claim("the signed QR code is not a valid PSBT")
}

struct Reader<'a> {
    b: &'a [u8],
    i: usize,
}

impl<'a> Reader<'a> {
    fn take(&mut self, n: usize) -> Result<&'a [u8], EngineError> {
        if self.b.len() - self.i < n {
            return Err(bad());
        }
        let s = &self.b[self.i..self.i + n];
        self.i += n;
        Ok(s)
    }
    fn compact(&mut self) -> Result<usize, EngineError> {
        let f = self.take(1)?[0];
        let v = match f {
            0xfd => u16::from_le_bytes(self.take(2)?.try_into().unwrap()) as u64,
            0xfe => u32::from_le_bytes(self.take(4)?.try_into().unwrap()) as u64,
            0xff => u64::from_le_bytes(self.take(8)?.try_into().unwrap()),
            x => x as u64,
        };
        usize::try_from(v).map_err(|_| bad())
    }
    fn map(&mut self) -> Result<Vec<(&'a [u8], &'a [u8])>, EngineError> {
        let mut out: Vec<(&[u8], &[u8])> = Vec::new();
        loop {
            let klen = self.compact()?;
            if klen == 0 {
                return Ok(out);
            }
            let k = self.take(klen)?;
            let vlen = self.compact()?;
            let v = self.take(vlen)?;
            if out.iter().any(|(ok, _)| *ok == k) {
                return Err(bad());
            }
            out.push((k, v));
        }
    }
}

pub fn read_signed_psbt(bytes: &[u8]) -> Result<SignedPsbt, EngineError> {
    let mut r = Reader { b: bytes, i: 0 };
    if r.take(5)? != b"psbt\xff" {
        return Err(bad());
    }
    let global = r.map()?;
    let unsigned_tx = global.iter().find(|(k, _)| *k == [0x00]).map(|(_, v)| v.to_vec()).ok_or_else(bad)?;
    let tx: Transaction = deserialize(&unsigned_tx).map_err(|_| bad())?;
    let mut inputs = Vec::with_capacity(tx.input.len());
    for _ in 0..tx.input.len() {
        let mut si = SignedInput::default();
        for (k, v) in r.map()? {
            match k[0] {
                0x02 => si.partial_sigs.push((k[1..].to_vec(), v.to_vec())),
                0x13 if k.len() == 1 => si.tap_key_sig = Some(v.to_vec()),
                _ => {}
            }
        }
        inputs.push(si);
    }
    for _ in 0..tx.output.len() {
        r.map()?;
    }
    if r.i != bytes.len() {
        return Err(bad());
    }
    Ok(SignedPsbt { unsigned_tx, inputs })
}

#[cfg(test)]
pub(crate) fn write_minimal_signed_psbt(unsigned_tx: &[u8], inputs: &[SignedInput], n_outputs: usize) -> Vec<u8> {
    use bdk_wallet::bitcoin::consensus::encode::{serialize, VarInt};
    let kv = |out: &mut Vec<u8>, k: &[u8], v: &[u8]| {
        out.extend(serialize(&VarInt(k.len() as u64)));
        out.extend(k);
        out.extend(serialize(&VarInt(v.len() as u64)));
        out.extend(v);
    };
    let mut out = b"psbt\xff".to_vec();
    kv(&mut out, &[0x00], unsigned_tx);
    out.push(0);
    for si in inputs {
        for (pk, sig) in &si.partial_sigs {
            let mut k = vec![0x02];
            k.extend(pk);
            kv(&mut out, &k, sig);
        }
        if let Some(t) = &si.tap_key_sig {
            kv(&mut out, &[0x13], t);
        }
        out.push(0);
    }
    for _ in 0..n_outputs {
        out.push(0);
    }
    out
}
```
Run: `cargo test -p wren-engine claim::psbt_sigs` — Expected: PASS (2 tests).

- [ ] **Step 3: Write the failing verification tests** (in `verify.rs`)

A test signer that behaves like the SeedSigner fork firmware (signs each input with its key, hash type as instructed) and writes a minimal signed PSBT:
```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::claim::build::ClaimPlan;
    use crate::claim::psbt_sigs::{write_minimal_signed_psbt, SignedInput};
    use crate::electrum::ElectrumClient;
    use crate::sighash::{unified_sighash, ScriptType};
    use crate::testutil::{abandon_master, claim_fixture, serve};
    use crate::ur::ScannedPayload;
    use base64::Engine;
    use bdk_wallet::bitcoin::key::TapTweak;
    use bdk_wallet::bitcoin::secp256k1::{Keypair, Message, Secp256k1};
    use bdk_wallet::bitcoin::{consensus::encode::serialize, Address, FeeRate, Network, Script};
    use std::str::FromStr;

    const ALL: [ClaimScriptType; 4] = [ClaimScriptType::P2wpkh, ClaimScriptType::P2shP2wpkh, ClaimScriptType::P2pkh, ClaimScriptType::P2trKeyPath];

    fn dest() -> Address {
        Address::from_str("bcrt1qw508d6qejxtdg4y5r3zarvary0c5xw7kygt080").unwrap().require_network(Network::Regtest).unwrap()
    }

    /// Signs like the fork firmware. `type_byte` is the hash-type byte appended (0x21 normal; 0x01 = legacy firmware).
    fn sign(t: &ClaimTx, type_byte: u8, unsigned_override: Option<Vec<u8>>) -> ScannedPayload {
        let secp = Secp256k1::new();
        let master = abandon_master();
        let tx = &t.psbt.unsigned_tx;
        let spent: Vec<_> = t.psbt.inputs.iter().zip(&tx.input).map(|(i, txin)| i.witness_utxo.clone()
            .unwrap_or_else(|| i.non_witness_utxo.as_ref().unwrap().output[txin.previous_output.vout as usize].clone())).collect();
        let mut sigs = Vec::new();
        for (idx, inp) in t.psbt.inputs.iter().enumerate() {
            if t.script_type == ClaimScriptType::P2trKeyPath {
                let (_, (_, (_, path))) = inp.tap_key_origins.iter().next().unwrap();
                let sk = master.derive_priv(&secp, path).unwrap().private_key;
                let tweaked = Keypair::from_secret_key(&secp, &sk).tap_tweak(&secp, None);
                let d = unified_sighash(tx, idx, &spent, 0x21, ScriptType::Taproot, Script::new(), None).unwrap();
                let mut s = secp.sign_schnorr_no_aux_rand(&Message::from_digest(d), &tweaked.to_keypair()).serialize().to_vec();
                s.push(type_byte);
                sigs.push(SignedInput { partial_sigs: vec![], tap_key_sig: Some(s) });
            } else {
                let (pk, (_, path)) = inp.bip32_derivation.iter().next().unwrap();
                let sk = master.derive_priv(&secp, path).unwrap().private_key;
                let (st, code) = match t.script_type {
                    ClaimScriptType::P2pkh => (ScriptType::Bare, spent[idx].script_pubkey.clone()),
                    _ => (ScriptType::WitnessV0, crate::sign::p2wpkh_script_code(pk)),
                };
                let d = unified_sighash(tx, idx, &spent, 0x21, st, &code, None).unwrap();
                let mut s = secp.sign_ecdsa_low_r(&Message::from_digest(d), &sk).serialize_der().to_vec();
                s.push(type_byte);
                sigs.push(SignedInput { partial_sigs: vec![(pk.serialize().to_vec(), s)], tap_key_sig: None });
            }
        }
        let u = unsigned_override.unwrap_or_else(|| serialize(tx));
        let b = write_minimal_signed_psbt(&u, &sigs, tx.output.len());
        ScannedPayload::Text(base64::engine::general_purpose::STANDARD.encode(b))
    }

    async fn plan(types: &[ClaimScriptType]) -> (std::sync::Arc<std::sync::Mutex<crate::testutil::MockWorld>>, crate::claim::ClaimSession, ClaimPlan) {
        let (w, _c, mut s) = claim_fixture(types, 1).await;
        let p = s.build(&dest(), FeeRate::from_sat_per_vb(2).unwrap()).unwrap().clone();
        (w, s, p)
    }

    #[tokio::test]
    async fn all_four_types_verify_finalize_and_broadcast() {
        let (world, mut s, p) = plan(&ALL).await;
        let signed: Vec<_> = p.txs.iter().map(|t| sign(t, 0x21, None)).collect();
        let c = ElectrumClient::connect(serve(world.clone())).await.unwrap();
        let txids = s.finish(&signed, &c).await.unwrap();
        assert_eq!(txids.len(), 4);
        let sent = world.lock().unwrap().broadcasts.clone();
        assert_eq!(sent.len(), 4);
        for tx in &sent {
            assert!(tx.input.iter().all(|i| !i.witness.is_empty() || !i.script_sig.is_empty()));
        }
    }

    #[tokio::test]
    async fn normal_signature_is_blocked_with_the_firmware_message() {
        let (world, mut s, p) = plan(&[ClaimScriptType::P2wpkh]).await;
        let c = ElectrumClient::connect(serve(world.clone())).await.unwrap();
        match s.finish(&[sign(&p.txs[0], 0x01, None)], &c).await {
            Err(EngineError::Claim(m)) => assert_eq!(m, WRONG_SIGNATURE_MESSAGE),
            other => panic!("expected refusal, got {other:?}"),
        }
        assert!(world.lock().unwrap().broadcasts.is_empty());
    }

    #[tokio::test]
    async fn changed_transaction_is_refused_for_each_kind_of_change() {
        let (world, mut s, p) = plan(&[ClaimScriptType::P2wpkh]).await;
        let c = ElectrumClient::connect(serve(world.clone())).await.unwrap();
        let base = p.txs[0].psbt.unsigned_tx.clone();
        let mut out_changed = base.clone();
        out_changed.output[0].script_pubkey = bdk_wallet::bitcoin::ScriptBuf::new_op_return([1u8; 4]);
        let mut fee_changed = base.clone();
        fee_changed.output[0].value = fee_changed.output[0].value - bdk_wallet::bitcoin::Amount::from_sat(500);
        let mut input_added = base.clone();
        input_added.input.push(base.input[0].clone());
        let mut input_dropped = base.clone();
        input_dropped.output.clear();
        for bad in [out_changed, fee_changed, input_added, input_dropped] {
            assert!(s.finish(&[sign(&p.txs[0], 0x21, Some(serialize(&bad)))], &c).await.is_err());
        }
        assert!(world.lock().unwrap().broadcasts.is_empty());
    }

    #[tokio::test]
    async fn missing_signature_is_refused() {
        let (world, mut s, p) = plan(&[ClaimScriptType::P2pkh]).await;
        let c = ElectrumClient::connect(serve(world.clone())).await.unwrap();
        let empty = write_minimal_signed_psbt(&serialize(&p.txs[0].psbt.unsigned_tx), &[SignedInput::default()], 1);
        let payload = ScannedPayload::Text(base64::engine::general_purpose::STANDARD.encode(empty));
        assert!(s.finish(&[payload], &c).await.is_err());
    }

    #[tokio::test]
    async fn nothing_is_broadcast_unless_every_transaction_verifies() {
        let (world, mut s, p) = plan(&[ClaimScriptType::P2wpkh, ClaimScriptType::P2pkh]).await;
        let c = ElectrumClient::connect(serve(world.clone())).await.unwrap();
        let signed = vec![sign(&p.txs[0], 0x21, None), sign(&p.txs[1], 0x01, None)];
        assert!(s.finish(&signed, &c).await.is_err());
        assert!(world.lock().unwrap().broadcasts.is_empty());
    }
}
```

- [ ] **Step 4: Run to verify failure**

Run: `cargo test -p wren-engine claim::verify`
Expected: FAIL to compile — `verify_and_finalize` / `finish` missing.

- [ ] **Step 5: Implement `verify.rs`**

```rust
//! The pre-broadcast checks for claim transactions, and finalization.
use super::build::ClaimTx;
use super::psbt_sigs::SignedPsbt;
use crate::error::EngineError;
use crate::sighash::{ScriptType, SIGHASH_ALL_UNIFIED};
use crate::sign::{p2wpkh_script_code, verify_ecdsa_unified, verify_schnorr_unified};
use crate::xpub_import::ClaimScriptType;
use bdk_wallet::bitcoin::script::{Builder, PushBytesBuf};
use bdk_wallet::bitcoin::secp256k1::{Secp256k1, XOnlyPublicKey};
use bdk_wallet::bitcoin::{consensus::{deserialize, encode::serialize}, CompressedPublicKey, ScriptBuf, Transaction, TxOut, Witness};

pub const WRONG_SIGNATURE_MESSAGE: &str = "Your signer made a normal signature, which could also move your BTC. Wren blocked it. Your signer needs the fork firmware — see the guide.";

fn diagnose(ours: &Transaction, theirs: &[u8]) -> EngineError {
    let Ok(t) = deserialize::<Transaction>(theirs) else { return EngineError::Claim("the signed transaction was changed") };
    let ins = |x: &Transaction| x.input.iter().map(|i| i.previous_output).collect::<Vec<_>>();
    if ins(ours) != ins(&t) {
        EngineError::Claim("the signed transaction spends different coins than Wren found")
    } else if ours.output != t.output {
        EngineError::Claim("the signed transaction pays a different destination, amount, or fee")
    } else {
        EngineError::Claim("the signed transaction was changed")
    }
}

fn push(b: &[u8]) -> PushBytesBuf {
    PushBytesBuf::try_from(b.to_vec()).expect("signature and key pushes are small")
}

pub fn verify_and_finalize(t: &ClaimTx, signed: &SignedPsbt) -> Result<Transaction, EngineError> {
    let ours = &t.psbt.unsigned_tx;
    if signed.unsigned_tx != serialize(ours) {
        return Err(diagnose(ours, &signed.unsigned_tx));
    }
    if signed.inputs.len() != ours.input.len() {
        return Err(EngineError::Claim("the signed transaction was changed"));
    }
    let spent: Vec<TxOut> = t.psbt.inputs.iter().zip(&ours.input).map(|(i, txin)| {
        i.witness_utxo.clone()
            .or_else(|| i.non_witness_utxo.as_ref().and_then(|p| p.output.get(txin.previous_output.vout as usize).cloned()))
            .ok_or(EngineError::Claim("a coin's previous transaction is missing; scan again"))
    }).collect::<Result<_, _>>()?;
    let secp = Secp256k1::verification_only();
    let not_verified = |_| EngineError::Claim("a signature does not verify; nothing was sent");
    let mut tx = ours.clone();
    for (i, s) in signed.inputs.iter().enumerate() {
        let spk = &spent[i].script_pubkey;
        if t.script_type == ClaimScriptType::P2trKeyPath {
            let sig = s.tap_key_sig.as_ref().ok_or(EngineError::Claim("the signer did not sign every coin"))?;
            if sig.len() != 65 || sig[64] as u32 != SIGHASH_ALL_UNIFIED {
                return Err(EngineError::Claim(WRONG_SIGNATURE_MESSAGE));
            }
            if !spk.is_p2tr() {
                return Err(EngineError::Claim("coin type does not match"));
            }
            let key = XOnlyPublicKey::from_slice(&spk.as_bytes()[2..34]).map_err(|_| EngineError::Claim("coin type does not match"))?;
            verify_schnorr_unified(&secp, &tx, i, &spent, sig, &key).map_err(not_verified)?;
            tx.input[i].witness = Witness::from_slice(&[sig.clone()]);
            continue;
        }
        let (pk, _) = t.psbt.inputs[i].bip32_derivation.iter().next().ok_or(EngineError::Claim("coin is missing its key path"))?;
        let sig = s.partial_sigs.iter().find(|(k, _)| k.as_slice() == pk.serialize()).map(|(_, v)| v)
            .ok_or(EngineError::Claim("the signer did not sign every coin"))?;
        if sig.last().map(|b| *b as u32) != Some(SIGHASH_ALL_UNIFIED) {
            return Err(EngineError::Claim(WRONG_SIGNATURE_MESSAGE));
        }
        let cpk = CompressedPublicKey(*pk);
        match t.script_type {
            ClaimScriptType::P2wpkh => {
                if *spk != ScriptBuf::new_p2wpkh(&cpk.wpubkey_hash()) {
                    return Err(EngineError::Claim("coin type does not match"));
                }
                verify_ecdsa_unified(&secp, &tx, i, &spent, ScriptType::WitnessV0, &p2wpkh_script_code(pk), sig, pk).map_err(not_verified)?;
                tx.input[i].witness = Witness::from_slice(&[sig.clone(), pk.serialize().to_vec()]);
            }
            ClaimScriptType::P2shP2wpkh => {
                let redeem = ScriptBuf::new_p2wpkh(&cpk.wpubkey_hash());
                if *spk != ScriptBuf::new_p2sh(&redeem.script_hash()) {
                    return Err(EngineError::Claim("coin type does not match"));
                }
                verify_ecdsa_unified(&secp, &tx, i, &spent, ScriptType::WitnessV0, &p2wpkh_script_code(pk), sig, pk).map_err(not_verified)?;
                tx.input[i].script_sig = Builder::new().push_slice(push(redeem.as_bytes())).into_script();
                tx.input[i].witness = Witness::from_slice(&[sig.clone(), pk.serialize().to_vec()]);
            }
            ClaimScriptType::P2pkh => {
                if *spk != ScriptBuf::new_p2pkh(&cpk.pubkey_hash()) {
                    return Err(EngineError::Claim("coin type does not match"));
                }
                verify_ecdsa_unified(&secp, &tx, i, &spent, ScriptType::Bare, spk, sig, pk).map_err(not_verified)?;
                tx.input[i].script_sig = Builder::new().push_slice(push(sig)).push_slice(push(&pk.serialize())).into_script();
            }
            ClaimScriptType::P2trKeyPath => unreachable!(),
        }
    }
    Ok(tx)
}
```

- [ ] **Step 6: Add `finish` to `claim/mod.rs`**

```rust
use crate::ur::{psbt_from_payload, ScannedPayload};

impl ClaimSession {
    pub async fn finish(&mut self, signed: &[ScannedPayload], client: &ElectrumClient) -> Result<Vec<Txid>, EngineError> {
        let plan = self.plan.as_ref().ok_or(EngineError::Claim("review the claim first"))?;
        if signed.len() != plan.txs.len() {
            return Err(EngineError::Claim("scan one signed QR code for each claim transaction"));
        }
        // Verify every transaction before broadcasting any.
        let mut finals = Vec::with_capacity(plan.txs.len());
        for (t, p) in plan.txs.iter().zip(signed) {
            let bytes = psbt_from_payload(p)?;
            let s = psbt_sigs::read_signed_psbt(&bytes)?;
            finals.push(verify::verify_and_finalize(t, &s)?);
        }
        let mut txids = Vec::with_capacity(finals.len());
        for (n, tx) in finals.iter().enumerate() {
            match client.broadcast(tx).await {
                Ok(id) => txids.push(id),
                Err(e) => return Err(EngineError::Server(format!("{n} of {} claim transactions were sent; the next one failed: {e}", finals.len()))),
            }
        }
        Ok(txids)
    }
}
```

- [ ] **Step 7: Run tests**

Run: `cargo test -p wren-engine claim::`
Expected: PASS (all claim tests, including 5 new verification tests).

- [ ] **Step 8: Commit**

```bash
git add core/wren-engine/src/claim
git commit -m "feat(engine): verify signer PSBTs (0x21 only, byte-identical tx), finalize all types, broadcast"
```

---

### Task 20: `WrenEngine` — the public API

**Files:**
- Create: `core/wren-engine/src/engine.rs`
- Modify: `core/wren-engine/src/lib.rs` (add `pub mod engine; pub use engine::*;`), `core/wren-engine/Cargo.toml` (feature `test-hooks`)
- Test: inline tests in `engine.rs`

**Interfaces:**
- Consumes: everything above.
- Produces (the surface sub-plan 2 wraps with UniFFI; names follow spec §3.1):
```rust
pub struct EngineConfig { pub data_dir: PathBuf, pub params: ChainParams, pub server: ServerAddress, pub tor: bool }
#[derive(Clone, Debug, Default)]
pub struct ConnectionStatus { pub connected: bool, pub verified: bool, pub tip_height: Option<u32>, pub protocol: Option<String>, pub tor: bool }
#[derive(Clone, Debug)]
pub struct ClaimReview { pub claim_id: u64, pub destination: String, pub total_in: Amount, pub total_fee: Amount, pub total_out: Amount, pub transactions: Vec<ClaimReviewTx> }
#[derive(Clone, Debug)]
pub struct ClaimReviewTx { pub inputs: usize, pub amount: Amount, pub fee: Amount, pub qr_frames: Vec<String> }
pub struct WrenEngine;
impl WrenEngine {
    pub fn new(cfg: EngineConfig) -> Result<Self, EngineError>;
    pub async fn create_wallet(&self) -> Result<Vec<String>, EngineError>;                 // createWallet -> words (shown once)
    pub async fn restore_wallet(&self, words: &str, passphrase: &str) -> Result<(), EngineError>;
    pub async fn load_wallet(&self, words: &str, passphrase: &str) -> Result<(), EngineError>;
    pub async fn set_server(&self, server: ServerAddress);
    pub async fn connect(&self) -> Result<ConnectionStatus, EngineError>;
    pub async fn sync(&self) -> Result<(), EngineError>;
    pub async fn connection_status(&self) -> ConnectionStatus;
    pub async fn balance(&self) -> Result<Balance, EngineError>;
    pub async fn transactions(&self) -> Result<Vec<TxSummary>, EngineError>;
    pub async fn next_receive_address(&self) -> Result<String, EngineError>;
    pub async fn fee_estimates(&self) -> Result<FeeEstimates, EngineError>;
    pub async fn build_payment(&self, destination: &str, amount: Option<Amount>, send_all: bool, fee_rate: FeeRate) -> Result<ReviewSummary, EngineError>;
    pub async fn sign_and_broadcast(&self, review_id: u64) -> Result<Txid, EngineError>;
    pub async fn bump_fee(&self, txid: Txid, fee_rate: FeeRate) -> Result<ReviewSummary, EngineError>;
    pub fn parse_payment_link(&self, s: &str) -> Result<PaymentLink, EngineError>;
    pub fn make_payment_link(&self, address: &str, amount: Option<Amount>) -> Result<String, EngineError>;
    pub fn claim_supported_types(&self) -> Vec<ClaimScriptType>;
    pub async fn claim_start(&self, xpub: ScannedPayload) -> Result<u64, EngineError>;
    pub async fn claim_scan(&self, claim_id: u64, search_further: bool) -> Result<FoundSummary, EngineError>;
    pub async fn claim_build(&self, claim_id: u64, destination: Option<&str>, fee_rate: FeeRate) -> Result<ClaimReview, EngineError>;
    pub async fn claim_finish(&self, claim_id: u64, signed: Vec<ScannedPayload>) -> Result<Vec<Txid>, EngineError>;
    pub async fn claim_cancel(&self, claim_id: u64);
    #[cfg(feature = "test-hooks")] pub async fn debug_sign_review(&self, review_id: u64) -> Result<Transaction, EngineError>;
}
```
Rules enforced here: nothing that sends (payment or claim) runs unless the current server connection reached the tip through the verified header chain (`EngineError::NotVerified`); a wrong-chain server leaves the engine unverified; Claim requires Wren's own wallet to exist first (spec §5); one claim session at a time, and `claim_finish`/`claim_cancel` drop it.

- [ ] **Step 1: Add the feature to `Cargo.toml`**

```toml
[features]
default = []
test-hooks = []
```

- [ ] **Step 2: Write the failing tests** (in `engine.rs`)

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::testutil::{serve, MockWorld, TestChain, ABANDON};

    fn engine(tc: &TestChain) -> (tempfile::TempDir, WrenEngine) {
        let dir = tempfile::tempdir().unwrap();
        let e = WrenEngine::new(EngineConfig {
            data_dir: dir.path().to_path_buf(),
            params: tc.params(),
            server: ServerAddress::Tcp { host: "unused".into(), port: 1 },
            tor: false,
        }).unwrap();
        (dir, e)
    }

    #[tokio::test]
    async fn sending_requires_a_verified_server() {
        let tc = TestChain::new(100, 20, 105, 1);
        let (_d, e) = engine(&tc);
        e.restore_wallet(ABANDON, "").await.unwrap();
        let r = e.build_payment("bcrt1qw508d6qejxtdg4y5r3zarvary0c5xw7kygt080", Some(Amount::from_sat(1000)), false, FeeRate::from_sat_per_vb(1).unwrap()).await;
        assert!(matches!(r, Err(EngineError::NotVerified)));
    }

    #[tokio::test]
    async fn wrong_chain_server_is_refused_and_engine_stays_unverified() {
        let tc = TestChain::new(100, 20, 105, 1);
        let (_d, e) = engine(&tc);
        e.restore_wallet(ABANDON, "").await.unwrap();
        let other = MockWorld::new(TestChain::new(100, 20, 105, 7));
        assert!(matches!(e.connect_with_stream(serve(other)).await, Err(EngineError::WrongChain)));
        assert!(!e.connection_status().await.verified);
        assert!(matches!(e.claim_scan(1, false).await, Err(_)));
    }

    #[tokio::test]
    async fn verified_connection_syncs_and_reports_tip() {
        let tc = TestChain::new(100, 20, 105, 1);
        let (_d, e) = engine(&tc);
        e.restore_wallet(ABANDON, "").await.unwrap();
        let st = e.connect_with_stream(serve(MockWorld::new(tc.clone()))).await.unwrap();
        assert!(st.verified);
        assert_eq!(st.tip_height, Some(119));
        e.sync().await.unwrap();
    }

    #[tokio::test]
    async fn claim_requires_wallet_and_matching_id_and_cancel_drops_session() {
        let tc = TestChain::new(100, 20, 105, 1);
        let (_d, e) = engine(&tc);
        let xpub = crate::ur::ScannedPayload::Text(crate::testutil::old_account_text(crate::xpub_import::ClaimScriptType::P2wpkh));
        assert!(e.claim_start(xpub.clone()).await.is_err(), "Wren wallet must exist first");
        e.restore_wallet(ABANDON, "").await.unwrap();
        let id = e.claim_start(xpub).await.unwrap();
        assert!(e.claim_active().await);
        e.claim_cancel(id + 1).await;
        assert!(e.claim_active().await, "wrong id does not cancel");
        e.claim_cancel(id).await;
        assert!(!e.claim_active().await);
    }

    #[tokio::test]
    async fn create_returns_12_words_once_and_refuses_second_wallet() {
        let tc = TestChain::new(100, 20, 105, 1);
        let (_d, e) = engine(&tc);
        assert_eq!(e.create_wallet().await.unwrap().len(), 12);
        assert!(e.create_wallet().await.is_err());
    }
}
```

- [ ] **Step 3: Run to verify failure**

Run: `cargo test -p wren-engine engine::`
Expected: FAIL to compile — `WrenEngine` missing.

- [ ] **Step 4: Implement**

```rust
//! The public engine API. Holds all state behind one async mutex.
use crate::claim::{ClaimSession, FoundSummary};
use crate::electrum::rpc::BoxStream;
use crate::electrum::transport::{self, ServerAddress};
use crate::electrum::ElectrumClient;
use crate::error::EngineError;
use crate::headerchain::HeaderChain;
use crate::params::ChainParams;
use crate::payment::{self, FeeEstimates, PaymentAmount, PendingPayments, ReviewSummary};
use crate::sync::{load_chain, save_chain, sync_headers, sync_wallet};
use crate::ur::ScannedPayload;
use crate::uri::{make_payment_link, parse_payment_link, PaymentLink};
use crate::wallet::{generate_words, TxSummary, WrenWallet};
use crate::xpub_import::{import_accounts, ClaimScriptType};
use bdk_wallet::bitcoin::{Address, Amount, FeeRate, Txid};
use bdk_wallet::Balance;
use std::path::PathBuf;
use std::str::FromStr;
use std::time::{SystemTime, UNIX_EPOCH};
use tokio::sync::Mutex;

pub struct EngineConfig {
    pub data_dir: PathBuf,
    pub params: ChainParams,
    pub server: ServerAddress,
    pub tor: bool,
}

#[derive(Clone, Debug, Default)]
pub struct ConnectionStatus {
    pub connected: bool,
    pub verified: bool,
    pub tip_height: Option<u32>,
    pub protocol: Option<String>,
    pub tor: bool,
}

#[derive(Clone, Debug)]
pub struct ClaimReviewTx {
    pub inputs: usize,
    pub amount: Amount,
    pub fee: Amount,
    pub qr_frames: Vec<String>,
}

#[derive(Clone, Debug)]
pub struct ClaimReview {
    pub claim_id: u64,
    pub destination: String,
    pub total_in: Amount,
    pub total_fee: Amount,
    pub total_out: Amount,
    pub transactions: Vec<ClaimReviewTx>,
}

struct State {
    server: ServerAddress,
    chain: HeaderChain,
    client: Option<ElectrumClient>,
    verified: bool,
    wallet: Option<WrenWallet>,
    pending: PendingPayments,
    claim: Option<(u64, ClaimSession)>,
    next_claim_id: u64,
}

pub struct WrenEngine {
    cfg: EngineConfig,
    st: Mutex<State>,
}

fn now() -> u64 {
    SystemTime::now().duration_since(UNIX_EPOCH).map(|d| d.as_secs()).unwrap_or(0)
}

impl WrenEngine {
    pub fn new(cfg: EngineConfig) -> Result<Self, EngineError> {
        std::fs::create_dir_all(&cfg.data_dir).map_err(|e| EngineError::Wallet(e.to_string()))?;
        let chain = match load_chain(cfg.params.clone(), &cfg.data_dir.join("headers.bin")) {
            Ok(Some(c)) => c,
            _ => HeaderChain::new(cfg.params.clone()), // missing or corrupt: re-verify from the anchor
        };
        let st = State { server: cfg.server.clone(), chain, client: None, verified: false, wallet: None,
            pending: PendingPayments::default(), claim: None, next_claim_id: 0 };
        Ok(WrenEngine { cfg, st: Mutex::new(st) })
    }

    fn db(&self) -> PathBuf {
        self.cfg.data_dir.join("wallet.sqlite")
    }

    pub async fn create_wallet(&self) -> Result<Vec<String>, EngineError> {
        let words = generate_words()?;
        let w = WrenWallet::create(&self.db(), self.cfg.params.network, &words.join(" "), "")?;
        self.st.lock().await.wallet = Some(w);
        Ok(words)
    }

    pub async fn restore_wallet(&self, words: &str, passphrase: &str) -> Result<(), EngineError> {
        let w = WrenWallet::create(&self.db(), self.cfg.params.network, words, passphrase)?;
        self.st.lock().await.wallet = Some(w);
        Ok(())
    }

    pub async fn load_wallet(&self, words: &str, passphrase: &str) -> Result<(), EngineError> {
        let w = WrenWallet::open(&self.db(), self.cfg.params.network, words, passphrase)?;
        self.st.lock().await.wallet = Some(w);
        Ok(())
    }

    pub async fn set_server(&self, server: ServerAddress) {
        let mut st = self.st.lock().await;
        st.server = server;
        st.client = None;
        st.verified = false;
    }

    pub async fn connect(&self) -> Result<ConnectionStatus, EngineError> {
        let server = self.st.lock().await.server.clone();
        if self.cfg.tor {
            return Err(EngineError::Network("Tor is enabled; use connect_tor (Task 21)".into()));
        }
        let stream = transport::connect(&server).await?;
        self.connect_with_stream(stream).await
    }

    pub(crate) async fn connect_with_stream(&self, stream: BoxStream) -> Result<ConnectionStatus, EngineError> {
        let mut st = self.st.lock().await;
        st.verified = false;
        st.client = None;
        let client = ElectrumClient::connect(stream).await?;
        let st = &mut *st;
        sync_headers(&client, &mut st.chain).await?;
        save_chain(&st.chain, &self.cfg.data_dir.join("headers.bin"))?;
        st.client = Some(client);
        st.verified = true;
        Ok(Self::status_of(st, self.cfg.tor))
    }

    fn status_of(st: &State, tor: bool) -> ConnectionStatus {
        ConnectionStatus {
            connected: st.client.is_some(),
            verified: st.verified,
            tip_height: st.chain.tip_height(),
            protocol: st.client.as_ref().map(|c| c.protocol().to_string()),
            tor,
        }
    }

    pub async fn connection_status(&self) -> ConnectionStatus {
        Self::status_of(&*self.st.lock().await, self.cfg.tor)
    }

    pub async fn sync(&self) -> Result<(), EngineError> {
        let mut guard = self.st.lock().await;
        let st = &mut *guard;
        if !st.verified {
            return Err(EngineError::NotVerified);
        }
        let client = st.client.as_ref().ok_or(EngineError::NotVerified)?;
        if let Err(e) = sync_headers(client, &mut st.chain).await {
            st.verified = false;
            return Err(e);
        }
        save_chain(&st.chain, &self.cfg.data_dir.join("headers.bin"))?;
        if let Some(w) = st.wallet.as_mut() {
            sync_wallet(client, &st.chain, w.bdk_mut(), now()).await?;
            w.persist()?;
        }
        Ok(())
    }

    fn wallet_mut(st: &mut State) -> Result<&mut WrenWallet, EngineError> {
        st.wallet.as_mut().ok_or(EngineError::Wallet("no wallet is loaded".into()))
    }

    pub async fn balance(&self) -> Result<Balance, EngineError> {
        Ok(Self::wallet_mut(&mut *self.st.lock().await)?.balance())
    }

    pub async fn transactions(&self) -> Result<Vec<TxSummary>, EngineError> {
        Ok(Self::wallet_mut(&mut *self.st.lock().await)?.transactions())
    }

    pub async fn next_receive_address(&self) -> Result<String, EngineError> {
        Ok(Self::wallet_mut(&mut *self.st.lock().await)?.next_receive_address()?.to_string())
    }

    pub async fn fee_estimates(&self) -> Result<FeeEstimates, EngineError> {
        let st = self.st.lock().await;
        payment::fee_estimates(st.client.as_ref().ok_or(EngineError::NotVerified)?).await
    }

    pub async fn build_payment(&self, destination: &str, amount: Option<Amount>, send_all: bool, fee_rate: FeeRate) -> Result<ReviewSummary, EngineError> {
        let mut guard = self.st.lock().await;
        let st = &mut *guard;
        if !st.verified {
            return Err(EngineError::NotVerified);
        }
        let link = parse_payment_link(destination, self.cfg.params.network)?;
        let amt = if send_all {
            PaymentAmount::SendAll
        } else {
            PaymentAmount::Exact(amount.or(link.amount).ok_or(EngineError::Input("enter an amount"))?)
        };
        let tip = st.chain.tip_height().ok_or(EngineError::NotVerified)?;
        let w = st.wallet.as_mut().ok_or(EngineError::Wallet("no wallet is loaded".into()))?;
        payment::build_payment(w, &self.cfg.params, tip, &mut st.pending, &link.address, amt, fee_rate)
    }

    pub async fn sign_and_broadcast(&self, review_id: u64) -> Result<Txid, EngineError> {
        let mut guard = self.st.lock().await;
        let st = &mut *guard;
        if !st.verified {
            return Err(EngineError::NotVerified);
        }
        let client = st.client.as_ref().ok_or(EngineError::NotVerified)?;
        let w = st.wallet.as_mut().ok_or(EngineError::Wallet("no wallet is loaded".into()))?;
        payment::sign_and_broadcast(w, &mut st.pending, review_id, client, now()).await
    }

    pub async fn bump_fee(&self, txid: Txid, fee_rate: FeeRate) -> Result<ReviewSummary, EngineError> {
        let mut guard = self.st.lock().await;
        let st = &mut *guard;
        if !st.verified {
            return Err(EngineError::NotVerified);
        }
        let w = st.wallet.as_mut().ok_or(EngineError::Wallet("no wallet is loaded".into()))?;
        payment::bump_fee(w, &mut st.pending, txid, fee_rate)
    }

    #[cfg(feature = "test-hooks")]
    pub async fn debug_sign_review(&self, review_id: u64) -> Result<bdk_wallet::bitcoin::Transaction, EngineError> {
        let st = self.st.lock().await;
        let w = st.wallet.as_ref().ok_or(EngineError::Wallet("no wallet".into()))?;
        let psbt = st.pending.peek(review_id).ok_or(EngineError::Input("no such review"))?;
        payment::sign_pending(w, psbt)
    }

    pub fn parse_payment_link(&self, s: &str) -> Result<PaymentLink, EngineError> {
        parse_payment_link(s, self.cfg.params.network)
    }

    pub fn make_payment_link(&self, address: &str, amount: Option<Amount>) -> Result<String, EngineError> {
        let a = Address::from_str(address).map_err(|_| EngineError::Input("not a valid address"))?
            .require_network(self.cfg.params.network).map_err(|_| EngineError::Input("address is for a different network"))?;
        Ok(make_payment_link(&a, amount))
    }

    pub fn claim_supported_types(&self) -> Vec<ClaimScriptType> {
        crate::claim::claim_supported_types()
    }

    pub async fn claim_start(&self, xpub: ScannedPayload) -> Result<u64, EngineError> {
        let mut st = self.st.lock().await;
        if st.wallet.is_none() {
            return Err(EngineError::Claim("create your Wren wallet first"));
        }
        let accounts = import_accounts(&xpub, self.cfg.params.network)?;
        let session = ClaimSession::start(accounts, self.cfg.params.network)?;
        st.next_claim_id += 1;
        let id = st.next_claim_id;
        st.claim = Some((id, session)); // replaces (and drops) any earlier session
        Ok(id)
    }

    fn session(st: &mut State, id: u64) -> Result<&mut ClaimSession, EngineError> {
        match st.claim.as_mut() {
            Some((cid, s)) if *cid == id => Ok(s),
            _ => Err(EngineError::Claim("this claim is no longer active; start again")),
        }
    }

    pub async fn claim_scan(&self, claim_id: u64, search_further: bool) -> Result<FoundSummary, EngineError> {
        let mut guard = self.st.lock().await;
        let st = &mut *guard;
        if !st.verified {
            return Err(EngineError::NotVerified);
        }
        let client = st.client.as_ref().ok_or(EngineError::NotVerified)?;
        let session = match st.claim.as_mut() {
            Some((cid, s)) if *cid == claim_id => s,
            _ => return Err(EngineError::Claim("this claim is no longer active; start again")),
        };
        session.scan(client, &st.chain, search_further, now()).await
    }

    pub async fn claim_build(&self, claim_id: u64, destination: Option<&str>, fee_rate: FeeRate) -> Result<ClaimReview, EngineError> {
        let mut guard = self.st.lock().await;
        let st = &mut *guard;
        let dest = match destination {
            Some(s) => parse_payment_link(s, self.cfg.params.network)?.address,
            None => Self::wallet_mut(st)?.next_receive_address()?,
        };
        let plan = Self::session(st, claim_id)?.build(&dest, fee_rate)?.clone();
        Ok(ClaimReview {
            claim_id,
            destination: plan.destination,
            total_in: plan.total_in,
            total_fee: plan.total_fee,
            total_out: plan.total_out,
            transactions: plan.txs.into_iter().map(|t| ClaimReviewTx { inputs: t.inputs, amount: t.amount, fee: t.fee, qr_frames: t.qr_frames }).collect(),
        })
    }

    pub async fn claim_finish(&self, claim_id: u64, signed: Vec<ScannedPayload>) -> Result<Vec<Txid>, EngineError> {
        let mut guard = self.st.lock().await;
        let st = &mut *guard;
        if !st.verified {
            return Err(EngineError::NotVerified);
        }
        let client = st.client.as_ref().ok_or(EngineError::NotVerified)?;
        let session = match st.claim.as_mut() {
            Some((cid, s)) if *cid == claim_id => s,
            _ => return Err(EngineError::Claim("this claim is no longer active; start again")),
        };
        let txids = session.finish(&signed, client).await?;
        st.claim = None; // wipe the xpub and everything derived from it
        Ok(txids)
    }

    pub async fn claim_cancel(&self, claim_id: u64) {
        let mut st = self.st.lock().await;
        if matches!(&st.claim, Some((cid, _)) if *cid == claim_id) {
            st.claim = None;
        }
    }

    #[cfg(test)]
    pub(crate) async fn claim_active(&self) -> bool {
        self.st.lock().await.claim.is_some()
    }
}
```
Add to `lib.rs`: `pub mod engine; pub use engine::{ClaimReview, ClaimReviewTx, ConnectionStatus, EngineConfig, WrenEngine};`.

- [ ] **Step 5: Run tests**

Run: `cargo test -p wren-engine engine::`
Expected: PASS (5 tests).

- [ ] **Step 6: Run the full suite and the gate**

Run: `core/scripts/check.sh`
Expected: fmt, clippy (`-D warnings`), all tests, and `cargo audit` pass.

- [ ] **Step 7: Commit**

```bash
git add core/wren-engine
git commit -m "feat(engine): WrenEngine public API with verified-server and claim-session rules"
```

---

### Task 21: Tor transport (embedded Arti)

**Files:**
- Create: `core/wren-engine/src/electrum/tor.rs`
- Modify: `core/wren-engine/Cargo.toml` (optional deps + `tor` feature), `core/wren-engine/src/electrum/transport.rs` (`connect_via`), `core/wren-engine/src/engine.rs` (Tor-aware `connect`), `core/scripts/check.sh` (clippy with `--features tor`)
- Test: `core/wren-engine/tests/tor.rs` (ignored; run manually)

**Interfaces:**
- Consumes: `transport::{ServerAddress, PinnedCert}` (Task 10).
- Produces:
```rust
#[cfg(feature = "tor")]
pub struct Tor;
#[cfg(feature = "tor")]
impl Tor {
    pub async fn bootstrap(state_dir: &Path, cache_dir: &Path) -> Result<Tor, EngineError>;
    pub async fn stream(&self, host: &str, port: u16) -> Result<BoxStream, EngineError>;
}
pub async fn connect_via(addr: &ServerAddress, tor: Option<&Tor>) -> Result<BoxStream, EngineError>; // transport.rs
```
With Tor on (the default the app will use), **every** connection goes through Tor: onion addresses directly, clearnet hosts through an exit. `ServerAddress::Onion { tls: true }` requires a pinned certificate (onion + self-signed is the Start9/Umbrel norm). Building Arti for iOS and Android is part of sub-plan 2; this task proves the transport on the desktop.

- [ ] **Step 1: Add the dependencies**

Run:
```bash
cd core/wren-engine
cargo add arti-client@0.46 --optional --no-default-features --features tokio,rustls,onion-service-client,compression
cargo tree -e normal -i tor-rtcompat --features arti-client 2>/dev/null | head -1   # prints the tor-rtcompat version arti-client uses
cargo add tor-rtcompat@<that exact version> --optional --no-default-features --features tokio,rustls
```
Then add to `Cargo.toml`:
```toml
[features]
default = []
test-hooks = []
tor = ["dep:arti-client", "dep:tor-rtcompat"]
```
Expected: `cargo build --features tor` succeeds.

- [ ] **Step 2: Write `electrum/tor.rs`**

```rust
//! Embedded Tor (Arti). Only compiled with `--features tor`.
use super::rpc::BoxStream;
use crate::error::EngineError;
use arti_client::config::TorClientConfigBuilder;
use arti_client::TorClient;
use std::path::Path;
use tor_rtcompat::PreferredRuntime;

pub struct Tor {
    client: TorClient<PreferredRuntime>,
}

fn net<E: std::fmt::Display>(e: E) -> EngineError {
    EngineError::Network(e.to_string())
}

impl Tor {
    pub async fn bootstrap(state_dir: &Path, cache_dir: &Path) -> Result<Tor, EngineError> {
        let cfg = TorClientConfigBuilder::from_directories(state_dir, cache_dir).build().map_err(net)?;
        let client = TorClient::create_bootstrapped(cfg).await.map_err(net)?;
        Ok(Tor { client })
    }

    pub async fn stream(&self, host: &str, port: u16) -> Result<BoxStream, EngineError> {
        let s = self.client.connect((host, port)).await.map_err(net)?;
        Ok(Box::new(s))
    }
}
```
Add to `electrum/mod.rs`: `#[cfg(feature = "tor")] pub mod tor;`.

- [ ] **Step 3: Generalize the transport**

In `transport.rs`, change `tls_connect` to wrap any stream and add `connect_via`:
```rust
async fn tls_over(stream: BoxStream, host: &str, config: ClientConfig) -> Result<BoxStream, EngineError> {
    let tls = TlsConnector::from(Arc::new(config)).connect(server_name(host)?, stream).await.map_err(net)?;
    Ok(Box::new(tls))
}

async fn tls_connect(host: &str, port: u16, config: ClientConfig) -> Result<BoxStream, EngineError> {
    let tcp: BoxStream = Box::new(TcpStream::connect((host, port)).await.map_err(net)?);
    tls_over(tcp, host, config).await
}

fn tls_config(pinned: Option<[u8; 32]>) -> Result<ClientConfig, EngineError> {
    let p = provider();
    let builder = ClientConfig::builder_with_provider(p.clone()).with_safe_default_protocol_versions().map_err(net)?;
    Ok(match pinned {
        Some(pin) => builder.dangerous().with_custom_certificate_verifier(Arc::new(PinnedCert { pin, provider: p })).with_no_client_auth(),
        None => {
            let mut roots = RootCertStore::empty();
            roots.extend(webpki_roots::TLS_SERVER_ROOTS.iter().cloned());
            builder.with_root_certificates(roots).with_no_client_auth()
        }
    })
}

#[cfg(feature = "tor")]
pub use super::tor::Tor;
#[cfg(not(feature = "tor"))]
pub enum Tor {}

pub async fn connect_via(addr: &ServerAddress, tor: Option<&Tor>) -> Result<BoxStream, EngineError> {
    #[cfg(feature = "tor")]
    if let Some(t) = tor {
        return match addr {
            ServerAddress::Tcp { host, port } => t.stream(host, *port).await,
            ServerAddress::Tls { host, port, pinned_sha256 } => tls_over(t.stream(host, *port).await?, host, tls_config(*pinned_sha256)?).await,
            ServerAddress::Onion { host, port, tls: false } => t.stream(host, *port).await,
            ServerAddress::Onion { tls: true, .. } => Err(EngineError::Input("an onion server with TLS needs a pinned certificate; use Tls with the .onion host")),
        };
    }
    #[cfg(not(feature = "tor"))]
    let _ = tor;
    match addr {
        ServerAddress::Onion { .. } => Err(EngineError::Network("onion servers need Tor".into())),
        _ => connect(addr).await,
    }
}
```
and make `connect`'s `Tls` arm call `tls_connect(host, *port, tls_config(*pinned_sha256)?)`. (A `.onion` host with a pinned certificate is expressed as `ServerAddress::Tls { host: "<x>.onion", … }` routed through Tor.)

- [ ] **Step 4: Make the engine use Tor when enabled**

In `engine.rs`, add a lazily bootstrapped Tor handle to `State` — the field `#[cfg(feature = "tor")] tor: Option<std::sync::Arc<crate::electrum::tor::Tor>>`, initialized with `#[cfg(feature = "tor")] tor: None,` in `WrenEngine::new` — then replace `connect` and add the two cfg-selected helpers:
```rust
    pub async fn connect(&self) -> Result<ConnectionStatus, EngineError> {
        let server = self.st.lock().await.server.clone();
        let stream = if self.cfg.tor { self.tor_stream(&server).await? } else { transport::connect_via(&server, None).await? };
        self.connect_with_stream(stream).await
    }

    #[cfg(feature = "tor")]
    async fn tor_stream(&self, server: &ServerAddress) -> Result<BoxStream, EngineError> {
        let tor = {
            let mut st = self.st.lock().await;
            if st.tor.is_none() {
                let t = crate::electrum::tor::Tor::bootstrap(&self.cfg.data_dir.join("tor-state"), &self.cfg.data_dir.join("tor-cache")).await?;
                st.tor = Some(std::sync::Arc::new(t));
            }
            st.tor.clone().unwrap()
        };
        transport::connect_via(server, Some(&tor)).await
    }

    #[cfg(not(feature = "tor"))]
    async fn tor_stream(&self, _server: &ServerAddress) -> Result<BoxStream, EngineError> {
        Err(EngineError::Network("this build does not include Tor".into()))
    }
```

- [ ] **Step 5: Write the manual Tor test**

`tests/tor.rs`:
```rust
//! Manual: WREN_TOR_TEST="<host>:<port>:tcp" or "<host>:<port>:tls:<sha256hex>" \
//!   cargo test -p wren-engine --features tor --test tor -- --ignored --nocapture
#![cfg(feature = "tor")]
use wren_engine::electrum::transport::{connect_via, ServerAddress, Tor};
use wren_engine::electrum::ElectrumClient;

#[tokio::test]
#[ignore]
async fn tor_reaches_an_electrum_server() {
    let spec = std::env::var("WREN_TOR_TEST").expect("set WREN_TOR_TEST");
    let parts: Vec<&str> = spec.split(':').collect();
    let (host, port) = (parts[0].to_string(), parts[1].parse().unwrap());
    let addr = match parts[2] {
        "tcp" => ServerAddress::Tcp { host, port },
        _ => ServerAddress::Tls { host, port, pinned_sha256: Some(hex::decode(parts[3]).unwrap().try_into().unwrap()) },
    };
    let dir = tempfile::tempdir().unwrap();
    let tor = Tor::bootstrap(&dir.path().join("s"), &dir.path().join("c")).await.unwrap();
    let c = ElectrumClient::connect(connect_via(&addr, Some(&tor)).await.unwrap()).await.unwrap();
    println!("protocol {}", c.protocol());
    assert!(c.tip().await.unwrap().0 > 961_640);
}
```
Add `cargo add --dev hex@0.4` is already a normal dependency; nothing else needed.

- [ ] **Step 6: Extend `check.sh`**

Add after the clippy line: `cargo clippy --all-targets --features tor -- -D warnings`.

- [ ] **Step 7: Run**

Run: `core/scripts/check.sh` — Expected: all pass (Tor test is ignored).
Manual (owner's device, during development only): `WREN_TOR_TEST="<fork node onion>:50001:tcp" cargo test -p wren-engine --features tor --test tor -- --ignored --nocapture` — Expected: prints the protocol and passes.

- [ ] **Step 8: Commit**

```bash
git add core/wren-engine core/scripts/check.sh core/Cargo.lock
git commit -m "feat(engine): embedded Arti Tor transport (feature tor)"
```

---

### Task 22: Local test chain — full lifecycle, cross-signing, replay proof, wrong-chain proof

**Files:**
- Create: `core/integration/docker-compose.yml`, `core/integration/knots/Dockerfile`, `core/integration/core/Dockerfile`, `core/integration/fulcrum/Dockerfile`, `core/integration/run.sh`
- Create: `core/wren-engine/tests/regtest.rs`
- Modify: `core/wren-engine/Cargo.toml` (dev-dependency `ureq`)

**Interfaces:**
- Consumes: `WrenEngine` + `test-hooks` (Task 20), `ChainParams::regtest` (Task 6), `QrCollector` / `psbt_from_payload` (Task 16).
- Produces: `core/integration/run.sh` — one command that proves, against real node software:
  1. **Lifecycle:** restore, receive, sync, send, send all, speed up (RBF), confirm.
  2. **Cross-sign byte equality:** Wren's signed transaction equals Knots `signrawtransactionwithkey … "ALL|UNIFIED"` byte for byte (spec §8).
  3. **Replay proof:** transactions spending coins that exist on **both** chains (created before the regtest split) are accepted by fork Knots and **rejected with a script/signature error** by Bitcoin Core (spec §8).
  4. **Claim, all four script types:** signed by Knots `walletprocesspsbt … "ALL|UNIFIED"` as the signer stand-in; accepted on the fork, rejected by Core; a standard-signature claim is refused with the fork-firmware message.
  5. **Wrong-chain proof:** pointed at an Electrum server backed by Bitcoin Core, the engine refuses to verify and refuses to send (spec §8).

How the split is reproduced: fork Knots runs regtest with `-testactivationheight=blake2b@200`; Bitcoin Core peers with it and follows the shared SHA256d history until height 199, then rejects block 200 (a v2 header) and stays on the legacy side. Coins mined before 200 exist on both chains, exactly like real pre-split coins.

**Deviations from spec §8, flagged for Gate B:** (a) the automated byte-exact cross-sign reference is **Knots** (the consensus implementation) rather than Shrike; Shrike is still exercised in the real-chain trial. (b) The automated SeedSigner stand-in is **Knots `walletprocesspsbt`**; the real SeedSigner device test runs in sub-plan 2 (it needs the phone app for QR display).

- [ ] **Step 1: Write the Dockerfiles**

`core/integration/knots/Dockerfile`:
```dockerfile
FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y --no-install-recommends curl ca-certificates gnupg && rm -rf /var/lib/apt/lists/*
ARG KNOTS_VERSION=29.4.2.knots20260508
ARG LUKE_KEY=1A3E761F19D2CC7785C5502EA291A2C45D0C504A
WORKDIR /tmp/knots
RUN set -eux; base="https://bitcoinknots.org/files/29.x/${KNOTS_VERSION}"; \
    curl -fsSLO "$base/SHA256SUMS"; curl -fsSLO "$base/SHA256SUMS.asc"; \
    gpg --keyserver hkps://keys.openpgp.org --recv-keys "$LUKE_KEY" || gpg --keyserver hkps://keyserver.ubuntu.com --recv-keys "$LUKE_KEY"; \
    gpg --status-fd 1 --verify SHA256SUMS.asc SHA256SUMS 2>/dev/null | grep "VALIDSIG" | grep -q "$LUKE_KEY"; \
    f="bitcoin-${KNOTS_VERSION}-aarch64-linux-gnu.tar.gz"; grep -q " $f\$" SHA256SUMS; \
    curl -fsSLO "$base/$f"; sha256sum --ignore-missing -c SHA256SUMS; \
    tar xzf "$f" --strip-components=1 -C /usr/local; rm -rf /tmp/knots
ENTRYPOINT ["bitcoind"]
```

`core/integration/core/Dockerfile`:
```dockerfile
FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y --no-install-recommends curl ca-certificates gnupg && rm -rf /var/lib/apt/lists/*
ARG CORE_VERSION=29.1
ARG FANQUAKE_KEY=E777299FC265DD04793070EB944D35F9AC3DB76A
WORKDIR /tmp/core
RUN set -eux; base="https://bitcoincore.org/bin/bitcoin-core-${CORE_VERSION}"; \
    curl -fsSLO "$base/SHA256SUMS"; curl -fsSLO "$base/SHA256SUMS.asc"; \
    gpg --keyserver hkps://keys.openpgp.org --recv-keys "$FANQUAKE_KEY" || gpg --keyserver hkps://keyserver.ubuntu.com --recv-keys "$FANQUAKE_KEY"; \
    gpg --status-fd 1 --verify SHA256SUMS.asc SHA256SUMS 2>/dev/null | grep "VALIDSIG" | grep -q "$FANQUAKE_KEY"; \
    f="bitcoin-${CORE_VERSION}-aarch64-linux-gnu.tar.gz"; grep -q " $f\$" SHA256SUMS; \
    curl -fsSLO "$base/$f"; sha256sum --ignore-missing -c SHA256SUMS; \
    tar xzf "$f" --strip-components=1 -C /usr/local; rm -rf /tmp/core
ENTRYPOINT ["bitcoind"]
```

`core/integration/fulcrum/Dockerfile`:
```dockerfile
FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y --no-install-recommends curl ca-certificates gnupg && rm -rf /var/lib/apt/lists/*
ARG TAG=v2.1.2-blake-3
ARG KYLE_KEY=A47D99B6DB0D715D40C59A2023AE8A8EA7E24E38
WORKDIR /tmp/f
RUN set -eux; base="https://github.com/privkeyio/Fulcrum/releases/download/${TAG}"; \
    curl -fsSLO "$base/privkeyio-signing-key.asc"; gpg --import privkeyio-signing-key.asc; gpg --list-keys "$KYLE_KEY"; \
    curl -fsSLO "$base/Fulcrum-2.1.2-blake-shasums.txt"; curl -fsSLO "$base/Fulcrum-2.1.2-blake-shasums.txt.asc"; \
    gpg --status-fd 1 --verify Fulcrum-2.1.2-blake-shasums.txt.asc Fulcrum-2.1.2-blake-shasums.txt 2>/dev/null | grep "VALIDSIG" | grep -q "$KYLE_KEY"; \
    curl -fsSLO "$base/Fulcrum-2.1.2-blake-arm64-linux.tar.gz"; sha256sum --ignore-missing -c Fulcrum-2.1.2-blake-shasums.txt; \
    mkdir -p /opt/fulcrum /data; tar xzf Fulcrum-2.1.2-blake-arm64-linux.tar.gz --strip-components=1 -C /opt/fulcrum; rm -rf /tmp/f
ENTRYPOINT ["/opt/fulcrum/Fulcrum"]
```
Every image refuses to build unless the release signature is valid **and** made by the pinned key.

- [ ] **Step 2: Write `docker-compose.yml` and `run.sh`**

```yaml
services:
  knots:
    build: ./knots
    command: ["-regtest", "-server", "-txindex", "-listen=1", "-bind=0.0.0.0", "-rpcuser=wren", "-rpcpassword=wren",
              "-rpcbind=0.0.0.0", "-rpcallowip=0.0.0.0/0", "-fallbackfee=0.0001", "-testactivationheight=blake2b@200"]
    ports: ["18443:18443"]
  core:
    build: ./core
    command: ["-regtest", "-server", "-txindex", "-rpcuser=wren", "-rpcpassword=wren", "-rpcbind=0.0.0.0",
              "-rpcallowip=0.0.0.0/0", "-fallbackfee=0.0001", "-addnode=knots:18444"]
    ports: ["28443:18443"]
  fulcrum:
    build: ./fulcrum
    command: ["-D", "/data", "-b", "knots:18443", "-u", "wren", "-p", "wren", "-t", "0.0.0.0:50001"]
    ports: ["50001:50001"]
    depends_on: [knots]
  fulcrum-core:
    build: ./fulcrum
    command: ["-D", "/data", "-b", "core:18443", "-u", "wren", "-p", "wren", "-t", "0.0.0.0:50001"]
    ports: ["60001:50001"]
    depends_on: [core]
```

`core/integration/run.sh`:
```bash
#!/usr/bin/env bash
# Brings up a fresh fork-regtest + legacy-regtest world and runs the engine's integration suite.
set -euo pipefail
cd "$(dirname "$0")"
docker compose down -v --remove-orphans
docker compose up -d --build
for port in 18443 28443; do
  for _ in $(seq 60); do
    curl -sf --user wren:wren -H 'content-type: text/plain' \
      --data '{"jsonrpc":"1.0","id":"w","method":"getblockchaininfo","params":[]}' "http://127.0.0.1:${port}/" >/dev/null && break
    sleep 1
  done
done
export WREN_KNOTS_RPC=http://127.0.0.1:18443 WREN_CORE_RPC=http://127.0.0.1:28443
export WREN_FULCRUM=127.0.0.1:50001 WREN_FULCRUM_CORE=127.0.0.1:60001
cd ../wren-engine
cargo test --features test-hooks --test regtest -- --ignored --test-threads=1 --nocapture
```
Run: `chmod +x core/integration/run.sh && cd core/wren-engine && cargo add --dev ureq@2 --features json`

- [ ] **Step 3: Write `tests/regtest.rs`**

```rust
//! Integration suite against real node software. Run via core/integration/run.sh.
#![cfg(feature = "test-hooks")]
use base64::Engine as _;
use bdk_wallet::bitcoin::bip32::{DerivationPath, Xpriv, Xpub};
use bdk_wallet::bitcoin::secp256k1::Secp256k1;
use bdk_wallet::bitcoin::{consensus::encode::serialize, Amount, FeeRate, Network, PrivateKey};
use bdk_wallet::keys::bip39::{Language, Mnemonic};
use serde_json::{json, Value};
use std::str::FromStr;
use wren_engine::electrum::transport::ServerAddress;
use wren_engine::params::ChainParams;
use wren_engine::ur::{psbt_from_payload, QrCollector, ScannedPayload};
use wren_engine::{EngineConfig, EngineError, WrenEngine};

const WREN_WORDS: &str = "abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon about";
const OLD_WORDS: &str = "legal winner thank year wave sausage worth useful legal winner thank yellow";
const SPLIT: u64 = 200;

fn env(k: &str) -> String {
    std::env::var(k).unwrap_or_else(|_| panic!("{k} not set; run core/integration/run.sh"))
}

/// JSON-RPC call; Ok(result) or Err(error object).
fn rpc(url: &str, wallet: Option<&str>, method: &str, params: Value) -> Result<Value, Value> {
    let u = match wallet { Some(w) => format!("{url}/wallet/{w}"), None => url.to_string() };
    let auth = format!("Basic {}", base64::engine::general_purpose::STANDARD.encode("wren:wren"));
    let body = json!({"jsonrpc": "1.0", "id": "t", "method": method, "params": params});
    let resp: Value = match ureq::post(&u).set("Authorization", &auth).send_json(body) {
        Ok(r) => r.into_json().unwrap(),
        Err(ureq::Error::Status(_, r)) => r.into_json().unwrap(),
        Err(e) => panic!("rpc transport error: {e}"),
    };
    if resp["error"].is_null() { Ok(resp["result"].clone()) } else { Err(resp["error"].clone()) }
}
fn knots(w: Option<&str>, m: &str, p: Value) -> Value { rpc(&env("WREN_KNOTS_RPC"), w, m, p).unwrap() }
fn core(w: Option<&str>, m: &str, p: Value) -> Result<Value, Value> { rpc(&env("WREN_CORE_RPC"), w, m, p) }
fn height() -> u64 { knots(None, "getblockcount", json!([])).as_u64().unwrap() }
fn mine(n: u64) {
    let a = knots(Some("miner"), "getnewaddress", json!([])).as_str().unwrap().to_string();
    knots(None, "generatetoaddress", json!([n, a]));
}

fn server(var: &str) -> ServerAddress {
    let (h, p) = env(var).rsplit_once(':').map(|(h, p)| (h.to_string(), p.parse().unwrap())).unwrap();
    ServerAddress::Tcp { host: h, port: p }
}

fn engine(var: &str) -> (tempfile::TempDir, WrenEngine) {
    let dir = tempfile::tempdir().unwrap();
    let e = WrenEngine::new(EngineConfig { data_dir: dir.path().into(), params: ChainParams::regtest(SPLIT as u32), server: server(var), tor: false }).unwrap();
    (dir, e)
}

/// Connects, then syncs until the engine's tip equals Knots' height (Fulcrum indexes asynchronously).
async fn sync_to_tip(e: &WrenEngine) {
    for _ in 0..120 {
        if !e.connection_status().await.verified {
            let _ = e.connect().await;
        }
        let _ = e.sync().await;
        if e.connection_status().await.tip_height == Some(height() as u32) {
            return;
        }
        tokio::time::sleep(std::time::Duration::from_millis(500)).await;
    }
    panic!("engine did not reach the Knots tip");
}

fn master(words: &str) -> Xpriv {
    let seed = Mnemonic::parse_in(Language::English, words).unwrap().to_seed("");
    Xpriv::new_master(Network::Regtest, &seed).unwrap()
}

fn with_checksum(desc: &str) -> String {
    knots(None, "getdescriptorinfo", json!([desc]))["descriptor"].as_str().unwrap().to_string()
}

/// (wrapper, purpose) for each claim script type.
const TYPES: [(&str, u32); 4] = [("wpkh", 84), ("sh-wpkh", 49), ("pkh", 44), ("tr", 86)];
fn wrap(kind: &str, inner: &str) -> String {
    match kind { "sh-wpkh" => format!("sh(wpkh({inner}))"), k => format!("{k}({inner})") }
}

fn assert_legacy_chain_rejects(tx_hex: &str) {
    let r = core(None, "testmempoolaccept", json!([[tx_hex]])).unwrap();
    let res = &r[0];
    assert_eq!(res["allowed"], json!(false), "legacy chain accepted a Wren transaction: {res}");
    let reason = res["reject-reason"].as_str().unwrap_or("");
    assert!(reason.contains("script-verify-flag"), "rejected for the wrong reason (must be a signature failure, not missing inputs): {reason}");
}

#[tokio::test]
#[ignore]
async fn fork_regtest_suite() {
    let secp = Secp256k1::new();
    // --- shared history (both chains) ---
    knots(None, "createwallet", json!(["miner"]));
    mine(101);
    let (_d, e) = engine("WREN_FULCRUM");
    e.restore_wallet(WREN_WORDS, "").await.unwrap();
    let wren_addr0 = e.next_receive_address().await.unwrap();
    knots(Some("miner"), "sendtoaddress", json!([wren_addr0, 1.0]));

    // Old wallet (to be claimed): all four types, funded before the split.
    let old = master(OLD_WORDS);
    let old_fp = old.fingerprint(&secp);
    knots(None, "createwallet", json!(["old", false, true]));
    let mut claim_texts = Vec::new();
    let mut imports = Vec::new();
    for (kind, p) in TYPES {
        let path = DerivationPath::from_str(&format!("m/{p}'/0'/0'")).unwrap();
        let acct = old.derive_priv(&secp, &path).unwrap();
        let xpub = Xpub::from_priv(&secp, &acct);
        let prv = with_checksum(&wrap(kind, &format!("{old}/{p}h/0h/0h/0/*")));
        imports.push(json!({"desc": prv, "active": true, "range": [0, 20], "timestamp": "now", "internal": false}));
        let pub_desc = with_checksum(&wrap(kind, &format!("{xpub}/0/*")));
        let addr = knots(None, "deriveaddresses", json!([pub_desc, [0, 0]]))[0].as_str().unwrap().to_string();
        knots(Some("miner"), "sendtoaddress", json!([addr, 1.0]));
        claim_texts.push(wrap(kind, &format!("[{old_fp}/{p}h/0h/0h]{xpub}/0/*")));
    }
    knots(Some("old"), "importdescriptors", json!([imports]));
    mine(1);
    // Core must hold the same pre-split coins.
    for _ in 0..60 {
        if core(None, "getblockcount", json!([])).unwrap().as_u64() == Some(height()) { break; }
        tokio::time::sleep(std::time::Duration::from_millis(500)).await;
    }
    assert_eq!(core(None, "getblockcount", json!([])).unwrap().as_u64(), Some(height()), "Core did not follow shared history");

    // --- split: mine past BLAKE2b activation on Knots only ---
    mine(SPLIT + 5 - height());
    sync_to_tip(&e).await;
    assert_eq!(e.balance().await.unwrap().confirmed, Amount::from_sat(100_000_000));

    // --- cross-sign byte equality + replay proof on a pre-split coin ---
    let dest = knots(Some("miner"), "getnewaddress", json!([])).as_str().unwrap().to_string();
    let review = e.build_payment(&dest, Some(Amount::from_sat(40_000_000)), false, FeeRate::from_sat_per_vb(2).unwrap()).await.unwrap();
    let ours = e.debug_sign_review(review.review_id).await.unwrap();
    let mut unsigned = ours.clone();
    for i in &mut unsigned.input { i.witness.clear(); }
    let key0 = master(WREN_WORDS).derive_priv(&secp, &DerivationPath::from_str("m/84'/0'/0'/0/0").unwrap()).unwrap();
    let wif = PrivateKey::new(key0.private_key, Network::Regtest).to_wif();
    let prev = &ours.input[0].previous_output;
    let prev_tx = knots(None, "getrawtransaction", json!([prev.txid.to_string(), true]));
    let prev_out = &prev_tx["vout"][prev.vout as usize];
    let knots_signed = knots(None, "signrawtransactionwithkey", json!([
        hex::encode(serialize(&unsigned)), [wif],
        [{"txid": prev.txid.to_string(), "vout": prev.vout, "scriptPubKey": prev_out["scriptPubKey"]["hex"], "amount": prev_out["value"]}],
        "ALL|UNIFIED"]));
    assert_eq!(knots_signed["hex"].as_str().unwrap(), hex::encode(serialize(&ours)), "Wren and Knots signatures differ");
    let txid = e.sign_and_broadcast(review.review_id).await.unwrap();
    assert_legacy_chain_rejects(&hex::encode(serialize(&ours)));
    mine(1);
    sync_to_tip(&e).await;
    assert!(knots(None, "getrawtransaction", json!([txid.to_string(), true]))["confirmations"].as_u64().unwrap() >= 1);

    // --- post-split lifecycle: receive, send, speed up, send all ---
    let a1 = e.next_receive_address().await.unwrap();
    knots(Some("miner"), "sendtoaddress", json!([a1, 0.5]));
    mine(1);
    sync_to_tip(&e).await;
    let r = e.build_payment(&dest, Some(Amount::from_sat(5_000_000)), false, FeeRate::from_sat_per_vb(1).unwrap()).await.unwrap();
    let slow = e.sign_and_broadcast(r.review_id).await.unwrap();
    let b = e.bump_fee(slow, FeeRate::from_sat_per_vb(5).unwrap()).await.unwrap();
    let fast = e.sign_and_broadcast(b.review_id).await.unwrap();
    let pool = knots(None, "getrawmempool", json!([]));
    assert!(pool.as_array().unwrap().iter().any(|t| t == &json!(fast.to_string())));
    assert!(!pool.as_array().unwrap().iter().any(|t| t == &json!(slow.to_string())));
    mine(1);
    sync_to_tip(&e).await;

    // --- claims: all four types; standard signature refused first ---
    for (n, text) in claim_texts.iter().enumerate() {
        let id = e.claim_start(ScannedPayload::Text(text.clone())).await.unwrap();
        let found = e.claim_scan(id, false).await.unwrap();
        assert_eq!(found.total, Amount::from_sat(100_000_000), "type {n}");
        let review = e.claim_build(id, None, FeeRate::from_sat_per_vb(2).unwrap()).await.unwrap();
        let mut qr = QrCollector::new();
        for f in &review.transactions[0].qr_frames {
            if qr.receive(f).unwrap() { break; }
        }
        let psbt_bytes = psbt_from_payload(&qr.finish().unwrap()).unwrap();
        if n == 0 {
            // A signer without the fork firmware: strip the 0x21 request and sign standard.
            let mut p = bdk_wallet::bitcoin::Psbt::deserialize(&psbt_bytes).unwrap();
            for i in &mut p.inputs { i.sighash_type = None; }
            let std_b64 = base64::engine::general_purpose::STANDARD.encode(p.serialize());
            let signed = knots(Some("old"), "walletprocesspsbt", json!([std_b64, true, "ALL", true, false]));
            let res = e.claim_finish(id, vec![ScannedPayload::Text(signed["psbt"].as_str().unwrap().to_string())]).await;
            match res {
                Err(EngineError::Claim(m)) => assert!(m.contains("fork firmware")),
                other => panic!("standard-signed claim was not refused: {other:?}"),
            }
        }
        let b64 = base64::engine::general_purpose::STANDARD.encode(&psbt_bytes);
        let signed = knots(Some("old"), "walletprocesspsbt", json!([b64, true, "ALL|UNIFIED", true, false]));
        let txids = e.claim_finish(id, vec![ScannedPayload::Text(signed["psbt"].as_str().unwrap().to_string())]).await.unwrap();
        let hex_tx = knots(None, "getrawtransaction", json!([txids[0].to_string()])).as_str().unwrap().to_string();
        assert_legacy_chain_rejects(&hex_tx);
        mine(1);
        sync_to_tip(&e).await;
    }

    // --- wrong-chain proof ---
    let a = knots(Some("miner"), "getnewaddress", json!([])).as_str().unwrap().to_string();
    core(None, "createwallet", json!(["c"])).ok();
    let ca = core(Some("c"), "getnewaddress", json!([])).unwrap();
    core(None, "generatetoaddress", json!([30, ca])).unwrap(); // legacy chain grows past 200 with v1 headers
    let (_d2, legacy) = engine("WREN_FULCRUM_CORE");
    legacy.restore_wallet(WREN_WORDS, "").await.unwrap();
    let mut refused = false;
    for _ in 0..60 {
        match legacy.connect().await {
            Err(EngineError::WrongChain) => { refused = true; break; }
            _ => tokio::time::sleep(std::time::Duration::from_millis(500)).await,
        }
    }
    assert!(refused, "legacy-chain server was not refused");
    assert!(!legacy.connection_status().await.verified);
    assert!(matches!(legacy.build_payment(&a, Some(Amount::from_sat(1000)), false, FeeRate::from_sat_per_vb(1).unwrap()).await, Err(EngineError::NotVerified)));
}
```

- [ ] **Step 4: Run it**

Run: `core/integration/run.sh`
Expected: images build (all signature checks pass), then `test fork_regtest_suite ... ok`.
If an image fails its signature or checksum step, stop: do not bypass verification — report which release artifact failed.

- [ ] **Step 5: Commit**

```bash
git add core/integration core/wren-engine/tests/regtest.rs core/wren-engine/Cargo.toml core/Cargo.lock
git commit -m "test(engine): regtest split world — lifecycle, Knots cross-sign, replay and wrong-chain proofs, 4-type claims"
```

- [ ] **Step 6: Signing-milestone review (Gate B)**

Record in `docs/superpowers/reviews/<YYYY-MM-DD of the review>-gate-b.md`: reviewer, commit reviewed, `run.sh` output, findings, and resolutions. Sub-plan 2 (bridge + app) starts only after this file records a pass.

---

## Coverage map (spec → tasks)

| Spec requirement | Task |
|---|---|
| §3.1 fork header parsing/verification (164-byte) | 3, 4, 5, 6, 11 |
| §3.1 SIGHASH_UNIFIED signing, byte-exact | 7, 8, 22 |
| §3.1 engine API list | 20 (UniFFI export in sub-plan 2) |
| §3.1 Tor via Arti | 21 (mobile build in sub-plan 2) |
| §3.4 `xbt:` links, never `bitcoin:` | 15 |
| §4 12-word seed, OS randomness, passphrase | 12 |
| §4 replay-safe signing always on | 8, 14, 18, 19 (no code path signs otherwise) |
| §4 wrong-chain guard | 6, 11, 20, 22 |
| §4 BIP84 path, Shrike-restorable | 12 (BIP84 official vector) |
| §4.1 hardware-signer-only claim; xpub in memory only | 16, 17, 20 |
| §4.1 scanning with gap 20 + search further | 17 |
| §4.1 supported types (all four; legacy needs SeedSigner setting) | 16, 17, 18, 22 |
| §4.1 unsigned claim: 0x21 on every input, no change | 18 |
| §4.1 the five checks; wrong-signature message | 19, 22 |
| §4.1 splitting large claims | 18 |
| §5 send / send all / fee tiers / speed up / activity | 14, 12, 20, 22 |
| §5 custom server incl. self-signed onion nodes | 10, 21 |
| §8 vectors, local test chain, replay proof, wrong-chain proof | 1, 7, 22 |
| §8 cargo audit clean | 1 (`check.sh`) |
| App-side only (sub-plan 2): backup confirm-3-words, app lock (Wren PIN + optional biometrics, spec §4), secure storage, screenshot blocking, screens, visual design, SeedSigner real-device test, SeedSigner guide | — |
| Server (sub-plan 3): §7 | — |
