# BLAKE2b fork — wallet tooling facts for Wren

Collected 2026-09-28 by a research subagent (public GitHub, read-only). Verify against the
linked repos/tags before relying on a line number.

**Ecosystem index:** https://github.com/bitcoin-blake/awesome-bitcoin-blake
("SIGHASH_UNIFIED flag 0x20 per signature, so SIGHASH_ALL becomes 0x21"; header v2 = 164 bytes;
activation mainnet 961,640, testnet4 150,308; address prefixes + P2P magic unchanged.)

**Kyle = Kyle Santiago, GitHub org `privkeyio`.** Shrike, the SeedSigner build
and Fulcrum BLAKE2b are all his, signed with `A47D99B6DB0D715D40C59A2023AE8A8EA7E24E38`.

## Shrike (reference wallet, cross-signing oracle)

- https://github.com/privkeyio/shrike (Sparrow fork); fork logic in submodule
  https://github.com/privkeyio/drongo. Latest release v2.5.5-blake2b.27 (2026-09-23).
- Releases: `SHA256SUMS`, `SHA256SUMS.asc`, `privkeyio-signing-key.asc`;
  `gpg --verify SHA256SUMS.asc SHA256SUMS && sha256sum --ignore-missing -c SHA256SUMS`.
- PSBT: `PSBT_IN_SIGHASH_TYPE = 0x03`, value uint32 LE → UNIFIED_ALL = `21000000`
  (drongo `psbt/PSBTInput.java` L26, L204-220, write L498-502).
- `SigHash.java`: `UNIFIED_ALL = 0x21`, `UNIFIED_FLAG = 0x20`. Taproot DEFAULT → 0x21
  ("DEFAULT appends no hash type byte, so there is nothing to carry the bit").
- `UnifiedScriptType`: BARE (P2PKH+P2SH) 0, WITNESS_V0 1, TAPROOT 2, TAPSCRIPT 3
  (`PSBTInput.java` L1669-1713).
- Digest `Transaction.hashForUnifiedSignature` (`Transaction.java` ~L834-962): tagged hash
  "UnifiedSighash"; commits to every input's amount + spk; 5-byte locktime. A PSBT missing any
  input's UTXO cannot be signed.
- Headers `protocol/BlockHeader.java`: `V1_LENGTH = 80; V2_LENGTH = 164;
  HEADER_V2_FLAG = 0x80000000L`. v2 id = Blake2b256(asicInput) XOR key mask
  (`taggedHash(TAG_XOR_MASK, xorKey)` with leading bits cleared).
- Electrum client: `BlockHeaders.java` splits concatenated headers by each header's own length
  (80/164), accepts `hex` (<1.6) and list (>=1.6) forms. `ElectrumServer.java`
  `SUPPORTED_VERSIONS "1.3","1.8"` (electrs header-v2 build refuses below 1.8).
  `HEADERS_CHUNK_SIZE = 2016`. Headers re-verified from pinned checkpoints
  (`HeaderCheckpoints`, `HeaderStore.java`).
- Test vectors (drongo): `src/test/resources/unified_sighash.json` (166, copied from Knots
  commit 4a180066), `block_header_v2.json`, `headers/mainnet-activation.txt`,
  `testnet4_v2_headers.tsv`; regtest walkthrough `docs/unified-sighash-regtest.md`.
- Not retrieved: exact code that writes 0x21 into new PSBTs (`PSBT.java`, `WalletTransaction.java`).

## SeedSigner `v0.8.7-unified-sighash.6` (Claim signer)

- App https://github.com/privkeyio/seedsigner (branch `unified-sighash`); OS
  https://github.com/privkeyio/seedsigner-os. Release
  https://github.com/privkeyio/seedsigner/releases/tag/v0.8.7-unified-sighash.6 (2026-09-05).
  Reproducible from seedsigner-os `793a723`; embit pinned by sha256 `bc8540ec…`.
- Verify: same gpg/SHA256SUMS flow; "Good signature from Kyle Santiago".
- **Signs UNIFIED only if the PSBT declares 0x21.** No declared type → standard signature.
  `models/psbt_parser.py`: `SIGNABLE_SIGHASH_TYPES = {None, DEFAULT, ALL, UNIFIED|ALL}`; type must
  be identical on every input (`REFUSED_MIXED`), no partial signs (`REFUSED_PARTIAL`).
  → **Wren MUST set `PSBT_IN_SIGHASH_TYPE = 0x21` on every claim input.**
- Signable: P2PKH, P2SH-P2WPKH, P2WPKH, P2WSH, P2SH-P2WSH, P2SH multisig, P2TR. Tests cover
  bare/P2SH, v0, key path, tapscript vs Knots regtest (`tests/unified/`). Default enabled script
  types: Native Segwit, Nested Segwit, Taproot (legacy OFF by default — user must enable it in
  SeedSigner settings to claim from a 1… wallet).
- Screen shows "Unified sighash (0x21)" / "Standard sighash (0x01)"
  (`gui/screens/psbt_screens.py`); device verifies sigs against the displayed type.
- PSBT QR in: `UR:CRYPTO-PSBT/` (UR2, `urtypes`), Specter `pXofY`, base64, base43, BBQr.
  Out: animated `UR("crypto-psbt")` via `UREncoder`.
- Xpub export (`settings_definition.py`): `urca` animated UR `crypto-account` (default), `sta`
  static, `spl` Specter legacy `[fingerprint/derivation]xpub`.

## Fulcrum BLAKE2b (default server software)

- https://github.com/privkeyio/Fulcrum, release `v2.1.2-blake-3`; StartOS
  https://github.com/privkeyio/fulcrum-startos.
- Protocol unchanged: `MinProtocolVersion{1,4,0}`, `MaxProtocolVersion{1,6,0}`; `max` 2016.
  Headers served at real length (80 below 961,640, 164 at/above → 328 hex chars) in
  `blockchain.block.header`, `blockchain.headers.subscribe`, and 1.6 `headers` list. Below 1.6,
  concatenated `hex` is variable-width — split by version bit.
- Does NOT report the fork point ("a header announces itself through the top bit").
- `-3` fixed wrong merkle root for `cp_height` >= 961,640.
- Ports 50001 TCP / 50002 SSL.
- electrs-pruned (paulscode) uses protocol **1.8** (`docs/electrum-header-v2.md`). **Wren must
  negotiate 1.4-1.8 and accept both list and variable-width concatenated forms.**

## Other wallets / building blocks

- **Rust (directly usable):** https://github.com/privkeyio/rust-bitcoin (164-byte header, branch
  `0.32.xx`, the line bdk/electrum-client pin); https://github.com/privkeyio/rust-electrum-client
  (splits headers by parsed length). Python: https://github.com/privkeyio/embit.
- Lazarus Key (Bitkey fork, Rust + KMP): https://github.com/AwokenLazarus/lazarus-key —
  `Wallet::xbt_protect_from_replay` re-signs with 0x21; negotiates 1.8; reads 80/164 headers;
  regtest only, Android only.
- NOT replay-safe (do not copy): pyblock-blake2b (plain SIGHASH_ALL), pyblock-watch.
- Canary (watch-only, privkeyio); protocol write-up https://github.com/Kilombino/blake2b-light-clients.
- Public Electrum: `electrum.lazarus-xbt.xyz:50002:s` (electrs + header-v2 patch, protocol 1.8).
