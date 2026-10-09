# XBT wallet ecosystem — technical notes (2026-10-09)

These notes describe what other open-source XBT (BLAKE2b-chain) wallets and related projects do, so the Gate A reviewers can compare approaches with Wren's design.
They are factual notes, not an endorsement or a critique. Every claim is dated to when it was observed and links to its source.

**Method.** Repositories were read through GitHub and project web pages on 2026-10-09. No third-party code was built or run. Claims about code refer to the default or named branch as of that date. "Not determined" means the sources read did not settle the point.

---

## 1. RedWallet (`romanmodin/RedWallet`)

**Status, platforms, license (2026-10-09).** RedWallet is a BlueWallet fork for XBT, MIT licensed, created 2026-09-27. The repo description reads "Experimental; not for use with funds." It ships as an iPhone TestFlight beta and as an Android APK on GitHub Releases and Zapstore. It has no App Store or Google Play listing. — [repo](https://github.com/romanmodin/RedWallet), [README](https://github.com/romanmodin/RedWallet/blob/main/README.md), [releases](https://github.com/romanmodin/RedWallet/releases), [ANDROID_INSTALL.md](https://github.com/romanmodin/RedWallet/blob/main/ANDROID_INSTALL.md)

**Wallet types.** BIP84 P2WPKH and BIP86 Taproot (key path only). BIP84/BIP86 watch-only wallets with external PSBT signing. RBF and CPFP. It has no multisig, Lightning or silent payments. Its only URL scheme is `redwallet:`, because `bitcoin:` and `lightning:` were removed on 2026-10-01. — [README](https://github.com/romanmodin/RedWallet/blob/main/README.md), [TAPROOT.md](https://github.com/romanmodin/RedWallet/blob/main/TAPROOT.md), [XBT_PR_REVIEW.md](https://github.com/romanmodin/RedWallet/blob/main/XBT_PR_REVIEW.md)

**Chain verification.**
- On every connect and in the settings test, the wallet compares the server's full 164-byte header at height 961,640 byte for byte with a pinned value. — [electrum-checkpoint.ts](https://github.com/romanmodin/RedWallet/blob/main/class/xbt/electrum-checkpoint.ts)
- The code says this "is a compatibility check, not server authentication or proof of the current chain". — same file
- The tip height comes from `blockchain.headers.subscribe` and is not validated. There is no header-chain, PoW or merkle validation, and the project says so: "The app does not claim independent confirmation verification." — [BlueElectrum.ts](https://github.com/romanmodin/RedWallet/blob/main/blue_modules/BlueElectrum.ts), [SECURITY_FOLLOWUP.md](https://github.com/romanmodin/RedWallet/blob/main/SECURITY_FOLLOWUP.md)

**Signing (0x21).**
- P2WPKH ECDSA and Taproot Schnorr both sign with `SIGHASH_ALL|SIGHASH_UNIFIED` (`0x21`). Every exported unsigned PSBT input declares `0x21`. — [TAPROOT.md](https://github.com/romanmodin/RedWallet/blob/main/TAPROOT.md), [PROJECT_SCOPE.md](https://github.com/romanmodin/RedWallet/blob/main/PROJECT_SCOPE.md)
- A pre-broadcast gate (`assertXbtUnifiedTransaction`) refuses any input that is not a 65-byte Taproot witness ending `0x21`, or a P2WPKH witness whose signature ends `0x21`. — [broadcast-validation.ts](https://github.com/romanmodin/RedWallet/blob/main/class/xbt/broadcast-validation.ts)
- Project-reported evidence: Knots regtest accepts the transactions. Tampered outputs and stripped flags are rejected. Bitcoin Core 29.0 regtest rejects the XBT witness. — [XBT_VALIDATION.md](https://github.com/romanmodin/RedWallet/blob/main/XBT_VALIDATION.md)

**Servers and Tor.**
- No server is bundled (`hardcodedPeers = []`). The user configures a Fulcrum/Electrum server and may add up to 5 TLS backups, each with an optional pinned PEM CA. — [server-backups.ts](https://github.com/romanmodin/RedWallet/blob/main/class/xbt/server-backups.ts)
- TLS is authenticated (chain and hostname checks, no insecure fallback). Plain TCP is still allowed for the primary server. — [SECURITY_FOLLOWUP.md](https://github.com/romanmodin/RedWallet/blob/main/SECURITY_FOLLOWUP.md)
- There is no embedded Tor. `.onion` access relies on an external Orbot or VPN. — [BlueElectrum.ts](https://github.com/romanmodin/RedWallet/blob/main/blue_modules/BlueElectrum.ts)

**Hardware / external signing.**
- A returned PSBT must match the in-app request. Signatures are verified against the reviewed transaction and the prevout amounts. Relabeled BTC signatures, changed outputs and pasted raw transactions are refused. — [SECURITY_FOLLOWUP.md](https://github.com/romanmodin/RedWallet/blob/main/SECURITY_FOLLOWUP.md)
- The README calls physical cold-signer interop "unverified". — [README](https://github.com/romanmodin/RedWallet/blob/main/README.md)

**Coinbase maturity.** The wallet applies 6,480 confirmations to every coinbase. It classifies coinbases from raw parent transactions it hashes itself. A missing count is treated as immature, and conflicting counts throw an error. — [coinbase-maturity.ts](https://github.com/romanmodin/RedWallet/blob/main/class/xbt/coinbase-maturity.ts), [verified-transaction.ts](https://github.com/romanmodin/RedWallet/blob/main/class/xbt/verified-transaction.ts)

**Fees.**
- Server suggestions are capped at 100 sat/vB.
- A second confirmation is required above 100 sat/vB, above 100,000 sats, or above 5% of the amount.
- RBF keeps every recipient and pays the increase only from change. CPFP handles one unconfirmed parent.
- Sources: [fee-policy.ts](https://github.com/romanmodin/RedWallet/blob/main/class/xbt/fee-policy.ts), [TAPROOT.md](https://github.com/romanmodin/RedWallet/blob/main/TAPROOT.md)

**Recovery.**
- Accepts a BIP39 phrase with an optional passphrase, as BIP84 `m/84'/0'/0'` or BIP86 `m/86'/0'/0'` (coin type 0). — `class/wallet-import.ts`, [TAPROOT.md](https://github.com/romanmodin/RedWallet/blob/main/TAPROOT.md)
- An opt-in discovery scan (added 2026-10-09) covers accounts 0–2 or 0–9 with a gap of 20 or 100. — [recovery-discovery.ts](https://github.com/romanmodin/RedWallet/blob/main/class/xbt/recovery-discovery.ts)
- Pre-split coins get a notice only; "No automatic sweep or transaction is performed." — [replay-notice.ts](https://github.com/romanmodin/RedWallet/blob/main/class/xbt/replay-notice.ts)

**Notable safety patterns.**
- A single fail-closed XBT profile across all paths, with no BTC fallback. — [profile.ts](https://github.com/romanmodin/RedWallet/blob/main/class/xbt/profile.ts)
- Every send path shows "BTC and XBT addresses look the same. Confirm that every recipient expects XBT before continuing." — [payment-warning.ts](https://github.com/romanmodin/RedWallet/blob/main/class/xbt/payment-warning.ts)
- Copied secrets are cleared from the clipboard after 30 s, and the copy stays local on iOS. — [SECURITY_FOLLOWUP.md](https://github.com/romanmodin/RedWallet/blob/main/SECURITY_FOLLOWUP.md)
- Release provenance: per-release IPA and APK hashes, plus the Android signer cert SHA-256 `7843791c…884f`. Builds are signed from a private source repo, and a published public/private source-mapping hash ties the two together. — [RELEASE.md](https://github.com/romanmodin/RedWallet/blob/main/RELEASE.md), [ANDROID_INSTALL.md](https://github.com/romanmodin/RedWallet/blob/main/ANDROID_INSTALL.md)
- An external review of PR #1 by `kwsantiago` (2026-10-01) raised seven findings, all accepted and fixed. — [PR #1](https://github.com/romanmodin/RedWallet/pull/1)

**Open items (2026-10-09).**
- [#37](https://github.com/romanmodin/RedWallet/issues/37): request for a public Fulcrum server.
- [#38](https://github.com/romanmodin/RedWallet/issues/38): Android crash on Pixel/GrapheneOS.
- `SECURITY.md` still lists the upstream BlueWallet contact. — [SECURITY.md](https://github.com/romanmodin/RedWallet/blob/main/SECURITY.md)
- PR #26 is an undeployed prototype of a leased temporary Fulcrum relay. — [PR #26](https://github.com/romanmodin/RedWallet/pull/26)

---

## 2. Paperclip Wallet, Paperclip Ark/LSP, and Kilojoin (Kilombino)

### 2.1 Paperclip Wallet (`connorslab/paperclip-wallet-app`)

**Status, platforms, license (2026-10-09).** MIT licensed, Rust. It is built on Second's Bark (Ark) with BDK for on-chain. There are two deliverables:
- a self-hosted daemon with a web UI, v0.8.3 beta, a GitHub pre-release packaged for Umbrel, StartOS 0.4 and Docker;
- a SwiftUI iOS app on branch `feature/ios-wallet` that users build themselves in Xcode, with no TestFlight build. Its latest "iOS proof of concept" CI runs on 2026-10-09 failed.

Sources: [repo](https://github.com/connorslab/paperclip-wallet-app), [releases](https://github.com/connorslab/paperclip-wallet-app/releases/tag/v0.8.3-beta.1), [mobile/README.md](https://github.com/connorslab/paperclip-wallet-app/blob/feature/ios-wallet/mobile/README.md), [Actions](https://github.com/connorslab/paperclip-wallet-app/actions)

**Chain verification.**
- Electrum backends must pass a genesis check and an activated 164-byte header check before use. The vendored client decodes variable-length headers, including batches that mix pre- and post-activation headers.
- Esplora "still needs a compatible public deployment test".
- Whether PoW or chainwork is checked was not determined.
- Source: [mobile/README.md](https://github.com/connorslab/paperclip-wallet-app/blob/feature/ios-wallet/mobile/README.md)

**Signing (0x21).**
- "There is no legacy SHA256 BTC signing fallback. A regression test verifies that signatures validate under the unified digest and fail under the BTC BIP341 digest." — [mobile/README.md](https://github.com/connorslab/paperclip-wallet-app/blob/feature/ios-wallet/mobile/README.md)
- The bark-xbt port adds explicit 0x21 Ark witnesses and reports a consensus probe in which unified transactions confirmed on XBT and their replay on BTC was rejected. — [bark-xbt](https://github.com/connorslab/bark-xbt)

**Servers and Tor.**
- The default Electrum server is `ssl://pool.paperclip-xbt.xyz:50002`, which the README describes on 2026-10-09 as "operator is preparing this endpoint". Alternatives are a custom Electrum server, Esplora, or the user's own Knots RPC.
- TLS uses public roots, or an exact SHA-256 certificate pin that fails closed if the certificate changes.
- Tor is the embedded iCepa Tor 0.4.9.13, on by default, with per-connection switches and no direct fallback. iOS may suspend it in the background.
- Before admitting funded Ark positions, the app checks the Electrum server's capabilities (`broadcast_package`, `dustrelayfee`). One endpoint lacked them on 2026-10-06.
- Sources: [mobile/README.md](https://github.com/connorslab/paperclip-wallet-app/blob/feature/ios-wallet/mobile/README.md), [TorRuntime/UPSTREAM.txt](https://github.com/connorslab/paperclip-wallet-app/blob/feature/ios-wallet/mobile/TorRuntime/UPSTREAM.txt), [electrum-ark.md](https://github.com/connorslab/paperclip-wallet-app/blob/feature/ios-wallet/mobile/electrum-ark.md)

**Hardware / external signing.**
- QR signing over BBQr or UR with a SeedSigner (privkeyio fork) or Krux (`connorslab/krux-blake2b`, "explicit unified SIGHASH_ALL (0x21) only").
- Returned signatures are re-verified against trusted prevouts and the exact reviewed transaction. Changed outputs, fees, input order, sequences, locktime, or a BTC sighash are rejected.
- The README states: "This does not claim device-to-device testing."
- Sources: [mobile/README.md](https://github.com/connorslab/paperclip-wallet-app/blob/feature/ios-wallet/mobile/README.md), [connorslab repos](https://github.com/connorslab?tab=repositories)

**Keys and recovery (iOS).**
- New wallets get 24 BIP39 words; import accepts 12 or 24. **BIP39 passphrases are not supported.**
- Derivation: Taproot `m/86h/0h/0h`, SegWit `m/84h/0h/0h`, coinjoin account `m/84h/0h/1h`.
- The seed is stored in the Keychain as `AfterFirstUnlockThisDeviceOnly`.
- The encrypted backup (AES-256-GCM with a separate random key) contains the seed plus an SQLite snapshot of the Ark state.
- Source: [mobile/README.md](https://github.com/connorslab/paperclip-wallet-app/blob/feature/ios-wallet/mobile/README.md)

**Coinbase maturity.**
- The app shows confirmed, unconfirmed and immature balances. — [mobile/README.md](https://github.com/connorslab/paperclip-wallet-app/blob/feature/ios-wallet/mobile/README.md)
- The Paperclip pool pays miners in the coinbase and applies a 6,481-confirmation wallet policy, which stays after the temporary consensus window (973,440–979,919) ends. — [paperclip-xbt.xyz](https://paperclip-xbt.xyz/)

**Notable safety patterns.**
- Payment quotes expire after 60 s and can be used once.
- "A broadcast attempt saves the exact transaction before relay; an uncertain reply is not treated as permission to send a replacement payment."
- Lightning attempts are recorded before `pay`, and reconciliation never calls `pay` again.
- Source: [mobile/README.md](https://github.com/connorslab/paperclip-wallet-app/blob/feature/ios-wallet/mobile/README.md)

**Open items (2026-10-09).**
- Issue [#4](https://github.com/connorslab/paperclip-wallet-app/issues/4) (opened 2026-10-08): an on-chain board confirmed at 976,193, but every registration retry returns `Internal error` from the Ark server.
- Defaults are migrating from `paperclippool.xyz` to `paperclip-xbt.xyz`. — [commits](https://github.com/connorslab/paperclip-wallet-app/commits/feature/ios-wallet)

### 2.2 Paperclip Lightning (LSP) and the iOS remote-node client

- **LSP (as listed on 2026-10-09).** It runs privkeyio CLN `v26.06.8-blake2b.5`.
  - Init feature bit **512** (`option_blake2b`) is mandatory, bit 515 signals unified signatures, and new channels require channel-type bit **514**.
  - "A feature declaration does not prove a peer's chain or honesty."
  - Coinbase-funded channels are rejected. The page says to commit only amounts "you are willing to lose entirely".
  - Source: [lightning.paperclip-xbt.xyz](https://lightning.paperclip-xbt.xyz/)
- **Upstream CLN.** privkeyio `blake2b.6` (2026-10-02) halts if the backend serves a non-BLAKE2b block past activation. `blake2b.7` (2026-10-09) fixes peer-crash and splice issues. Channels opened under `.1`–`.3` do not use unified signatures. — [privkeyio/lightning releases](https://github.com/privkeyio/lightning/releases)
- **Invoices.** Paperclip's CLN fork "keeps invoice encoding and genesis unchanged" (test builds 2026-09-14). — [connorslab/lightning](https://github.com/connorslab/lightning)
- **iOS remote-node client.** It controls the user's own CLN (CLNRest + rune) or LND (REST + macaroon) node. It accepts a node whose `getinfo` reports `network == "bitcoin"`, which a BTC node also reports. The "(BLAKE2b)" labels are display text. — [LightningNetwork.swift](https://github.com/connorslab/paperclip-wallet-app/blob/feature/ios-wallet/mobile/Sources/PaperclipMobile/LightningNetwork.swift)

### 2.3 Paperclip Ark (ASP)

- **Implementation.** A fork of Bark's `captaind` and watchman, labeled "not independently audited". The deployed version and image digest are published, with the note that "the embedded revision is a build declaration, not proof of a reproducible build" (2026-10-04). — [paperclip-asp](https://github.com/connorslab/paperclip-asp), [deployed-version.md](https://github.com/connorslab/paperclip-asp/blob/main/docs/deployed-version.md)
- **Costs.** An Ark-to-Ark transfer allocates 2,660 sats in recovery reserves (3,990 with change). A 10,000-sat Lightning receive leaves 5,880 spendable. Boarded funds are spendable after 3 confirmations. — [ark.paperclippool.xyz](https://ark.paperclippool.xyz/)
- **Recovery.** "A seed alone is not a complete Ark recovery backup. Refresh or exit before expiry." With a QR signer, the hardware signs only the funding transaction; the Ark keys stay on the phone. — [README](https://github.com/connorslab/paperclip-wallet-app), [mobile/README.md](https://github.com/connorslab/paperclip-wallet-app/blob/feature/ios-wallet/mobile/README.md)

### 2.4 Kilojoin (Kilombino) and the Paperclip port

- **Protocol.** Joinstr-style coinjoin with no coordinator.
  - Pools are announced on a Nostr relay (default `wss://relay.kilombino.com`), and members talk over NIP-44.
  - Inputs are P2WPKH only. Outputs are equal-denomination. Each member signs only its own input, with 0x21.
  - There is no coordinator fee. Rounds have 2–20 participants.
  - Source: [Kilojoin PROTOCOL.md](https://github.com/Kilombino/kilojoin/blob/main/docs/PROTOCOL.md)
- **Interop.** The protocol doc reports rounds between Kilowallet (Android, Apache-2.0) and the StartOS server, including on mainnet. — [PROTOCOL.md](https://github.com/Kilombino/kilojoin/blob/main/docs/PROTOCOL.md), [Kilombino/kilowallet](https://github.com/Kilombino/kilowallet)
- **Paperclip Rust port (2026-10-07; MIT AND Apache-2.0).** "Experimental. Not independently audited … No live funded cross-client round has been validated with this port yet." Its stated limits:
  - inputs are visible to participants;
  - the shared pool key lets any member disrupt a round;
  - one person can hold several seats;
  - the relay can correlate IP addresses and timing.
  - Source: [paperclip-kilojoin-rs](https://github.com/connorslab/paperclip-kilojoin-rs)
- **iOS integration.** A separate `m/84h/0h/1h` account, Tor isolation per post, and "The page must remain open during a round." — [mobile/README.md](https://github.com/connorslab/paperclip-wallet-app/blob/feature/ios-wallet/mobile/README.md)

---

## 3. Bitchat (`permissionlesstech/bitchat`, `bitchat-android`)

- **Status, license.**
  - iOS: Swift, Unlicense.
  - Android: Kotlin, changed from MIT to GPL-3.0 in v1.7.2 (2026-03-30). The Android README still says "public domain".
  - No named third-party audit was found as of 2026-10-09.
  - Sources: [bitchat iOS](https://github.com/permissionlesstech/bitchat), [bitchat-android releases](https://github.com/permissionlesstech/bitchat-android/releases)
- **Payments.** Bitchat has no on-chain wallet and no Lightning node. Android v2.0.1 (2026-08-11) recognizes Cashu tokens, adds `/pay`, and hands redemption to "a compatible external wallet or web flow". It warns: "Cashu tokens are bearer instruments; anyone who obtains a token may be able to redeem it." No payment feature was found on iOS. — [bitchat-android v2.0.1](https://github.com/permissionlesstech/bitchat-android/releases/latest)
- **Offline "bitcoin".** This means Cashu ecash, which is a claim on a custodial mint. A token handed over offline is not final, and the sender can spend it first until the receiver reconnects and swaps it at the mint. — [btcpp Insider, 2025-07-18](https://insider.btcpp.dev/p/knock-knock-its-cashu-bitch), [Forbes, 2025-07-31](https://www.forbes.com/sites/davidbirnbaum/2025/07/31/can-bitchat-build-a-free-internet-with-bitcoin/)
- **Transport.**
  - BLE gossip mesh with TTL 7, Noise XX for live sessions, and fragments of about 469 bytes.
  - Files up to 1 MiB (with explicit acceptance). Store-and-forward courier envelopes up to 16 KiB of text.
  - There is no payload type or broadcast service for third-party data.
  - Source: [WHITEPAPER.md](https://github.com/permissionlesstech/bitchat/blob/main/WHITEPAPER.md)
- **Distribution (Android v2.0.1).**
  - Peer-to-peer "share an installed APK nearby" and signature-verified update downloads.
  - Byte-for-byte reproducible builds, `BITCHAT_SHA256SUMS`, and GitHub attestations.
  - The release notes list the tag signer keys, but no APK signing-certificate fingerprint.
  - Source: [bitchat-android v2.0.1](https://github.com/permissionlesstech/bitchat-android/releases/latest)
- **Distribution context.**
  - 2026-07-23: an order directed GitHub to geoblock three Bitchat repositories in India. — [Rest of World](https://restofworld.org/2026/india-github-geoblock-jack-dorsey-bitchat/)
  - ~2026-10-03: Apple removed the app from the Indian App Store, and TestFlight there is blocked. — [Inc42, 2026-10-04](https://inc42.com/buzz/jack-dorseys-bitchat-goes-offline-in-india-apple-cites-meity-order/)
- **Fork note.** In the Sonar fork, a signed BLE packet over 512 bytes caused a crash loop on Android 13+. — [hedwig-corp/bitchat-to-sonar PR #614](https://github.com/hedwig-corp/bitchat-to-sonar/pull/614)

---

## 4. Facts that affect Wren's design

1. **XBT and BTC Lightning share the invoice prefix and genesis hash.** BOLT 11 identifies the network only by the HRP (`lnbc`) and encodes no genesis hash, and the fork keeps both unchanged; separation relies on feature bits (item 2). — [BOLT 11](https://github.com/lightning/bolts/blob/master/11-payment-encoding.md), [connorslab/lightning](https://github.com/connorslab/lightning) (2026-09-14), [future-ideas.md](../../future-ideas.md)
2. **How XBT Lightning is kept apart from BTC Lightning (privkeyio CLN `.5`+, 2026-09-28).** No new `chain_hash` or invoice prefix (both were tried and withdrawn on 2026-09-17/19). Instead: feature bit **512** is compulsory in `init` (stock BTC nodes disconnect); gossip for channels funded below 961,640 is ignored; `channel_type` bit **514** marks unified-signature channels (515 advertised); and bit **512 is set as compulsory in BOLT11 invoice features and in BOLT12 offers/invoice requests/invoices**. A stock BTC wallet refuses an XBT invoice (CLN at decode, LND at pathfinding). The reverse is not automatic: a BTC invoice carries nothing that marks it as BTC, so an XBT wallet must itself refuse any invoice without bit 512, including on decode-only and display paths. A node's self-reported `network` is `"bitcoin"` on both chains. The protocol values are maintained in `lightning-blake2b/bolts` and are "open to change until more than two implementations have mainnet channels". — [privkeyio/lightning releases](https://github.com/privkeyio/lightning/releases), [lightning.paperclip-xbt.xyz](https://lightning.paperclip-xbt.xyz/), [LightningNetwork.swift](https://github.com/connorslab/paperclip-wallet-app/blob/feature/ios-wallet/mobile/Sources/PaperclipMobile/LightningNetwork.swift)
3. **The coinbase relay rule is stricter than consensus.** Consensus requires 6,480 confirmations only for coinbases created at ≥ 973,440 and spent below 979,920. Knots relay policy applies 6,480 to every coinbase. RedWallet uses 6,480 always, and the Paperclip pool uses 6,481. Wren's Gate A amendments (Area B) already apply both rules. — [Wren consensus facts](./2026-09-28-fork-consensus-facts.md), [coinbase-maturity.ts](https://github.com/romanmodin/RedWallet/blob/main/class/xbt/coinbase-maturity.ts), [paperclip-xbt.xyz](https://paperclip-xbt.xyz/)
4. **Public XBT Electrum endpoints.**
   - `electrum.lazarus-xbt.xyz:50002` (TLS) is listed for clients that speak "Electrum protocol 1.8 and understand 164-byte headers". The page is undated, and uptime was not verified. — [lazarus-xbt.xyz/ecosystem](https://lazarus-xbt.xyz/ecosystem/)
   - `ssl://pool.paperclip-xbt.xyz:50002` was "being prepared" on 2026-10-09. — [mobile/README.md](https://github.com/connorslab/paperclip-wallet-app/blob/feature/ios-wallet/mobile/README.md)
5. **Mixed header batches occur.** Clients must handle a header batch that spans activation, with both 80-byte and 164-byte headers. — [mobile/README.md](https://github.com/connorslab/paperclip-wallet-app/blob/feature/ios-wallet/mobile/README.md)
6. **Signers and QR formats in use.** `krux-blake2b` (0x21 only) and the privkeyio SeedSigner fork are both used over **BBQr** and UR. Wren's spec specifies BC-UR only (spec §3.1). Neither project reports a device-to-device test. — [connorslab repos](https://github.com/connorslab?tab=repositories), [mobile/README.md](https://github.com/connorslab/paperclip-wallet-app/blob/feature/ios-wallet/mobile/README.md), [RedWallet README](https://github.com/romanmodin/RedWallet/blob/main/README.md)
7. **Derivation paths match BTC.** Both projects use coin type 0 (`m/84'/0'/0'`, `m/86'/0'/0'`), as Wren does for BIP84, so one seed gives the same addresses on both chains. Replay safety therefore rests on 0x21 signing.
8. **Bitchat's "offline bitcoin" is custodial Cashu ecash.** Settlement requires reconnecting to the mint. Bitchat itself relays no on-chain transactions. A signed transaction (a few hundred bytes) fits easily within Bitchat's text and file limits, but a person on the receiving end still has to broadcast it. — [bitchat-android v2.0.1](https://github.com/permissionlesstech/bitchat-android/releases/latest), [WHITEPAPER.md](https://github.com/permissionlesstech/bitchat/blob/main/WHITEPAPER.md)
9. **Release provenance practices seen.**
   - RedWallet publishes its Android signer cert SHA-256 and per-release hashes. — [ANDROID_INSTALL.md](https://github.com/romanmodin/RedWallet/blob/main/ANDROID_INSTALL.md)
   - Bitchat Android publishes reproducible builds, a checksum manifest and attestations, but no signing-cert fingerprint. — [bitchat-android v2.0.1](https://github.com/permissionlesstech/bitchat-android/releases/latest)

---

## 5. Proposed additions for review (not yet adopted)

These items are not in the [spec](../specs/2026-09-24-wren-wallet-design.md), the [Gate A amendments](../plans/2026-10-08-wren-engine-gate-a-amendments.md) or [future-ideas.md](../../future-ideas.md) as of 2026-10-09. They are offered for discussion only.

1. **Claim: state the BIP39-passphrase handling and the exact address types.**
   - *Proposal:* add three things to spec §4.1 and the Claim guide.
     - The passphrase is applied on the SeedSigner before it exports the xpub, so the xpub already reflects it, and Wren never asks for it.
     - The exported master fingerprint is shown so the user can compare it with the signer's screen.
     - The four claim script types the plan already defines (`P2wpkh`, `P2shP2wpkh`, `P2pkh`, `P2trKeyPath`), each with its path (`84'/49'/44'/86'`, coin type 0).
   - *Rationale:* spec §4.1 still says that types other than native SegWit "must be confirmed against the firmware during planning", while the plan already enumerates them. The empty-scan copy (§5, Claim step 3) mentions a "missing passphrase" but no flow explains it.
   - *Touches:* spec §4.1 and §5 (Claim steps 2–3); plan Tasks 16–18 (xpub import, scan).
2. **A physical signer test matrix.**
   - *Proposal:* the matrix covers the privkeyio SeedSigner fork plus `connorslab/krux-blake2b`. For each signer it records the firmware version, the claim types it signs with 0x21, and the PSBT size limits. It also asks whether to accept **BBQr** in addition to BC-UR, and publishes the results.
   - *Rationale:* both signers are in use over BBQr/UR (§4 item 6). Spec §10 already plans to "add other signers if their makers support SIGHASH_UNIFIED".
   - *Touches:* spec §3.1 (QR transport), §8 (Claim trials), §10; plan Task 22a; Gate B.
3. **A revised Lightning gate (future phase).**
     - *Proposal:* any Lightning path in Wren requires the peer to signal bits **512 and 514**, checks the peer's chain against Wren's own verified XBT header chain (for example, its tip hash at a known height), never trusts a node's self-reported network, **refuses any invoice or offer without compulsory bit 512 on every path (decode, display and pay)**, and ignores `f` fallbacks.
     - *Rationale:* §4 items 1–2. Invoices and offers carry bit 512 since CLN `.5`; peer feature declarations are not chain proofs.
     - *Touches:* `docs/future-ideas.md` ("XBT Lightning"); a separate Lightning note follows.
4. **Release provenance.**
   - *Proposal:*
     - publish the Android signing-cert SHA-256 and per-release artifact hashes;
     - add an in-app "Verify this app" screen showing version, commit and signer fingerprint;
     - work toward reproducible Android builds;
     - mirror the source on a second git host.
   - *Rationale:* §4 item 9. Peer-shared APKs have no trust anchor without a published fingerprint, and source hosting can be geoblocked (§3).
   - *Touches:* spec §8 (release gates), §10 ("Fake Wren apps" row); app and release sub-plans.
5. **Share a signed transaction; broadcast a pasted transaction.**
   - *Proposal:* add two actions.
     - *Share signed transaction:* export the hex as text, QR or file.
     - *Broadcast a pasted transaction:* accepted only over a verified XBT connection, and only when every input carries a valid `0x21` signature checked against prevouts fetched from that server.
   - *Rationale:* a signed 0x21 transaction holds no secret and is not valid on BTC, so any channel can carry it (§4 item 8). The broadcast gate follows RedWallet's per-input check.
   - *Touches:* spec §3.1 (API), §5 (Activity/Settings); plan Tasks 14/20; v1.x candidate.
6. **Publish deployment evidence for the default server.**
   - *Proposal:* publish the Knots and Fulcrum versions and commits, the TLS SPKI SHA-256 pin, and the onion address, updated on each upgrade. Hosting and operations details stay private, as spec §7 requires.
   - *Rationale:* users and reviewers can then check the server pin that Wren shows. The Paperclip ASP publishes a comparable record ([deployed-version.md](https://github.com/connorslab/paperclip-asp/blob/main/docs/deployed-version.md)).
   - *Touches:* spec §7; `infra/` runbook.
7. **Labels and coin control as a later privacy step.**
   - *Proposal:* plan per-coin labels and manual coin selection for after v1.
   - *Rationale:* these are client-only and need no third-party operator. Coin control is currently listed only as "Out of scope for v1" (§5), with no future entry.
   - *Touches:* spec §5 (Out of scope); `docs/future-ideas.md`.
8. **Ark, coinjoin and ecash: not proposed for v1.** Recording them as watch items only, for these technical reasons.
   - *Ark:* a seed alone does not recover funds, because full state must be backed up. VTXOs expire about 4,300 blocks after boarding and need the app online to refresh (issue #4: board at 976,193, expiry 980,512). Recovery reserves are large relative to small payments. There is a single beta operator. Exit timelocks have not been analyzed against reorg depth on this chain.
   - *Coinjoin (Kilojoin):* rounds are 2–20 participants. The shared pool key lets any member disrupt a round, and one person can hold several seats. The relay sees IP and timing metadata. Inputs are P2WPKH only. The Rust port has no validated funded cross-client round and no audit. A round needs the app in the foreground.
   - *Ecash:* the mint is custodial, and no XBT mint is known. Offline tokens are not final until swapped. Lightning in and out of a mint inherits the invoice chain-identity issue in §4 item 1.

**Considered and already covered.**
- **A test that Wren signatures fail under BTC rules.** Covered by the spec §8 replay proof (Core regtest must reject every pre-split spend), the Task 7 raw legacy/BIP143 digests, and Review Focus #6 (`DangerousSignature`).
- **The Knots 6,480 relay rule.** Covered by Area B.
- **The bare-address "expects XBT" warning.** Covered by spec §3.4.
- **Save-before-broadcast with an `Unknown` outcome.** Covered by Area B.
- **SPKI pinning and Tor without automatic fallback.** Covered by Area C.

---

## 6. Sources (all accessed 2026-10-09 unless dated otherwise)

- RedWallet: [repo](https://github.com/romanmodin/RedWallet) · [README](https://github.com/romanmodin/RedWallet/blob/main/README.md) · [RELEASE.md](https://github.com/romanmodin/RedWallet/blob/main/RELEASE.md) · [ANDROID_INSTALL.md](https://github.com/romanmodin/RedWallet/blob/main/ANDROID_INSTALL.md) · [SECURITY_FOLLOWUP.md](https://github.com/romanmodin/RedWallet/blob/main/SECURITY_FOLLOWUP.md) · [TAPROOT.md](https://github.com/romanmodin/RedWallet/blob/main/TAPROOT.md) · [PROJECT_SCOPE.md](https://github.com/romanmodin/RedWallet/blob/main/PROJECT_SCOPE.md) · [XBT_VALIDATION.md](https://github.com/romanmodin/RedWallet/blob/main/XBT_VALIDATION.md) · [XBT_PR_REVIEW.md](https://github.com/romanmodin/RedWallet/blob/main/XBT_PR_REVIEW.md) · [electrum-checkpoint.ts](https://github.com/romanmodin/RedWallet/blob/main/class/xbt/electrum-checkpoint.ts) · [broadcast-validation.ts](https://github.com/romanmodin/RedWallet/blob/main/class/xbt/broadcast-validation.ts) · [coinbase-maturity.ts](https://github.com/romanmodin/RedWallet/blob/main/class/xbt/coinbase-maturity.ts) · [fee-policy.ts](https://github.com/romanmodin/RedWallet/blob/main/class/xbt/fee-policy.ts) · [payment-warning.ts](https://github.com/romanmodin/RedWallet/blob/main/class/xbt/payment-warning.ts) · [replay-notice.ts](https://github.com/romanmodin/RedWallet/blob/main/class/xbt/replay-notice.ts) · [recovery-discovery.ts](https://github.com/romanmodin/RedWallet/blob/main/class/xbt/recovery-discovery.ts) · [PR #1](https://github.com/romanmodin/RedWallet/pull/1) · [PR #26](https://github.com/romanmodin/RedWallet/pull/26) · issues [#37](https://github.com/romanmodin/RedWallet/issues/37), [#38](https://github.com/romanmodin/RedWallet/issues/38)
- Paperclip: [paperclip-wallet-app](https://github.com/connorslab/paperclip-wallet-app) · [mobile/README.md](https://github.com/connorslab/paperclip-wallet-app/blob/feature/ios-wallet/mobile/README.md) · [electrum-ark.md](https://github.com/connorslab/paperclip-wallet-app/blob/feature/ios-wallet/mobile/electrum-ark.md) · [LightningNetwork.swift](https://github.com/connorslab/paperclip-wallet-app/blob/feature/ios-wallet/mobile/Sources/PaperclipMobile/LightningNetwork.swift) · [issue #4](https://github.com/connorslab/paperclip-wallet-app/issues/4) · [bark-xbt](https://github.com/connorslab/bark-xbt) · [paperclip-asp](https://github.com/connorslab/paperclip-asp) · [deployed-version.md](https://github.com/connorslab/paperclip-asp/blob/main/docs/deployed-version.md) · [connorslab/lightning](https://github.com/connorslab/lightning) · [connorslab repos](https://github.com/connorslab?tab=repositories) · [lightning.paperclip-xbt.xyz](https://lightning.paperclip-xbt.xyz/) · [ark.paperclippool.xyz](https://ark.paperclippool.xyz/) · [paperclip-xbt.xyz](https://paperclip-xbt.xyz/)
- Lightning: [privkeyio/lightning releases](https://github.com/privkeyio/lightning/releases) · [BOLT 11](https://github.com/lightning/bolts/blob/master/11-payment-encoding.md)
- Kilojoin: [Kilombino/kilojoin PROTOCOL.md](https://github.com/Kilombino/kilojoin/blob/main/docs/PROTOCOL.md) · [Kilombino/kilowallet](https://github.com/Kilombino/kilowallet) · [connorslab/paperclip-kilojoin-rs](https://github.com/connorslab/paperclip-kilojoin-rs)
- Bitchat: [bitchat iOS](https://github.com/permissionlesstech/bitchat) · [WHITEPAPER.md](https://github.com/permissionlesstech/bitchat/blob/main/WHITEPAPER.md) · [bitchat-android](https://github.com/permissionlesstech/bitchat-android) · [v2.0.1 release](https://github.com/permissionlesstech/bitchat-android/releases/latest) · [releases](https://github.com/permissionlesstech/bitchat-android/releases) · [btcpp Insider (2025-07-18)](https://insider.btcpp.dev/p/knock-knock-its-cashu-bitch) · [Forbes (2025-07-31)](https://www.forbes.com/sites/davidbirnbaum/2025/07/31/can-bitchat-build-a-free-internet-with-bitcoin/) · [Sonar PR #614](https://github.com/hedwig-corp/bitchat-to-sonar/pull/614) · [Rest of World (2026-07)](https://restofworld.org/2026/india-github-geoblock-jack-dorsey-bitchat/) · [Inc42 (2026-10-04)](https://inc42.com/buzz/jack-dorseys-bitchat-goes-offline-in-india-apple-cites-meity-order/)
- Ecosystem: [lazarus-xbt.xyz/ecosystem](https://lazarus-xbt.xyz/ecosystem/)
- Wren: [design spec](../specs/2026-09-24-wren-wallet-design.md) · [Gate A amendments](../plans/2026-10-08-wren-engine-gate-a-amendments.md) · [fork consensus facts](./2026-09-28-fork-consensus-facts.md) · [future ideas](../../future-ideas.md)
