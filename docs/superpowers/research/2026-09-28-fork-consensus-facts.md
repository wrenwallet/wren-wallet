# BLAKE2b fork — consensus facts for Wren engine

**Source of truth:** signed tag `v29.4.2.knots20260508` of github.com/bitcoinknots/bitcoin
(published 2026-09-21, tag signed by Luke Dashjr `1A3E761F19D2CC7785C5502EA291A2C45D0C504A`).
Base: `https://github.com/bitcoinknots/bitcoin/blob/v29.4.2.knots20260508/`.
Fork code first shipped in `v29.4.1.knots20260508` (2026-09-02); release notes cite knots#357
(unified sighash), #358 (RDTS), #359 (BLAKE2b). `doc/unified-sighash.md` identical in both.
PR #357 on GitHub was closed unmerged (head `privkeyio/bitcoin:hf-sighash-opt-in`), but its
code is in the tagged releases. Collected 2026-09-28 by a research subagent; verify line
numbers against the tag before porting.

## Heights

| Height | Meaning | Source |
|---|---|---|
| 961,632 | First BIP110 block — where the chains split. "Pre-split" coins = created below this. | `src/kernel/chainparams.cpp:240-242` |
| 961,639 | Last SHA256d block on the fork chain | same |
| 961,640 | First BLAKE2b block (v2 headers mandatory from here) | `chainparams.cpp:129`, `:240-242` |
| 973,440 | Long coinbase maturity starts | `chainparams.cpp:143-146` |

Checkpoint, block 961,640: `0000000000000050c1e5f69672f459293be14f46e5a494e7a8c8541396f18eeb`

## SIGHASH_UNIFIED

- `SIGHASH_UNIFIED = 0x20` (`src/script/interpreter.h:41`); script flag
  `SCRIPT_VERIFY_UNIFIED_SIGHASH = (1U << 22)` (`:174`), consensus-on at >= BLAKE2b height
  (`src/validation.cpp:2752-2753`).
- Wren signs with `0x21` = `SIGHASH_ALL | SIGHASH_UNIFIED` (ECDSA and Schnorr alike).
- Opt-in per signature; signatures without the bit remain valid on the fork (NOT replay-safe).
- Covers all script types via a `script_type` byte: 0 = bare/P2PKH/P2SH, 1 = segwit v0
  (P2WPKH, P2WSH, P2SH-wrapped), 2 = taproot key path, 3 = tapscript.
- Algorithm `SignatureHashUnified`, `src/script/interpreter.cpp:1635-1754`; spec
  `doc/unified-sighash.md:151-214`. Digest = BIP340 tagged hash, tag `"UnifiedSighash"`
  (`interpreter.cpp:1491`). (Older PR comments say "2026Sighash" — obsolete.)
- Message, in order:
  1. `0x00` epoch
  2. hash_type (1 byte)
  3. nVersion (4 LE)
  4. nLockTime (4 LE) + `0x00` (5 bytes total)
  5. if !ANYONECANPAY: `sha_prevouts`, `sha_amounts`, `sha_scripts`, `sha_sequences` (single SHA256, as BIP341)
  6. if `(ht & 0x1f)` not NONE/SINGLE: `sha_outputs`
  7. script_type (1 byte)
  8. if ANYONECANPAY: outpoint (36) + spent TxOut (value + compact-size spk) + nSequence; else input index u32 LE
  9. types 0/1: compact-size-prefixed `scriptCode` (BIP143 P2PKH-style for P2WPKH; type 0 applies
     legacy FindAndDelete of the sig); types 2/3: annex flag byte (+ annex hash)
  10. if SINGLE: `SHA256(ser(vout[nIn]))` — invalid if that output doesn't exist (no "1" bug)
  11. type 3 only: tapleaf hash, `0x00` key version, codesep pos (4 LE)
- Commits to ALL inputs' amounts and scriptPubKeys → signer needs every spent TxOut (PSBT
  `witness_utxo`/`non_witness_utxo` must be present for every input).
- Taproot: only ALL/NONE/SINGLE (±ANYONECANPAY) with UNIFIED; `SIGHASH_DEFAULT` unusable →
  Schnorr sigs always 65 bytes. Fee estimation must budget 65 (Knots had a 64-byte bug).
- Invalid on legacy chain: different message for same byte; BIP341 rejects the hash type;
  STRICTENC policy rejects 0x2x for legacy/v0 (`doc/unified-sighash.md:60-63`, `:123-126`).
- Message signing stays legacy.
- **Test vectors:** `src/test/data/unified_sighash.json` — 166 vectors (76 bare/P2SH, 66 segwit
  v0, 12 key path, 12 tapscript). Columns: `scriptCode, rawTx, inIdx, hashType, scriptType,
  spentOutputs [[value, spk]], sighash`. Sighash bytes are raw order. Second reference impl:
  `UnifiedSignatureHash` in `test/functional/test_framework/script.py:710`.

## Block header

- `src/primitives/block.h:103-125`; v2 flagged by nVersion top bit `0x80000000` (`:25`). All LE.
- Bytes 0-79 (legacy shape): version(4) prevhash(32) merkle(32) time_on_wire(4) nBits(4) nNonce(4).
- v2 adds 84 bytes (total 164): nonce2(4) nonce3(4) extranonce(16) time_offset(4) txcount(u16)
  flags(u8) xor_key_mask_clear_bits(u8) xor_key(16) height(i32) mm_rhs(32).
- Effective time: if `flags & 4`, `nTime = time_on_wire + time_offset`.
- Rules (`validation.cpp:4423-4445`, `:4789-4798`, `:4452`): `flags & 0xc0 == 0`;
  `height == prev.height + 1`; height >= 961,640 must be v2 (`bad-version-blake2b`); v2 below
  that is invalid; txcount == block tx count.
- Pre-961,640 headers (incl. 961,632-961,639) are 80-byte SHA256d.
- Hash (`src/primitives/block.cpp:13-105`) — PoW hash == block hash == next prevhash:
  - v1: SHA256d.
  - v2: h1 = tagged SHA256 over 119-byte preimage (version, prevhash reversed, height, merkle,
    wire time, 0x00, nBits, txcount as u32, flags, mask bits, xor_key hash) (`:43-55`);
    h2 = TaggedHash("Merge-mining hook")(h1 ‖ 32 zero bytes ‖ mm_rhs);
    BLAKE2b-256 unkeyed of `u32 0 ‖ h2 ‖ extranonce` (52 bytes);
    second BLAKE2b-256 pass whose preimage depends on `flags & 3` (`:77-96`);
    XOR with tagged-SHA256 mask from xor_key; byte-reverse. PORT EXACTLY from source.
- Difficulty: Bitcoin 2016-block retarget, 4x clamp; at 961,640 only the target is shifted left
  by `Blake2bTargetShift = 22` and capped at powLimit (`src/pow.cpp:19-31`, `:70-73`,
  `:125-127`). PoW: hash <= target; powLimit `00000000ff…`.

## Other rules

- Long coinbase maturity: coinbase created at >= 973,440 needs 6,480 confs while spend height
  < 979,920; else 100 (`src/consensus/tx_verify.cpp:199`). (Wren never mines; affects only
  users receiving coinbase outputs directly.)
- RDTS/BIP110 consensus from 961,640 until parent MTP >= 1819756800 (2027-09-01)
  (`chainparams.cpp:141`): output spk <= 34 bytes, OP_RETURN <= 83 bytes
  (`src/consensus/consensus.h:39-40`) — P2PKH/P2SH/P2WPKH/P2WSH/P2TR fine, bare multisig not;
  witness/push elements <= 256 bytes; annex forbidden; OP_IF/NOTIF banned in tapscript; control
  block <= 7 nodes; block weight <= 800,000. Inputs spending pre-activation coins are exempt
  from script-level rules (`validation.cpp:3050-3057`).
- Policy: dust relay fee 3000 sat/kvB; min relay fee 1000 sat/kvB (`src/policy/policy.h:103,108`).
- Network params IDENTICAL to Bitcoin mainnet: magic `f9beb4d9`, P2P 8333, RPC 8332, hrp `bc`,
  base58 0/5/128, xpub `0488B21E`. Fork peers advertise `NODE_BLAKE2B = 1<<28`
  (`src/protocol.h:355`).

## Releases

- https://github.com/bitcoinknots/bitcoin/releases ;
  https://bitcoinknots.org/files/29.x/29.4.2.knots20260508/ (`SHA256SUMS.asc`, 7 sigs incl.
  `1A3E…504A` Luke and `A47D99B6DB0D715D40C59A2023AE8A8EA7E24E38` Kyle).
- Packages: github.com/paulscode/knots-blake2b-startos, github.com/paulscode/umbrel-bitcoin-blake2b.
- Not found: a written spec of the v2 header hash beyond the code (design doc is draft, no BIP).
