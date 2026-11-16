# XBT wallet ecosystem — update (2026-11-16)

These notes update [the 2026-10-09 ecosystem notes](./2026-10-09-xbt-wallet-ecosystem.md) and [the 2026-10-09 Lightning notes](./2026-10-09-xbt-lightning.md). They cover what changed in five weeks, plus two XBT wallets the earlier notes did not cover. They are factual notes, not an endorsement or a critique. Every claim links to its source.

**Method.** Repositories, release notes and project web pages were read through GitHub and the web on 2026-11-16. A US App Store search was run the same day. No third-party code was built or run. "Not determined" means the sources read did not settle the point.

---

## 1. Changes since 2026-10-09

### 1.1 RedWallet (`romanmodin/RedWallet`)

- **Version 8.0.1.** iPhone via TestFlight and, since 2026-11-08, a sideload IPA. Android APK on GitHub Releases. No App Store or Google Play listing. — [releases](https://github.com/romanmodin/RedWallet/releases), [README](https://github.com/romanmodin/RedWallet/blob/main/README.md)
- **Embedded Lightning.** The app now runs LND inside the phone, syncing through Neutrino. The default peer is `ln.kilombino.com:11010`, and fee data comes from `nodex.kilombino.com`. Peer selection and Neutrino setup are "adapted from Kilombino/kilowallet-ios". Release notes say the build includes "XBT Lightning wallet and Tor support". — [MainnetConfiguration.swift](https://github.com/romanmodin/RedWallet/blob/main/ios/XBTLightning/Sources/XBTLightning/MainnetConfiguration.swift), [8.0.1 sideload release](https://github.com/romanmodin/RedWallet/releases)
- **Default Electrum server.** Still none. Issue #37 ("Add public Fulcrum or Electra Server") is open. — [issue #37](https://github.com/romanmodin/RedWallet/issues/37)
- **On-chain chain verification.** No change found to the single-checkpoint design described on 2026-10-09.

### 1.2 Paperclip (`connorslab/paperclip-wallet-app`)

- **Daemon 0.10.0-beta.1 (2026-11-11)** for StartOS and Umbrel, on Bark 0.8.1. "A seed alone does not preserve Ark recovery history." — [release](https://github.com/connorslab/paperclip-wallet-app/releases)
- **Mobile.** iOS is still a development branch (`feature/ios-wallet`, last commit 2026-11-12). The README says the CI artifact "is not a signed device IPA or a TestFlight release". An Android branch (`feature/android-wallet`) is active. BIP39 passphrases are still not supported. — [mobile/README.md](https://github.com/connorslab/paperclip-wallet-app/blob/feature/ios-wallet/mobile/README.md), [branches](https://github.com/connorslab/paperclip-wallet-app/branches)

---

## 2. KiloWallet (`Kilombino/kilowallet`, `Kilombino/kilowallet-ios`)

**Status, platforms, license.** Apache-2.0. Android 1.6.0 (2026-11-13) ships as an APK with a published signing-certificate SHA-256 and a documented reproducible build. iOS 1.0.0-rc6 (2026-11-14) ships as an unsigned IPA. The iOS README names an official App Store build, bundle ID `com.kilombino.kilowallet`, "published by romanmodin on Kilombino's behalf". It did not appear in a US App Store search on 2026-11-16. — [Android repo](https://github.com/Kilombino/kilowallet), [Android 1.6.0](https://github.com/Kilombino/kilowallet/releases), [iOS repo](https://github.com/Kilombino/kilowallet-ios), [iOS rc6](https://github.com/Kilombino/kilowallet-ios/releases)

**Android features (README).**
- Watch-only xpubs on the BLAKE2b and SHA-256 chains side by side, or a hot wallet whose seed comes from dice or the secure RNG.
- Optional BIP39 passphrase, with the master fingerprint shown.
- WIF private-key sweep for P2PKH, P2SH-P2WPKH, P2WPKH and P2TR, signed with `0x21`.
- **Pre-fork LND seed rescue.** Takes 24 aezeed words, sweeps the node's on-chain coins on BLAKE2b with `0x21`, and replays cooperative channel closes made on SHA-256 after the fork. With a `channel.backup`, it lists each channel's state on both chains and sweeps force-close outputs after their CSV delay.
- Embedded Tor (Arti). Default servers on both chains: `fulcrum.kilombino.com:17717` (BLAKE2b, Fulcrum 2.1.2) and a SHA-256 Frigate server.
- Kilojoin pools, and JoinBlaket (JoinMarket on the BLAKE2b chain; taker payments, maker jars).
- No PSBT or hardware-signer workflow is documented.

**iOS.** The `main` README describes 0.4: on-chain only, BIP39 passphrase, watch-only, SPV checks per incoming payment against 164-byte headers, and default Fulcrum with the certificate pinned on first use. It lists as not included "a full header chain (SPV is per payment)", PSBT and Lightning. The rc6 release bundles LND (Lightning Fork v0.21.4-beta-blake2b.21.2, "neutrino NODE_BLAKE2B"), the Paperclip Bark Ark wallet and Arti 2.7.0.

---

## 3. XBT Wallet (`BitcoinMechanic/xbt-wallet`)

**Status, platform, license.** Android only. Repo created 2026-11-09. First release v0.1.0-alpha.1 (2026-11-14) is a debug-signed mainnet test build: "Use small test amounts." The README states: "There is no new blanket license grant for the application." — [repo](https://github.com/BitcoinMechanic/xbt-wallet), [README.txt](https://github.com/BitcoinMechanic/xbt-wallet/blob/main/README.txt), [release](https://github.com/BitcoinMechanic/xbt-wallet/releases)

**Modes.**
- *Easy mode:* embedded LND (Lightning Fork at a pinned commit, plus local patches in `patches/`) syncing through Neutrino. The seed is LND aezeed, not BIP39.
- *Remote Control:* the phone drives the user's own XBT Core Lightning node over REST, through a VPN, Start Tunnel, or an onion address over embedded Tor.

**BTC↔XBT Lightning swaps.** The wallet can receive a BTC Lightning payment as XBT, or pay a BTC Lightning invoice with XBT. It does this through a self-hosted "swap controller" on StartOS, which needs BTC and XBT full nodes, swap-enabled Core Lightning on both chains, and funded channels on both. Per SCOPE.md, the mechanism links an incoming hold invoice and the destination invoice through one payment hash, so the swap is atomic. There is no public controller. — [SCOPE.md](https://github.com/BitcoinMechanic/xbt-wallet/blob/main/SCOPE.md), [swap-controller-startos](https://github.com/BitcoinMechanic/swap-controller-startos)

**Stated limitations.** "Full recovery import is unfinished; seed words alone do not restore active Lightning channels." Recovery and deadline protection "may force-close channels".

---

## 4. Lightning updates

1. **Releases.**
   - Lightning Fork (LND) v0.21.8-beta-blake2b.24, 2026-11-12. — [releases](https://github.com/paulscode/lightning-fork/releases)
   - privkeyio Core Lightning v26.06.10-blake2b.1, 2026-11-06. — [releases](https://github.com/privkeyio/lightning/releases)
   - Both are packaged for StartOS. — [lightning-fork-startos](https://github.com/paulscode/lightning-fork-startos), [cln-startos](https://github.com/privkeyio/cln-startos)
2. **Lightning note §1.6 item 2 (LND-fork invoice bit) is still not settled from documents.** The identity doc says bit 512 is in the BOLT 11 `9` field, "implemented on both sides since 2026-09-28". The LND-fork README still says an invoice "carries nothing that says which of the two chains it belongs to". Lightning Fork Swap's README says its backend "refuses invoices without" bit 512. — [identity doc §5](https://github.com/paulscode/lightning-fork/blob/master/docs/blake2b-chain-identity.md), [LND-fork README](https://github.com/paulscode/lightning-fork), [Lightning Fork Swap](https://github.com/paulscode/lightning-fork-swap)
3. **The identity spec is still provisional.** It is "open to change until more than two implementations have mainnet channels".
4. **Embedded LND with Neutrino is now shipped by three wallets:** RedWallet, KiloWallet iOS rc6 and XBT Wallet. This architecture is not a row in the Lightning note's §2 options table.
5. **privkeyio `rust-lightning` (LDK).** The `blake2b` branch still has no tags. The last push was 2026-11-07. — [repo](https://github.com/privkeyio/rust-lightning)
6. **Cross-chain bridges now exist in shipping clients** (§3). This bears on Lightning note §4 rule 4 ("No implicit cross-chain conversion").

---

## 5. Facts that affect Wren's design

1. **Pre-fork Lightning users hold aezeed seeds**, not BIP39 seeds. One wallet now rescues their on-chain coins and cooperative-close outputs (§2). Wren's Claim takes an xpub from a hardware signer, so it does not cover aezeed seeds or channel-close outputs.
2. **WIF keys and paper wallets** are another pre-split holding that Claim does not cover.
3. **Neutrino on this chain needs peers that serve BLAKE2b compact filters** (KiloWallet's rc6 notes name a `NODE_BLAKE2B` service). The wallets read default to one operator's peer.
4. **Several shared services depend on single operators.** One operator provides the default Electrum server, Lightning peer, fee feed and Kilojoin relay used by two wallets (§1.1, §2). Paperclip runs the LSP and Ark server. Lightning Fork Swap has one operator.

---

## 6. Proposed additions for review (not yet adopted)

These are offered for discussion only.

1. **Settle §1.6 item 2 empirically.** Before any Lightning phase, decode a fresh mainnet invoice from each implementation and record whether the `9` field carries bit 512. Keep the results as test fixtures.
2. **Add "embedded LND + Neutrino" to the Lightning note's §2 options table.** Record its recovery limitation (seed words alone do not restore channels), its binary size and background limits on iOS, and its dependence on BLAKE2b filter peers.
3. **Add "swap client" as an option.** Wren would stay an on-chain wallet and pay or receive Lightning invoices through submarine and reverse swaps with a third-party service, running no node. Lightning note §4 rules 1, 3 and 4 would apply. The service's chain identity would be checked against Wren's verified header chain.
4. **Claim scope statement (spec §4.1).** State explicitly that aezeed (LND) seeds, WIF keys and channel-close outputs are out of scope for v1.

---

## 7. Sources (all accessed 2026-11-16)

- RedWallet: [repo](https://github.com/romanmodin/RedWallet) · [releases](https://github.com/romanmodin/RedWallet/releases) · [MainnetConfiguration.swift](https://github.com/romanmodin/RedWallet/blob/main/ios/XBTLightning/Sources/XBTLightning/MainnetConfiguration.swift) · [issue #37](https://github.com/romanmodin/RedWallet/issues/37)
- Paperclip: [releases](https://github.com/connorslab/paperclip-wallet-app/releases) · [mobile/README.md](https://github.com/connorslab/paperclip-wallet-app/blob/feature/ios-wallet/mobile/README.md)
- KiloWallet: [Android](https://github.com/Kilombino/kilowallet) · [Android releases](https://github.com/Kilombino/kilowallet/releases) · [iOS](https://github.com/Kilombino/kilowallet-ios) · [iOS releases](https://github.com/Kilombino/kilowallet-ios/releases)
- XBT Wallet: [repo](https://github.com/BitcoinMechanic/xbt-wallet) · [README.txt](https://github.com/BitcoinMechanic/xbt-wallet/blob/main/README.txt) · [SCOPE.md](https://github.com/BitcoinMechanic/xbt-wallet/blob/main/SCOPE.md) · [swap-controller-startos](https://github.com/BitcoinMechanic/swap-controller-startos)
- Lightning: [lightning-fork](https://github.com/paulscode/lightning-fork) · [identity doc](https://github.com/paulscode/lightning-fork/blob/master/docs/blake2b-chain-identity.md) · [lightning-fork-swap](https://github.com/paulscode/lightning-fork-swap) · [lightningfork.com](https://lightningfork.com) · [privkeyio/lightning](https://github.com/privkeyio/lightning/releases) · [privkeyio/rust-lightning](https://github.com/privkeyio/rust-lightning)
- Wren: [ecosystem notes 2026-10-09](./2026-10-09-xbt-wallet-ecosystem.md) · [Lightning notes 2026-10-09](./2026-10-09-xbt-lightning.md) · [design spec](../specs/2026-09-24-wren-wallet-design.md)
