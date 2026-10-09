# XBT Lightning — state and options for Wren (2026-10-09)

This note records what XBT (BLAKE2b-chain) Lightning software and services exist as of 2026-10-09, and how a non-custodial mobile wallet could use them.
It is meant for reviewers of Wren's design. The companion note on other XBT wallets is [2026-10-09-xbt-wallet-ecosystem.md](./2026-10-09-xbt-wallet-ecosystem.md), and this note does not repeat it.

**Status:** Lightning is out of scope for Wren v1; this note proposes a phased path for review.

**Method.** Repositories, release notes and project pages were read on 2026-10-09. No third-party code was built or run. Effort figures are estimates, not measurements.

**Chain context.**
- The chains split at block 961,632 (2026-08-08).
- BLAKE2b proof of work activated at block 961,640 (2026-08-30), with 164-byte headers.
- The activation block id is `0000000000000050c1e5f69672f459293be14f46e5a494e7a8c8541396f18eeb`.

Sources: [awesome-bitcoin-blake](https://github.com/bitcoin-blake/awesome-bitcoin-blake), [lightning-fork-swap README](https://github.com/paulscode/lightning-fork-swap)

---

## 1. State of XBT Lightning (2026-10-09)

### 1.1 Implementations

| Component | Status | Version / date | Notes | Source |
|---|---|---|---|---|
| privkeyio Core Lightning fork (`blake2b-unified`) | Shipped, mainnet | `.5` 2026-09-28, `.6` 2026-10-02, `.7` (`v26.06.9-blake2b.7`) 2026-10-09 | `.5` to `.7` peer with each other; `.4` and older cannot. `.6` stops if its backend serves a post-activation block without a BLAKE2b header. `.7` rebases on the upstream v26.06.9 security fixes and keeps unified HTLC hash types after a splice locks. Builds are reproducible and signed with key `A47D99B6DB0D715D40C59A2023AE8A8EA7E24E38`. The StartOS package tracks `.7`. | [releases](https://github.com/privkeyio/lightning/releases), [cln-startos](https://github.com/privkeyio/cln-startos/releases) |
| paulscode LND fork "Lightning Fork" | Shipped, mainnet | `.18` (`v0.21.4-beta-blake2b.18`) 2026-10-09 | Refuses opens without unified sigs. Taproot channels can be neither opened nor accepted. The fundee refuses coinbase funding. PSBT and remote-signer signing is bound to the unified hash. Adds BOLT 3 vectors for unified sigs. Packaged for StartOS and Umbrel. | [release .18](https://github.com/paulscode/lightning-fork/releases/tag/v0.21.4-beta-blake2b.18) |
| privkeyio rust-lightning, branch `blake2b` | Branch only, no tags | Commits 2026-10-06/07 | Built on the BLAKE2b rust-bitcoin. Adds unified signing, sets and requires 512 in BOLT 11/12, the gossip floor, coinbase refusal, and no splice without 514. No ldk-node fork or mobile bindings exist. | [commits](https://github.com/privkeyio/rust-lightning/commits/blake2b), [privkeyio repos](https://github.com/orgs/privkeyio/repositories) |
| Lampo (LDK node), branch `blake2b` | Regtest only | `0.1.0-blake2b.1` in Polar `v4.0.0-blake2b.2` (2026-10-09) | Lampo nodes "pay and are paid by both [CLN and LND], and close cooperatively". Force-close interop with LDK is not stated. Polar removed Eclair "since [it does] not follow BLAKE2b". | [lampo.rs](https://github.com/privkeyio/lampo.rs/commits/blake2b), [Polar release](https://github.com/privkeyio/polar/releases/tag/v4.0.0-blake2b.2) |

No Eclair, lightning-kmp, Breez SDK or Greenlight port was found on 2026-10-09. All the software above describes itself as unofficial, unaudited and experimental.

### 1.2 Chain-separation mechanism

XBT Lightning keeps BTC's identifiers:
- `chain_hash` is the BTC genesis hash;
- the BOLT 11 prefix stays `lnbc`;
- `offer_chains` is unchanged;
- `getinfo` reports `network = "bitcoin"`;
- invoice fallback addresses use the shared formats.

A distinct `chain_hash` was tried and withdrawn on 2026-09-17. A distinct invoice prefix was tried and withdrawn on 2026-09-19. Source: [blake2b-chain-identity.md §1, §5, §8](https://github.com/paulscode/lightning-fork/blob/blake2b/docs/blake2b-chain-identity.md)

Separation relies on five things instead:

- **Bit 512 (`option_blake2b`), even only,** in `init` and `node_announcement`. A stock node does not know the bit and must disconnect. Stock lnd reports `feature vector contains unknown required features: [512]`.
- **Bit 514 (`option_unified_sigs`) in `channel_type`.** Bit 515 (odd) goes in `init` and `node_announcement`, and bits 516 to 519 are reserved. The interop-tested channel type is `[12, 22, 514]` (static_remotekey, anchors, unified).
- **A gossip floor.** Nodes ignore any `channel_announcement` with an SCID height below 961,640 (testnet4: 150,308).
- **Channel refusals.** Nodes refuse:
  - Taproot channel types, because MuSig2 partial signatures cannot carry the hash type;
  - coinbase-funded channels;
  - splices on non-unified channels.
- **Bit 512 in payment requests.** It is compulsory in the BOLT 11 `9` field and in the BOLT 12 `offer_features`, `invreq_features` and `invoice_features`. CLN `.5` says "one without it is not paid". Stock CLN refuses such an invoice when decoding it, and stock lnd refuses it during pathfinding.

The renumbering from bits 68/70 to 512/514 on 2026-09-28 "was a flag day ... made by both implementations at the same hour". Sources: [identity doc §2–§7](https://github.com/paulscode/lightning-fork/blob/blake2b/docs/blake2b-chain-identity.md), [CLN .5](https://github.com/privkeyio/lightning/releases/tag/v26.06.8-blake2b.5), [CLN .6](https://github.com/privkeyio/lightning/releases/tag/v26.06.8-blake2b.6)

The separation only works in one direction. BTC software refuses an XBT invoice, but a BTC invoice carries nothing that marks it as BTC. Both implementations also "ship a path that skips" the bit check on decode-only paths:
- CLN guards the check with `if (our_features)`, and a caller that passes NULL skips it.
- lnd's `errorOnUnknownFeature` is off by default.

Source: [doc/blake2b-upgrade.md](https://github.com/privkeyio/lightning/blob/blake2b-unified/doc/blake2b-upgrade.md)

### 1.3 Signing

`SIGHASH_UNIFIED` is bit `0x20`. The digest is `TaggedHash("UnifiedSighash", message)`, computed over a BIP 341-shaped message that commits to every spent output's value and scriptPubKey.

Hash types on channels with 514:

| Signature | Hash type |
|---|---|
| Commitment, cooperative close, non-anchor second-level HTLC | `0x21` (ALL\|UNIFIED) |
| Anchor channels: second-level HTLC half signed for the peer | `0xa3` (SINGLE\|ANYONECANPAY\|UNIFIED) |
| Anchor channels: broadcaster's own HTLC half | `0x21`. An implementation that "signs its own half `0xa3` produces a transaction that is valid and malleable by a third party". |
| Unilateral node signatures (funding inputs, sweeps, anchor spends) | `0x21` |
| Justice transaction handed to a watchtower | Legacy hash type, by design. It spends an output of a commitment that was already signed with the opt-in. |

Related points:
- **Downgrades break unified channels.** The older build "signs `0x01` instead. The peer then rejects every signature ... Nor can the older build force close it."
- **Interactive-tx `tx_signatures`.** The sender SHOULD use `0x21`, and the receiver accepts either ([bolts PR #4](https://github.com/lightning-blake2b/bolts/pull/4)).
- **Pre-split funding.** A channel funded from pre-split coins has a funding transaction "valid under both rule sets, which reopens the exposure" ([CLN .4](https://github.com/privkeyio/lightning/releases/tag/v26.06.7-blake2b.4)).

Sources: [identity doc §4, §7](https://github.com/paulscode/lightning-fork/blob/blake2b/docs/blake2b-chain-identity.md), [doc/blake2b-upgrade.md](https://github.com/privkeyio/lightning/blob/blake2b-unified/doc/blake2b-upgrade.md)

### 1.4 Live services

| Service | State (2026-10-09) | Notes | Source |
|---|---|---|---|
| Paperclip CLN node (`02bfcaa8…3389`) | Live, labeled experimental | Sells channel packages (Large = 25,000,000 sats). The page lists CLN `.5`. It advertises no LSPS, JIT channels, zero-conf or fees. It states: "A feature declaration does not prove a peer's chain or honesty." | [lightning.paperclip-xbt.xyz](https://lightning.paperclip-xbt.xyz/) |
| CLN LSPS2 plugin | Code is in `.7`; no operator advertises it | LSPS2 only (no LSPS1, no LSPS5). The JIT `fundchannel` call was patched from `[12, 46, 50]` to add the unified bit, but the plugin still tests the provisional 70/71 pair. `fundchannel_start` adds 514 separately. | [compare v26.06.9...blake2b.7](https://github.com/ElementsProject/lightning/compare/v26.06.9...privkeyio:lightning:v26.06.9-blake2b.7) |
| Lightning Fork Swap (lightningfork.com) | Live; repo created 2026-10-08 | Boltz-based submarine and reverse swaps between XBT on-chain and XBT Lightning. Refuses invoices without 512, and refuses to run unless 961,640 is the BLAKE2b activation block. Fees and limits are not published. | [lightning-fork-swap](https://github.com/paulscode/lightning-fork-swap) |
| LND-fork cross-chain bridge | Shipped in `.17`/`.18`; each operator runs their own | Pays BTC Lightning invoices from XBT Lightning, and the reverse, using matching hold invoices. The rate comes from an exchange feed, and the default fee is 1.5% per direction. | [bridge.md](https://github.com/paulscode/lightning-fork/blob/blake2b/docs/bridge.md) |
| XBTPay (BTCPay plugin) | Beta, created 2026-10-02 | CLN checkout verified; LND receive untested; no BOLT 12 merchant checkout. | [paperclip-btcpay-xbt](https://github.com/connorslab/paperclip-btcpay-xbt) |
| Lightning Addresses | One known, created 2026-10-09 | `xbt-zaps` adds `"chain": "bitcoin-blake2b"` to the LNURL-pay response and chain tags to NIP-57 events. | [xbt-zaps](https://github.com/Kilombino/xbt-zaps) |

The existing mobile clients are remote controls or Ark-based: Lightning Fork Android (v0.2.1, 2026-10-05) and Paperclip Wallet. See the [ecosystem note §2](./2026-10-09-xbt-wallet-ecosystem.md). No mobile app runs an XBT Lightning node on the device. Source: [lightning-fork-android](https://github.com/paulscode/lightning-fork-android)

### 1.5 Block-rate observation

The BLAKE2b chain "has run between 1.3x and 19x faster than its ten minute target, and a block on it costs about 1/325th as much to produce, so neither HTLC timing nor reorg depth can be reasoned about in blocks." Source: [lightning-fork-bridge README](https://github.com/paulscode/lightning-fork-bridge) (read 2026-10-09)

For a phone, this means the same `to_self_delay` and `cltv_expiry_delta` block counts last less wall-clock time than on BTC.

### 1.6 Open issues relevant to a wallet

1. **Stored gossip (CLN #20, open since 2026-09-28).** The gossip floor applies only to announcements as they arrive. On one upgraded mainnet node, "37,083 of the 37,085 channels in `listchannels` were funded before block 961,640". Route finding can therefore pick non-upgraded nodes, though "no funds are at risk". This is also the only data point found on network size. Source: [privkeyio/lightning #20](https://github.com/privkeyio/lightning/issues/20)
2. **LND-fork invoice bit, not yet confirmed.** The LND-fork README says an invoice "carries nothing that says which of the two chains it belongs to". The same repo's identity doc says the invoice bit is "implemented on both sides since 2026-09-28". The README likely predates the flag day, but this has not been checked against a `.18` invoice. Sources: [README](https://github.com/paulscode/lightning-fork), [identity doc §5](https://github.com/paulscode/lightning-fork/blob/blake2b/docs/blake2b-chain-identity.md)
3. **Stale bit-70 check in the LSPS2 plugin.** It is likely harmless because `fundchannel_start` adds 514, but this is unverified. Source: [compare diff](https://github.com/ElementsProject/lightning/compare/v26.06.9...privkeyio:lightning:v26.06.9-blake2b.7)
4. **Unified-sig test vectors.** These are still open as [bolts #8](https://github.com/lightning-blake2b/bolts/issues/8). CLN PR #27 and LND-fork PR #2 (both 2026-10-08) add vectors for their own implementations ([CLN #27](https://github.com/privkeyio/lightning/pull/27), [release .18](https://github.com/paulscode/lightning-fork/releases/tag/v0.21.4-beta-blake2b.18)). It was not verified whether the LDK branch handles the `0xa3`/`0x21` split and the watchtower exception the same way.
5. **Spec venue.** The spec lives in `lightning-blake2b/bolts`, forked 2026-09-17.
   - #1 and #3 were merged 2026-09-29.
   - #4 to #6 were merged 2026-10-07.
   - Issues #8 to #10 and draft #7 are open.

   Values are "open to change until more than two implementations have mainnet channels; changes after that would strand channels." Sources: [bolts pulls](https://github.com/lightning-blake2b/bolts/pulls?q=is%3Apr), [identity doc, Status](https://github.com/paulscode/lightning-fork/blob/blake2b/docs/blake2b-chain-identity.md)

---

## 2. Architecture options for a non-custodial mobile wallet

The comparison assumes Wren's design constraints:
- Wren runs no Lightning infrastructure.
- Its engine is Rust (patched BDK, rust-bitcoin 0.32, embedded Arti) under React Native ([spec §3](../specs/2026-09-24-wren-wallet-design.md)).

Effort figures are **estimates** of the Wren-side work only.

| Option | Custody | Third-party operator needed | XBT patches needed | Offline receive | Effort (estimate) | Status / notes (2026-10-09) |
|---|---|---|---|---|---|---|
| **Embedded LDK + third-party LSP** | On the phone. A zero-conf JIT channel trusts the LSP until it confirms. | An LSPS2 LSP to receive. None to send only (ordinary channel to a public node). | A rust-lightning fork: sighash paths in `chan_utils`, `channel`, `package`, `channelmonitor` and `interactivetxs`; features 512/514/515; the invoice bit; the gossip floor; coinbase refusal. Also 164-byte headers through a patched rust-bitcoin, and an LSPS2 client that accepts 514. privkeyio's branch covers most of the protocol work. | No. The app must be open. XBT has no LSPS5 and no LDK-based LSP (needed for async payments). | 3–6 months to beta, then about 2–5 engineer-days a month for LDK security rebases | The branch is untagged and days old. No operator advertises LSPS2. LDK 0.2.7 (2026-10-01) fixed two funds-theft issues, so tracking upstream is required. Source: [CHANGELOG](https://github.com/lightningdevkit/rust-lightning/blob/main/CHANGELOG.md) |
| **Embedded LND fork** | On the phone | Same LSP need, plus BIP 157/158 peers for a light backend | The protocol is already patched in Go. Needs gomobile and a BLAKE2b-aware neutrino (existence unverified). | No | Not estimated | Adds a second runtime and key store beside the Rust engine. |
| **lightning-kmp (Phoenix-style)** | On the phone | An XBT eclair LSP (none exists) | Forks of lightning-kmp, bitcoin-kmp and eclair | Only with an ACINQ-style server | Not estimated | No XBT port found. Adds a Kotlin stack beside Rust. |
| **Breez SDK** | Varies by variant | Spark operators, Greenlight, or Liquid/Boltz, plus an API key. None exist for XBT. | Not feasible client-side | n/a | n/a | No backend has an XBT deployment. |
| **Remote own node (CLNRest / LND REST over Tor)** | User's node | None | None in the node (CLN `.6`+ or LND `.18`+). Wren adds local invoice decoding and a chain check. | Yes, because the node is always on | 2–4 weeks | The pattern already exists: the Paperclip iOS branch (rune or macaroon; "never re-calls `pay`") and Lightning Fork Android. Source: [mobile README](https://github.com/connorslab/paperclip-wallet-app/blob/feature/ios-wallet/mobile/README.md) |
| **NWC (NIP-47)** | User's wallet service | A Nostr relay the user chooses | None. `get_info` returns `block_hash` and `block_height` for a chain check (`network` only says "mainnet"). | Yes, if the service is always on | 2–3 weeks on top of the REST work | NIP-44 is required, and the connection secret is revocable. An XBT CLN bridge exists but pins `.4`, which is stale. Sources: [NIP-47](https://github.com/nostr-protocol/nips/blob/master/47.md), [control-plane](https://github.com/btcblake2b/control-plane) |
| **Ark (bark-xbt)** | User, with ASP liveness and VTXO-expiry caveats | One ASP (Paperclip) | `bark-xbt` exists | BOLT 12 receive while the wallet service is online | 4–8 weeks | 250,000-sat Lightning cap. About 4,120 sats overhead on a 10,000-sat receive. The seed alone is not a full backup. Issue #4 is open. See [ecosystem note §2.3, §5 item 8](./2026-10-09-xbt-wallet-ecosystem.md). |
| **Submarine swaps** | User, through an HTLC with a refund path | A swap operator (Lightning Fork Swap) | Claim and refund signed `0x21`; header verification of the lockup | n/a | 3–5 weeks per provider | The service is days old. Fees and limits are unpublished. |

One point affects the order of work: **sending needs no LSP standard, but receiving does.** A wallet with an embedded node can pay after opening an ordinary channel to any public XBT node. Receiving needs inbound capacity. Today that means buying a channel manually or running your own node.

---

## 3. Proposed phased path (for review, not adopted)

In every phase, Wren would **not run** any LSP, swap service, Nostr relay, watchtower service or Lightning Address server. It would not hold invoices open on a node of its own.

### Phase 0: spec only (no code)

- Add the wrong-chain rules in §4 to the Wren spec as requirements for any future Lightning path.
- **v1 behavior for Lightning strings.** A user may scan or paste an `lnbc…` or `lno…` string, or a `lightning:` URI. Wren decodes just enough to recognize it as a Lightning payment request and explains that Lightning is not supported. It never extracts an `f` fallback, and never treats the string as an on-chain target.
- **Nest payment request.** Design it as a seller-generated object. Today it carries an `xbt:` target ([spec §3.4](../specs/2026-09-24-wren-wallet-design.md)). Later it can carry an invoice without changing the buyer flow.

### Phase 1: connect your own node

- **Work:**
  - CLNRest (rune) and LND REST (macaroon) clients over Arti, plus an NWC client (NIP-44);
  - local BOLT 11/12 decoding;
  - the chain check against Wren's own verified header chain;
  - least-privilege credentials, kept in the platform key store;
  - a payment journal that records each attempt before paying and never retries when the result is uncertain.
- **Effort estimate:** 4–8 weeks.
- **Gates:**
  - user-installable CLN `.6`+ or LND `.18`+ packages (both exist on StartOS as of 2026-10-09);
  - every §4 rule passing in Polar against both implementations;
  - a per-payment amount cap.

### Phase 2: embedded LDK with a third-party LSPS2 LSP

- **Base:** privkeyio's rust-lightning branch, pinned to a reviewed commit. It is fed from Wren's verified header chain rather than a stock Electrum client, and channel state is backed up encrypted.
- **Order:** start **send-only**, with ordinary channels and no LSP standard needed. Add receive only once an outside operator offers LSPS2.
- **Effort estimate:** 3–6 months to beta, plus ongoing rebases.
- **Gates (all required):**
  - a tagged rust-lightning release;
  - published unified-sig test vectors ([bolts #8](https://github.com/lightning-blake2b/bolts/issues/8));
  - force-close interop shown between LDK, CLN and LND (Polar currently states only cooperative close for Lampo);
  - feature bits unchanged for roughly 2–3 months;
  - at least one independently run LSP advertising LSPS2 with published fees;
  - an independent security review;
  - a passing reorg test of force-close safety. [XBTPay](https://github.com/connorslab/paperclip-btcpay-xbt) also lists this test as outstanding.

---

## 4. Wrong-chain rules Wren would enforce on any path

All checks fail closed: if a check cannot be completed, the action is refused.

1. **Payment requests.** Wren decodes every BOLT 11 invoice and BOLT 12 offer or invoice itself. It refuses any without compulsory bit 512, which goes in the BOLT 11 `9` field or the BOLT 12 `offer_features`, `invreq_features` and `invoice_features`. This applies to display and decode paths as well as paying. Wren never relies on a remote node's check.
2. **Legacy format.** Wren refuses `lnblake…` and `tblake…` invoices. The format was withdrawn on 2026-09-19; the LND fork still decodes it, CLN does not.
3. **Fallbacks.** Wren ignores BOLT 11 `f` fallback addresses, because address formats are shared between chains ([spec §4](../specs/2026-09-24-wren-wallet-design.md)).
4. **No implicit cross-chain conversion.** An invoice without bit 512 is refused even when the connected node has a bridge configured.
5. **Remote node or NWC service identity.**
   - `network == "bitcoin"` or `"mainnet"` never counts as evidence.
   - The reported `block_hash` at the reported height must match Wren's own checkpointed BLAKE2b header chain, within its reorg limit.
   - The check repeats every session.
   - For CLN, `our_features.init` must also contain 512.
6. **Peers of an embedded node.** `init` must contain 512 and 515, and the negotiated `channel_type` must contain 514. Taproot channel types are refused. Whether JIT channels must also use anchors is an open question (§6 item 4), since the CLN plugin's JIT type has none.
7. **Funding.** A channel is usable only after its funding transaction confirms on Wren's verified chain. It must confirm at height ≥ 961,640, must not be a coinbase, and must match the expected script and amount. Zero-conf JIT channels have a low cap until they confirm.
8. **Funding inputs.** Channels are funded only from post-split coins with `0x21` signatures. Pre-split coins must first be moved with Wren's Claim flow.
9. **Own signatures.** Every on-chain signature Wren produces (funding, sweeps, anchors, claims) is `0x21`. The one exception is the watchtower justice transaction, which keeps the legacy hash type as the spec requires.
10. **Time.** Safety margins are set in wall-clock hours from observed block rates (§1.5). Block-denominated delays are larger than BTC defaults, and the app warns when it has been offline longer than the margin.
11. **Labels.** Every Lightning amount and request is labeled "XBT Lightning". `lightning:` URIs are never passed to or accepted from BTC handling.
12. **Marketplace payees.** Wren Nest accepts only payee node keys that a chain-verified Wren app registered for that listing. It never accepts pasted invoices or addresses.

---

## 5. Marketplace payment flow sketch (technical framing)

Goal: Wren Nest payment with a single confirmation sheet, with Wren outside the payment path.

1. The buyer taps "Buy". Wren Nest sends the seller's app a trade record: trade ID, amount in sats and listing hash.
2. The **seller's** wallet or node issues a short-lived invoice whose description commits to the trade record.
3. Wren relays the invoice bytes to the buyer.
4. The **buyer's** wallet checks three things, then shows one confirmation and pays:
   - the invoice carries bit 512 (§4 rule 1);
   - the payee is the node key the seller registered for the listing;
   - the amount and trade record match.
5. On success the buyer's wallet holds the preimage (NWC `pay_invoice` also returns it). The buyer's app submits the invoice and the preimage.
6. The Nest backend checks three things: `sha256(preimage) == payment_hash`, the invoice signature against the registered key, and the trade fields. Separately, the seller's app posts a signed "received" acknowledgement. The trade counts as verified only when both are present.

Properties and limits:

- **Wren stays out of the funds.** It runs no node, issues no hold invoices and takes no part of the payment amount.
- **Wren never hosts sellers' Lightning Addresses.** A LUD-16 server chooses which invoice it serves, so a compromised server could substitute one. Source: [LUD-16](https://github.com/lnurl/luds/blob/luds/16.md)
- **A preimage proves payment, not the payer.** It shows the invoice was paid, not who paid it. BOLT 12 payer proofs were specified in July 2026, and CLN's `createproof` is experimental. Sources: [Optech: proof of payment](https://bitcoinops.org/en/topics/proof-of-payment/), [CLN createproof](https://docs.corelightning.org/reference/createproof)
- **Start with in-person pickup, because Lightning payments are final.**
  - At a pickup, both phones are online at the handoff, so the offline-receive limit does not apply.
  - The buyer pays while holding the item.
  - **Shipped goods stay on-chain** (`xbt:`), where confirmations can be watched over the delivery period.

---

## 6. Questions for the lightning-blake2b spec process

These are questions for reviewers' input, not positions.

1. **Flag-day policy.** Would the spec adopt advance notice and an overlap window, in which nodes accept both old and new values, instead of same-hour switches? App-store clients cannot update at a fixed hour, and a downgraded unified channel cannot be recovered (§1.3).
2. **Payer-side refusal.** Should it be normative for payers to refuse BOLT 11/12 payment requests without bit 512, explicitly including decode-only and display paths? Should invoice and offer test vectors cover this?
3. **Legacy format.** Should decoding of `lnblake`, `tblake`, `tbsblake` and `blakert` be formally retired?
4. **XBT LSPS profile.** Would a profile be in scope that specifies:
   - the required channel types for JIT opens (514, and an anchors policy);
   - zero-conf limits;
   - removal of the provisional bit-70 check in the CLN LSPS2 plugin?

   Are operators expected to support LSPS1 and LSPS5?
5. **Verifiable chain identity for app-facing interfaces.** Could conventions for `getinfo` (CLN/LND) and NWC `get_info` expose the activation block hash (height 961,640) alongside the tip hash, so clients can compare both against their own headers? Should the `"chain": "bitcoin-blake2b"` LNURL tag proposed by xbt-zaps be standardized?
6. **Timelock guidance for mobile.** Given block rates between 1.3x and 19x the target, would the spec give wall-clock guidance (minimum `to_self_delay` and `cltv_expiry_delta`) for clients that are often offline?
7. **Test vectors and interop.**
   - Could bolts #8 cover the anchor `0xa3`/`0x21` split and the watchtower legacy exception?
   - Should a cross-implementation force-close test be required before a third implementation goes to mainnet?
   - Should the gossip floor also apply to stored gossip, as CLN #20 proposes?

---

## 7. Sources (accessed 2026-10-09 unless dated otherwise)

- privkeyio CLN: [releases](https://github.com/privkeyio/lightning/releases) · [.4](https://github.com/privkeyio/lightning/releases/tag/v26.06.7-blake2b.4) · [.5](https://github.com/privkeyio/lightning/releases/tag/v26.06.8-blake2b.5) · [.6](https://github.com/privkeyio/lightning/releases/tag/v26.06.8-blake2b.6) · [.7](https://github.com/privkeyio/lightning/releases/tag/v26.06.9-blake2b.7) · [doc/blake2b-upgrade.md](https://github.com/privkeyio/lightning/blob/blake2b-unified/doc/blake2b-upgrade.md) · [compare v26.06.9...blake2b.7](https://github.com/ElementsProject/lightning/compare/v26.06.9...privkeyio:lightning:v26.06.9-blake2b.7) · [issue #20](https://github.com/privkeyio/lightning/issues/20) · [PR #27](https://github.com/privkeyio/lightning/pull/27) · [cln-startos](https://github.com/privkeyio/cln-startos/releases)
- LND fork: [repo](https://github.com/paulscode/lightning-fork) · [release .18](https://github.com/paulscode/lightning-fork/releases/tag/v0.21.4-beta-blake2b.18) · [blake2b-chain-identity.md](https://github.com/paulscode/lightning-fork/blob/blake2b/docs/blake2b-chain-identity.md) · [bridge.md](https://github.com/paulscode/lightning-fork/blob/blake2b/docs/bridge.md) · [lightning-fork-bridge](https://github.com/paulscode/lightning-fork-bridge) · [lightning-fork-swap](https://github.com/paulscode/lightning-fork-swap) · [lightning-fork-android](https://github.com/paulscode/lightning-fork-android)
- LDK: [privkeyio rust-lightning `blake2b`](https://github.com/privkeyio/rust-lightning/commits/blake2b) · [lampo.rs `blake2b`](https://github.com/privkeyio/lampo.rs/commits/blake2b) · [Polar v4.0.0-blake2b.2](https://github.com/privkeyio/polar/releases/tag/v4.0.0-blake2b.2) · [rust-lightning CHANGELOG](https://github.com/lightningdevkit/rust-lightning/blob/main/CHANGELOG.md) · [ldk-node](https://github.com/lightningdevkit/ldk-node)
- Spec: [lightning-blake2b/bolts pulls](https://github.com/lightning-blake2b/bolts/pulls?q=is%3Apr) · [PR #4](https://github.com/lightning-blake2b/bolts/pull/4) · [issue #8](https://github.com/lightning-blake2b/bolts/issues/8)
- Services and clients: [Paperclip node](https://lightning.paperclip-xbt.xyz/) · [paperclip-wallet-app](https://github.com/connorslab/paperclip-wallet-app) · [mobile README](https://github.com/connorslab/paperclip-wallet-app/blob/feature/ios-wallet/mobile/README.md) · [paperclip-btcpay-xbt](https://github.com/connorslab/paperclip-btcpay-xbt) · [xbt-zaps](https://github.com/Kilombino/xbt-zaps) · [control-plane](https://github.com/btcblake2b/control-plane) · [awesome-bitcoin-blake](https://github.com/bitcoin-blake/awesome-bitcoin-blake)
- Protocols: [NIP-47](https://github.com/nostr-protocol/nips/blob/master/47.md) · [LUD-16](https://github.com/lnurl/luds/blob/luds/16.md) · [bLIP-52 (LSPS2)](https://github.com/lightning/blips/blob/master/blip-0052.md) · [Optech: proof of payment](https://bitcoinops.org/en/topics/proof-of-payment/) · [CLN createproof](https://docs.corelightning.org/reference/createproof)
- Other mobile stacks: [ACINQ/lightning-kmp](https://github.com/ACINQ/lightning-kmp) · [Breez SDK (Spark) docs](https://sdk-doc-spark.breez.technology/)
- Wren: [design spec](../specs/2026-09-24-wren-wallet-design.md) · [future ideas](../../future-ideas.md) · [XBT wallet ecosystem notes](./2026-10-09-xbt-wallet-ecosystem.md)
