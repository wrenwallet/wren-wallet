# Wren Wallet — Design Spec (v1)

**Date:** 2026-09-24 (revised 2026-09-27: Claim your XBT added to v1; revised
2026-09-29 at owner's request: Wren PIN app lock, Face ID optional)
**Status:** Approved by owner 2026-09-28; 2026-09-29 revision requested by owner
**Owner:** chans
**Project:** XBT project, sub-project 1 of 4

---

## 1. Purpose

Wren Wallet is a simple, clean, non-custodial mobile wallet for **XBT**, the coin
of the BLAKE2b fork chain. It exists to make XBT easy to hold and spend on a
phone, which no wallet does today. The fork's only wallets are desktop builds of
Sparrow (Shrike, paulscode Sparrow).

It also gives pre-split BTC holders a safe way to **claim their XBT twins**: the
XBT that exists at the same addresses on the fork chain because they held BTC
before the split at height 961,632. Claiming is signed on a hardware signer, so
Wren never touches a seed that holds BTC (see §4.1).

It is the first piece of a larger project whose goal is to grow XBT as a unit of
trade for goods and services:

1. **Wren Wallet** (this spec): an XBT-only mobile wallet for iOS and Android.
2. **Wren Nest backend + web app**: an OfferUp-style, no-KYC, non-custodial
   marketplace priced in XBT.
3. **Wren Nest mobile app**: pays in one tap through Wren Wallet.
4. **Swaps**: USDT/USDC/BTC to XBT, added to the wallet later, after legal review.

Each sub-project gets its own spec, plan, and build cycle. This spec covers
sub-project 1 only.

### Audience

- **Priority 1:** pre-split BTC holders who are afraid of losing their XBT twins
  and want them moved safely into a clean, XBT-only wallet.
- **Priority 2:** people with a clean Wren wallet who spend XBT on Wren Nest.
- **Also:** fork community members who already hold XBT.
- **End goal:** the public, including people new to XBT. Buying XBT with
  stablecoins inside the app is deferred to sub-project 4, after legal review.

### Success criteria for v1

- A user can create or restore a wallet, back it up, receive XBT, and send XBT on
  both iOS and Android.
- A user with a SeedSigner running the fork firmware can claim the XBT twins of
  a pre-split wallet into Wren, or to any other XBT address, without their seed
  ever touching the phone.
- Every transaction the wallet signs or broadcasts is replay-safe. This is proven by an
  automated test on every build, not assumed.
- The wallet never connects to a main-chain server.
- The external security review is passed before public release.
- The app collects no user data. The App Store privacy label is "Data Not
  Collected."

---

## 2. Fixed decisions

| Topic | Decision |
|---|---|
| Coins | XBT only. No BTC, no stablecoins, no swaps. No cross-chain transactions until replay is a non-issue. |
| Custody | Non-custodial. Keys live only on the user's device. No accounts, no KYC. |
| Platforms | iOS and Android, shipped together. |
| Architecture | React Native (Expo) app over a shared Rust engine. Chosen so the wallet, marketplace mobile app, and marketplace web app share TypeScript/React. |
| Backup | 12-word BIP39 phrase on paper. Optional BIP39 passphrase under advanced settings. No cloud backup. |
| Claim your XBT | In v1. Hardware signer only: SeedSigner with the unified-sighash fork firmware, over QR. Wren never asks for, accepts, or stores a seed that holds BTC. No standalone "check for twins" tool: if a claim scan finds nothing, the user knows they have no twins. |
| Hardware signers | Used for Claim only in v1. The everyday Wren wallet keeps its keys on the phone. Hardware signing for everyday sends comes in a later release. |
| Network privacy | Tor built in via Arti, on by default. |
| Default server | A project-run server on rented hosting. Users can switch to their own node. |
| Open source | Yes. MIT license, public repository before the first public beta. Users can verify the code that holds their money, as with Aqua, Sparrow, and Shrike. |
| Monetization | None in the wallet v1: no fees, no paid features, no in-app tipping. Donations go through a web page. |
| Visual style | Deep forest/olive green gradient, frosted glass panels, glossy 3D green hero object, white type. Dark-first. |

---

## 3. Architecture

Four parts, each with one job:

```
┌─────────────────────────────────────────────┐
│ App — React Native (Expo, dev-client build) │  screens, camera, app lock,
│                                             │  secure storage of the seed
├─────────────────────────────────────────────┤
│ Bridge — generated TypeScript bindings      │  uniffi-bindgen-react-native
├─────────────────────────────────────────────┤
│ Engine — Rust (patched BDK + Arti)          │  keys, addresses, sync, build,
│                                             │  fee, sign, broadcast, Tor
└───────────────┬─────────────────────────────┘
                │ Electrum protocol over Tor (default) or SSL
┌───────────────▼─────────────────────────────┐
│ Server — fork Knots + Fulcrum (BLAKE2b)     │  project-run host
└─────────────────────────────────────────────┘
```

### 3.1 Engine (Rust)

Everything that touches keys or money lives here. No private key ever crosses
the bridge.

- **Base:** `bdk_wallet` + `bdk_electrum` on `rust-bitcoin`, pinned to exact
  versions.
- **Fork patches**, kept as a small, documented patch set in the repo:
  - **164-byte BLAKE2b block headers.** Upstream `rust-bitcoin` assumes 80-byte
    headers. The Electrum sync path (header fetch, merkle-proof checks) must parse
    and verify fork headers.
  - **SIGHASH_UNIFIED signing.** This is the fork's opt-in replay-protection
    signature type from Knots PR #357, released in Knots v29.4.1.knots20260508.
    The flag value and digest algorithm are implemented from the Knots source.
    Byte-exact agreement with Knots and Shrike is required (see §8).
- **Tor:** the `arti-client` crate is embedded, with onion-service client support
  enabled. The engine opens all server connections itself, over Tor or direct
  SSL, so the app never handles network traffic to the server.
- **Public API**, as exposed to TypeScript:
  - `createWallet() -> words`
  - `restoreWallet(words, passphrase?)`
  - `loadWallet(encryptedSeed)`
  - `sync()`
  - `balance()`
  - `transactions()`
  - `nextReceiveAddress()`
  - `buildPayment(address, amount | sendAll, feeRate) -> reviewSummary`
  - `signAndBroadcast(reviewId)`
  - `bumpFee(txid, newFeeRate)`
  - `feeEstimates()`
  - `setServer(config)`
  - `connectionStatus()`
  - `parsePaymentLink(uri)`
  - `makePaymentLink(address, amount?)`
  - Claim (§4.1):
    - `claimSupportedTypes() -> scriptTypes`
    - `claimStart(xpubQr) -> claimId`
    - `claimScan(claimId, searchFurther?) -> foundSummary`
    - `claimBuild(claimId, destination, feeRate) -> reviewSummary + unsignedPsbtQr`
    - `claimFinish(claimId, signedPsbtQr) -> txids`
    - `claimCancel(claimId)`
- **QR transport:** animated BC-UR encoding and decoding of PSBTs and xpubs lives
  in the engine, so the app only passes QR frames through.

### 3.2 Bridge

`uniffi-bindgen-react-native` generates the TypeScript/JSI bindings for iOS and
Android from the engine's UniFFI interface. The app calls engine functions and
receives plain results. It never receives keys.

### 3.3 App (React Native, Expo)

- Expo with a dev-client/prebuild workflow, required for the native Rust module.
- The seed is stored encrypted (see App lock, §4) with `expo-secure-store`,
  device-only (iOS `WHEN_UNLOCKED_THIS_DEVICE_ONLY`, Android Keystore). It is
  excluded from iCloud and Google backups.
- The hardware-bound key behind the Wren PIN lives in the Secure Enclave (iOS) or
  a non-exportable Keystore key, StrongBox where available (Android). Expo has no
  built-in for this, so it needs a small native module.
- The optional Face ID/fingerprint shortcut uses a biometric-only
  Keychain/Keystore item with **no device-passcode fallback**. The fallback is
  always the Wren PIN.
- `expo-camera` handles QR scanning.
- `expo-screen-capture` blocks screenshots on seed screens.
- No analytics, no remote crash reporting, no third-party SDKs that make network
  calls.

### 3.4 Payment links

Payment links use an **XBT-only URI scheme, `xbt:`**, in the form
`xbt:<address>?amount=<xbt>`. They never use `bitcoin:`, so a main-chain wallet
on the same phone can never open an Wren Wallet payment request and pay on the
wrong chain. The app registers as the handler for `xbt:`. Later, the marketplace
will use this same scheme for one-tap payment.

### 3.5 Repository layout

```
WrenWallet/
  core/      Rust engine + fork patch set + engine tests
  app/       Expo React Native app
  infra/     server setup script, config, runbook
  docs/      specs and plans
```

---

## 4. Key and signing safety

- **Seed creation:** 12 BIP39 words from the OS secure random source, generated
  in the engine. Shown once. Screenshots are blocked.
- **Backup check:** the user confirms three random words. Receiving works
  immediately. **Sending stays locked** until the backup check passes, and a Home
  banner stays up until then.
- **Storage:** the seed is encrypted in the Keychain/Keystore, device-only, and
  never in cloud backups.
- **App lock: Wren PIN, with optional Face ID or fingerprint.**
  - Every wallet has a **Wren PIN**: 6 digits, set during Create or Restore,
    entered twice, and separate from the phone's own passcode. Obvious PINs (one
    repeated digit, straight runs such as 123456 or 654321) are refused.
  - Face ID or fingerprint is an **optional shortcut**, off unless the user turns
    it on. The Wren PIN always works in its place. The phone's passcode never
    unlocks Wren.
  - Every app open and every send asks for the Wren PIN, or biometrics if turned
    on. Viewing the recovery phrase, changing the PIN, and turning biometrics on
    or off **always require the PIN**.
  - Adding a new face or fingerprint to the phone turns the shortcut off until
    the user turns it back on with the PIN, so a face added by someone else
    never opens Wren.
  - **The PIN cannot be guessed off the phone.** The seed's encryption key comes
    from the PIN (Argon2id) combined with a hardware-bound key that never leaves
    the phone's security chip. A copy of the app's storage is useless on its own,
    and every guess must run on that phone.
  - **Wrong PIN:** after 5 wrong tries, waits grow: 1 minute, 5 minutes,
    15 minutes, 1 hour, then 24 hours per try. The counter lives in secure
    storage, so restarting the app does not reset it. Wren never auto-erases
    the wallet, because that could destroy coins the user has not backed up yet.
  - **Forgotten PIN:** there is no reset. The user deletes the app and restores
    from the 12 words. Onboarding says so when the PIN is created.
  - **Reinstall:** iOS keeps Keychain items after an app is deleted. On first
    launch Wren wipes any leftover items, so an old seed or PIN counter never
    carries over.
  - Argon2id settings and the wait schedule are flagged for Gate B and the
    external security review.
- **Replay-safe signing is always on.** Every transaction is signed with
  SIGHASH_UNIFIED. There is no setting to disable it. A transaction signed this
  way is invalid on the main chain, so even coins from before the split cannot
  be replayed.
- **Wrong-chain guard:** on connect, the engine fetches the header at the fork's
  BLAKE2b activation height and independently verifies it as a valid 164-byte
  BLAKE2b header. It also checks the recent chain tip the same way. The
  activation height is pinned in the engine from the final signed fork Knots
  release. The engine does not rely on the server's own claims, because not
  every fork indexer reports its fork point. It **refuses** any server that
  fails, including every main-chain server. Sending is impossible without a
  verified fork connection.
- **Addresses:** native SegWit (P2WPKH), BIP84 path `m/84'/0'/0'`, which matches
  Shrike and Sparrow. Any Wren Wallet seed can be restored in Shrike, so users are
  never stranded if the app disappears. This is the same path main-chain wallets
  use, kept on purpose for restorability; the Restore warning below prevents the
  mix-up it could cause.
- **Restore warning:** the Restore screen states: "Only restore a seed that holds
  XBT alone. Never type a seed that holds BTC into Wren or any phone app. To
  move XBT twins from a BTC wallet, use Claim." Restoring a BTC seed would put
  it on an online phone and reveal its addresses to the fork server.
- **Address formats are shared between chains.** No address can reveal its chain.
  Wrong-chain protection therefore comes from the `xbt:` scheme and the server
  check, not from address validation.
- **Send review:** the full address is shown (never truncated), along with the
  amount, fee, and a "Replay-safe" badge. An address that came from the clipboard
  is flagged with "Pasted — check every character," because clipboard-swapping
  malware is common.

### 4.1 Claim safety

Claiming moves the XBT twins of a pre-split wallet. That wallet's seed still
controls the user's BTC, so the rules are strict:

- **Hardware signer only.** Wren never asks for, accepts, or stores the seed of
  the wallet being claimed. The user's signer (SeedSigner with the
  unified-sighash fork firmware) signs over QR. The app says, in large type:
  "Wren will never ask for your seed words. Anyone who does is a scammer."
- **xpub in memory only.** The xpub scanned from the signer is never written to
  storage. It is wiped when the claim finishes, is cancelled, or the app closes.
  A closed app means the claim starts over.
- **Scanning:** receive and change chains, stopping after 20 unused addresses in
  a row, with a "Search further" option for wallets with gaps. Queries go to the
  fork server over Tor. The server sees the scanned addresses (as script hashes),
  never the xpub.
- **Supported wallet types:** only the script types the pinned fork firmware can
  sign with SIGHASH_UNIFIED. Native SegWit (`bc1q`) is expected; legacy (`1…`),
  nested SegWit (`3…`), and Taproot (`bc1p`) must be confirmed against the
  firmware during planning. Unsupported types are named before scanning starts,
  never discovered halfway through.
- **Unsigned claim:** every input's PSBT sighash field is set to SIGHASH_UNIFIED.
  All found coins are swept; there is no change output.
- **Verification before broadcast.** All must pass, or nothing is sent:
  1. Every signature uses SIGHASH_UNIFIED.
  2. Every signature is valid against the fork's signature digest.
  3. The inputs are exactly the coins found in the scan.
  4. The destination and amount are exactly what the user reviewed.
  5. The fee is exactly what the user chose.
- **Wrong signature type:** if the signer returns a normal signature, Wren refuses
  it with: "Your signer made a normal signature, which could also move your BTC.
  Wren blocked it. Your signer needs the fork firmware — see the guide."
- **Many coins:** if one transaction would exceed standard size, the claim is
  split into several transactions, signed one after another. The review screen
  says so before signing starts.

---

## 5. Screens and flows

- **Welcome:** "Claim your XBT" comes first, as the large primary button, with
  the line "Held bitcoin before the split? Move your XBT safely, without typing
  your seed." Below it are smaller "Create new wallet" and "Restore wallet"
  buttons. Claim first walks the user through creating their Wren wallet, so
  the default destination exists.
- **Create:**
  1. Three self-custody cards: you hold the keys, no one can recover them, write
     the words on paper.
  2. The 12 words.
  3. Confirm three words. This step can be skipped, but sending stays locked.
  4. Create the Wren PIN, entered twice, with the note "Forget this PIN and
     you'll need your 12 words to get back in." Then "Also unlock with Face ID?"
     (or fingerprint), with "Turn on" and "Not now."
  5. Home.
- **Restore:** the Restore warning (§4), 12-word entry with BIP39 autocomplete, a
  passphrase field under "Advanced," the Wren PIN setup (Create, step 4), then
  a history scan.
- **Home:**
  - The XBT balance as the centerpiece on a frosted panel, with a sats toggle.
  - Send and Receive buttons, plus a "Claim XBT" entry.
  - A recent activity list.
  - A status line showing the Tor and server connection.
  - No USD value. XBT markets are too thin for a reliable price.
- **Receive:** QR code and address, copy and share buttons, and a fresh address
  each time. An optional amount builds an `xbt:` link.
- **Send:**
  1. Scan a QR code or paste an address, with the clipboard flag.
  2. Enter an amount, or choose "Send all."
  3. Choose a fee: slow, normal, or fast from server estimates, or custom.
  4. Review, with the Replay-safe badge.
  5. Confirm with the Wren PIN, or Face ID/fingerprint if turned on.
  6. Success screen.
- **Speed up:** a pending outgoing payment offers "Speed up" (RBF fee bump).
- **Claim XBT** (rules in §4.1). Cancel at any step wipes the xpub and sends
  nothing.
  1. Intro cards: what twins are (one sentence), why claiming needs a hardware
     signer, "Wren will never ask for your seed words. Anyone who does is a
     scammer," and a link to the SeedSigner guide.
  2. Connect: scan the xpub QR shown on the SeedSigner.
  3. Scan: shows the XBT and coin count found. If none: "No XBT found for this
     wallet," with the usual reasons (different wallet type, missing
     passphrase, coins received after the split), then stop.
  4. Destination: the user's Wren wallet (fresh address) by default. "Send
     elsewhere" accepts any XBT address by scan or paste, shown in full, with
     the clipboard flag.
  5. Fee: slow, normal, fast, or custom.
  6. Review: sweep of all found coins, no change, the Replay-safe badge, and
     "This moves XBT only. Your BTC stays where it is."
  7. Sign: Wren shows an animated QR; the SeedSigner scans it, shows the details
     on its own screen, and signs; Wren scans the signed QR back.
  8. Verify and broadcast (§4.1).
  9. Success: "**Your coins are now split.** Your BTC and XBT are fully separate.
     Spending one can never move the other." The xpub is wiped.
- **SeedSigner guide:** a plain-language guide, in the app and on the website:
  what to buy, how to flash and verify the pinned fork firmware, and how to
  claim.
- **Activity detail:**
  - Confirmation count and transaction ID.
  - An optional "Open in explorer" link, with a warning that the explorer will
    see the transaction.
- **Settings:**
  - Server: the default project server, or a custom one (host, port, SSL/onion).
    Tor is on by default and can be turned off.
  - View recovery phrase, which always requires the Wren PIN.
  - Units: XBT or sats.
  - App lock: change the Wren PIN, and turn Face ID/fingerprint on or off. Both
    require the current PIN.
  - "About replay protection," a plain-language explainer.

### Out of scope for v1

- BTC, stablecoins, and swaps, including buying XBT in the app.
- Hardware signing for everyday sends (v1 uses SeedSigner for Claim only).
- Typing a BTC-holding seed to claim; claiming is hardware-signer only.
- A standalone "check for twins" tool.
- Marketplace handoffs ("request address," "pay this trade").
- USD pricing.
- Cloud backup.
- Multiple wallets or accounts.
- Lightning.
- Coin control.

---

## 6. Visual design

- **Dark mode (primary):**
  - Background gradient from olive `#4E7A23` at the top-left to deep forest
    `#133A1E` at the bottom-right.
  - Frosted glass panels: a translucent sage fill (`#A9CBA2` at about 35%
    opacity, fading to `#1E4A28` at about 50%), a backdrop blur, and a 1 px light
    border at about 20% white.
  - Accent: bright leaf green `#6CC04A` for primary actions and success states.
  - Type: white `#FFFFFF` primary, `#FFFFFFB3` secondary. System font, SF Pro on
    iOS and Roboto on Android.
- **Light mode:** a soft sage background (`#EEF4EA` to `#D6E6D0`), white frosted
  panels, forest `#1E4A28` type, and the same accent.
- **Hero object:** a glossy, glassy 3D green XBT emblem on Home, in the style of
  the reference's disc. It is static, with a subtle idle shimmer that respects
  Reduce Motion.
- **Components:**
  - Large rounded panels (radius 24) and pill buttons.
  - One primary action per screen.
  - Blur via `expo-blur`, with a solid fallback when the OS has reduced
    transparency turned on.
- **Accessibility:**
  - All text meets WCAG AA contrast on its panel.
  - Dynamic Type and font scaling are supported.
  - Every control is labeled for VoiceOver and TalkBack.

---

## 7. Server

- Wren ships with a project-run default server: fork Knots (unpruned, signed
  releases verified before every install) plus Fulcrum BLAKE2b.
- The wallet reaches it over Tor (an onion service, the default) or SSL.
- The server holds no keys or funds and keeps no logs of client IPs or wallet
  addresses.
- Users can point Wren at their own fork node in Settings at any time.
- Hosting and operations details are kept out of this public repository.

---

## 8. Testing

- **Engine unit tests:**
  - BIP39 and BIP84 official test vectors.
  - Parsing and verifying real 164-byte fork headers from recorded mainnet fork
    blocks.
  - SIGHASH_UNIFIED digests and signatures checked against reference
    transactions produced by fork Knots.
  - **Cross-signing:** the same unsigned transaction is signed by the engine and
    by Shrike, and the outputs must be byte-identical.
- **Local test chain:** fork Knots in regtest plus Fulcrum on a developer machine. The
  full lifecycle runs there: create, receive, send, send all, speed up, and
  confirm.
- **Replay proof, run on every build:** every transaction the engine signs, and
  every claim transaction it would broadcast, is submitted to a stock main-chain
  Knots regtest node and **must be rejected**.
- **Claim tests, run on every build:**
  - **Signer compatibility:** a claim built by the engine and signed by the
    SeedSigner fork firmware (run in the SeedSigner desktop emulator) is
    byte-identical to the same claim signed by Shrike.
  - **Refusals:** the engine must refuse to broadcast when a signature is a
    normal (non-UNIFIED) type, an output was changed after review, an input was
    added or removed, or the fee was altered.
  - **QR round trip:** large animated BC-UR PSBTs survive encode, scan, and
    decode intact.
  - **Wipe:** after finish, cancel, or a force-quit mid-claim, no trace of the
    xpub remains in app storage.
  - **Supported types:** every script type Claim accepts is signed and verified
    end to end; every other type is refused before scanning.
- **Wrong-chain proof:** pointing the engine at a main-chain Electrum server (for
  example a stock electrs) must fail to connect, and sending must stay
  impossible.
- **App lock tests:**
  - Wrong-PIN waits survive an app restart and a force-quit.
  - Biometrics can never view the recovery phrase or change the PIN.
  - Enrolling a new face or fingerprint turns the shortcut off.
  - After delete and reinstall, no seed or PIN counter remains.
  - Copied app storage cannot be unlocked off the phone.
  - Obvious PINs are refused.
- **App tests:** unit tests for screen logic, plus Maestro end-to-end flows on
  the iOS simulator and Android emulator covering onboarding, the backup check,
  receive, send, and settings.
- **Real devices:** a developer iPhone and an Android test phone. These cover
  the Wren PIN, biometrics, camera scanning, and secure storage.
- **Real-chain trial:** a small round trip between Wren Wallet and Shrike on the
  live fork chain, verified on a developer-run fork node.
- **Claim trials:**
  - A SeedSigner (privkeyio unified-sighash firmware) claims from a test wallet
    on the local test chain, running the full flow on a real phone.
  - One real claim on the live fork chain, from a genuine pre-split wallet,
    before any public release.
  - **Guide test:** a non-developer follows the SeedSigner guide cold. Wherever a
    non-technical user gets stuck, the guide is fixed.
- **Release gates:**
  - `cargo audit` and `npm audit` are clean.
  - An external security review of the engine and key storage is complete.
  - All review findings are fixed before any public release.
- **Beta:** fork community testers via TestFlight and Google Play internal
  testing.

---

## 9. Custody and privacy commitments

- Wren Wallet is non-custodial software. It never holds user funds, never acts
  as a counterparty, and handles no fiat and no swaps.
- The project charges no fees on transactions.
- The app collects no data. The App Store privacy label is "Data Not
  Collected."
- Wren is described as private and self-custodial software, never as a tool
  for evading the law.

---

## 10. Risks

| Risk | Mitigation |
|---|---|
| Signing bug loses funds | Byte-exact cross-signing with Shrike and Knots; replay proof on every build; external review; small real-chain trial first. |
| Fork rules change (soft forks, hotfixes) | Pinned engine patch set; fork-watch monitoring; server upgrade runbook in `infra/`. |
| BDK/rust-bitcoin upstream drift | Pin exact versions; keep the fork patch set small and documented; rebase deliberately. |
| Arti onion-client maturity | SSL direct mode as a fallback; Tor status visible on Home. |
| App Store rejection | Organization account; no swaps or fiat; clear non-custodial description. |
| Thin XBT market confuses newcomers | No USD display; self-custody cards in onboarding. |
| Claim depends on one community firmware developer (privkeyio's SeedSigner unified-sighash build) | Pin a verified firmware version; the guide shows how to verify its signature; add other signers if their makers support SIGHASH_UNIFIED. |
| User forgets the Wren PIN | No reset, by design. Restore from the 12 words; onboarding says so at PIN setup. Sending is locked until the backup check passes, so anyone who can send has confirmed their words. |
| A 6-digit PIN has only 1,000,000 combinations | The hardware-bound key means guesses can only run on the phone itself; growing waits after 5 wrong tries; Argon2id settings and the wait schedule reviewed at Gate B and in the external review. |
| Fake Wren apps ask users for their seeds | Official links only; open source; "Wren never asks for your seed" in the app and on the website; claiming is hardware-signer only, so a real Wren never asks. |
| Claim scan reveals old BTC addresses to the fork server | Only during an actual claim; over Tor; xpub never leaves the phone; stated in the privacy policy. |
