# Wren Wallet — Design Spec (v1)

**Date:** 2026-09-24 (revised 2026-09-27: Claim your XBT added to v1; revised
2026-09-29 at owner's request: Wren PIN app lock, Face ID optional)
**Status:** Approved by owner 2026-09-28; 2026-09-29 revision requested by owner; amended 2026-10-08 after the Gate A review (see `docs/superpowers/reviews/2026-10-08-gate-a.md`)
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
| Claim your XBT | In v1. Hardware signer only: SeedSigner with the unified-sighash fork firmware, over QR. Wren never asks for, accepts, or stores a seed that holds BTC. No standalone "check for twins" tool. When a claim scan finds nothing, Wren says what it checked and why it may have missed coins. |
| Hardware signers | Used for Claim only in v1. The everyday Wren wallet keeps its keys on the phone. Hardware signing for everyday sends comes in a later release. |
| Network privacy | Tor built in via Arti, on by default. |
| Default server | A project-run server on rented hosting. Users can switch to their own node. |
| Open source | Yes. MIT license, public repository before the first public beta. Users can verify the code that holds their money, as with Aqua, Sparrow, and Shrike. |
| Monetization | None in the wallet v1: no fees, no paid features, no in-app tipping. Donations go through a web page. |
| Visual style | Warm red gradient (vermilion to crimson to deep oxblood, the XBT/Shrike red family), frosted glass panels, a glossy red coin with the faceted wren logo as the hero object, white type. Dark-first. Matches wrenwallet.app. |

---

## 3. Architecture

Four parts, each with one job:

```
┌─────────────────────────────────────────────┐
│ App — React Native (Expo, dev-client build) │  screens, camera; native key vault:
│                                             │  Wren PIN pad, sealed seed
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
  SSL, so the app never handles network traffic to the server. Unencrypted
  connections are allowed only to onion addresses. A custom SSL server with a
  self-signed certificate is trusted by its public key (SPKI SHA-256), captured
  over the same route the wallet uses (Tor when Tor is on).
- **Public API**, as exposed to TypeScript:
  - `createWallet() -> words` (shown once)
  - `restoreWallet(words, passphrase?)`, then `restoreCheck()` and `finishSetup()`
  - `confirmBackup(answers)`
  - `unlock()`, `lock()`
  - `revealWords()` (always asks for the Wren PIN)
  - `changePin()`
  - `sync()`
  - `balance() -> { spendable, incomingPending, immature }`
  - `transactions()` (each with a confirmation count)
  - `nextReceiveAddress()`
  - `buildPayment(address, amount | sendAll, feeRate) -> reviewSummary`
  - `signAndBroadcast(reviewId) -> Sent | Rejected(reason) | Unknown`
  - `bumpFee(txid, newFeeRate)`
  - `feeEstimates() -> slow, normal, fast, unreliable`
  - `setServer(config)`, `setTor(on)`, `disconnect()`, `probeCertificate(host, port)`
  - `connectionStatus()`
  - `parsePaymentLink(uri)`
  - `makePaymentLink(address, amount?)`
  - Claim (§4.1):
    - `claimSupportedTypes() -> scriptTypes`
    - `claimStart(xpubQr) -> claimId`
    - `claimScan(claimId, searchFurther?) -> foundSummary`
    - `claimBuild(claimId, destination, feeRate) -> reviewSummary + unsignedPsbtQr`
    - `claimFinish(claimId, signedPsbtQr) -> per-transaction Sent | Rejected(reason) | Unknown`
    - `claimCancel(claimId)`
- Sending asks the phone's key vault for the Wren PIN (or Face ID/fingerprint)
  inside the engine itself, so the app can never sign on its own. Between actions
  the engine holds no private key.
- **QR transport:** animated BC-UR encoding and decoding of PSBTs and xpubs lives
  in the engine, so the app only passes QR frames through.

### 3.2 Bridge

`uniffi-bindgen-react-native` generates the TypeScript/JSI bindings for iOS and
Android from the engine's UniFFI interface. The app calls engine functions and
receives plain results. It never receives keys, the Wren PIN, or the key that
unlocks the seed. The 12 words pass through the app only when they are shown at
Create, typed at Restore, and shown by View recovery phrase after the Wren PIN;
three of them are also typed back at the backup check.

### 3.3 App (React Native, Expo)

- Expo with a dev-client/prebuild workflow, required for the native Rust module.
- The engine encrypts the seed. Wren's own native key-vault module (Swift on
  iOS, Kotlin on Android) keeps the encrypted seed on this phone only (Keychain
  "this device only" on iOS, private no-backup storage on Android). It is never
  in iCloud or Google backups. The app's JavaScript cannot read, change, or
  delete it.
- The Wren PIN pad and the final "Confirm send" sheet are native screens. The
  PIN is never typed into the app's JavaScript, and the confirm sheet shows
  exactly what the engine will sign.
- The key behind the Wren PIN lives in the Secure Enclave (iOS) or a
  non-exportable Keystore key, StrongBox where available (Android).
- The optional Face ID/fingerprint shortcut uses a biometric-only
  Keychain/Keystore item with **no device-passcode fallback**. The fallback is
  always the Wren PIN.
- `expo-camera` handles QR scanning.
- `expo-screen-capture` blocks screenshots on seed screens (always on Android;
  on iOS as far as the system allows).
- No analytics, no remote crash reporting, no third-party SDKs that make network
  calls.

### 3.4 Payment links

Payment links use an **XBT-only URI scheme, `xbt:`**, in the form
`xbt:<address>?amount=<xbt>`. They never use `bitcoin:`, so a main-chain wallet
on the same phone can never open an Wren Wallet payment request and pay on the
wrong chain. The app registers as the handler for `xbt:`. Later, the marketplace
will use this same scheme for one-tap payment.

A plain address, scanned or pasted without `xbt:`, is accepted, but Send warns:
"This is a plain address. Make sure the person expects **XBT (BLAKE2b chain)**,
not BTC." "XBT" is also used for BTC on some exchanges, so the app labels the
coin "XBT (BLAKE2b chain)" wherever a mix-up would cost money.

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
- **Storage:** the engine encrypts the seed, and Wren's native key vault keeps it
  on this phone only, never in cloud backups.
- **App lock: Wren PIN, with optional Face ID or fingerprint.**
  - Every wallet has a **Wren PIN**: 6 digits by default, set during Create or
    Restore, entered twice, and separate from the phone's own passcode. In
    Settings the user can switch to a longer PIN: 8 to 12 digits, or a passcode
    with letters. Obvious PINs (one repeated digit, straight runs such as 123456
    or 654321) are refused at every length.
  - Face ID or fingerprint is an **optional shortcut**, off unless the user turns
    it on. The Wren PIN always works in its place. The phone's passcode never
    unlocks Wren.
  - Every app open and every send asks for the Wren PIN, or biometrics if turned
    on. Viewing the recovery phrase, changing the PIN, and turning biometrics on
    or off **always require the PIN**. Wren locks itself when the user leaves the
    app and after 5 minutes without use.
  - Adding a new face or fingerprint to the phone turns the shortcut off until
    the user turns it back on with the PIN, so a face added by someone else
    never opens Wren.
  - **How the PIN protects the seed.** The seed is locked with a key made from
    two parts. The first is the Wren PIN, slowed down on purpose (Argon2id, about
    1 second per try on the slowest supported phone). The second is a result that
    only this phone's security chip can produce, and that changes with every PIN
    tried. So every guess needs this exact phone.
  - **What that means in practice:**
    - A copy of Wren's storage or a phone backup is useless anywhere else.
    - Someone holding the locked phone gets 5 tries, then growing waits.
    - Someone who can run their own code inside Wren on that phone (for example
      on a jailbroken or rooted phone) is not slowed by the waits. They can try
      every 6-digit PIN in roughly a week. A longer PIN makes that take years.
    - Face ID and fingerprint are protected by the phone's own hardware limits.
  - **Wrong PIN:** after 5 wrong tries, waits grow: 1 minute, 5 minutes,
    15 minutes, 1 hour, then 24 hours per try. Each try is counted and saved
    before it is checked, so closing the app mid-try still counts. Waits are
    timed by the phone's internal clock, so changing the date or time does not
    shorten them; restarting the phone starts the current wait again. The
    counter is kept on this phone only. Wren never auto-erases the wallet,
    because that could destroy coins the user has not backed up yet.
  - **Typing safety:** the Wren PIN uses Wren's own number pad, never the phone
    keyboard. The 12-word field turns off autocorrect, spell-check, word
    suggestions, autofill and keyboard learning. On iOS, Wren does not allow
    third-party keyboards anywhere in the app. On Android, Wren asks the keyboard
    not to learn or suggest, but cannot block third-party keyboards.
  - **App switcher:** when Wren leaves the screen, it is covered with a solid
    cover, so the recovery phrase, balance and addresses never show in the
    recent-apps view.
  - **Forgotten PIN:** there is no reset. The user deletes the app and restores
    from the 12 words. Onboarding says so when the PIN is created.
  - **Wallet from an earlier install:** iOS can keep Wren's saved seed after the
    app is deleted. If Wren finds one, it never deletes it silently. It shows: "A
    Wren wallet from an earlier install is on this phone. Unlock it with its Wren
    PIN, or delete it." Deleting needs the user to type a confirmation and warns:
    "Only delete it if you have your 12 words." The wrong-PIN count carries over.
    After unlocking, sending stays locked until the backup check is done again.
  - Argon2id settings and the wait schedule are flagged for Gate B and the
    external security review.
- **Replay-safe signing is always on.** Every transaction is signed with
  SIGHASH_UNIFIED. There is no setting to disable it. A transaction signed this
  way is invalid on the main chain, so even coins from before the split cannot
  be replayed.
- **Wrong-chain guard:** on every connect, the engine fetches the server's header
  at the fork's BLAKE2b activation height and checks that it is a valid 164-byte
  BLAKE2b header matching the pinned checkpoint. It then verifies every header
  from the pinned split block to the server's tip. A connection counts as
  verified only when that chain has reached activation and carries at least the
  minimum total work pinned from the signed fork Knots release. The engine does
  not rely on the server's own claims, because not every fork indexer reports
  its fork point. It **refuses** any server that fails, including every
  main-chain server and any server that has not reached activation. Sending is
  impossible without a verified fork connection. When a server offers a
  different branch, the engine switches only to a fully verified branch with
  more total work, never more than 144 blocks back. Confirmations of coins from
  before the split count only when the server proves the block against a pinned
  checkpoint of the shared pre-split history; otherwise those coins are shown as
  unverified and cannot be claimed.
- **Addresses:** native SegWit (P2WPKH), BIP84 path `m/84'/0'/0'`, which matches
  Shrike and Sparrow. Any Wren Wallet seed can be restored in Shrike, so users are
  never stranded if the app disappears. This is the same path main-chain wallets
  use, kept on purpose for restorability; the Restore warning and the BTC-seed
  check below guard against the mix-up it could cause.
- **Restore warning:** the Restore screen states: "Only restore a seed that holds
  XBT alone. Never type a seed that holds BTC into Wren or any phone app. To
  move XBT twins from a BTC wallet, use Claim." Restoring a BTC seed would put
  it on an online phone and reveal its addresses to the fork server.
- **BTC-seed check:** after the restore scan, and before anything is saved, Wren
  looks for wallet activity from before the split (block 961,632). If it finds
  any, this phrase was used for bitcoin and holds or held BTC. Wren does not
  save it and shows: "This recovery phrase was used for bitcoin before the
  split, so it holds or held BTC. Wren will not keep it. Typing it into a phone
  put your BTC at risk: move your BTC to a new wallet with your hardware wallet
  now. To claim the XBT twins, use Claim with a hardware signer, or restore this
  phrase in Shrike on a computer." If such activity only shows up later, Wren
  blocks all sending except one "Send all" to move the XBT out, after the user
  confirms they understand. Wren cannot spot a phrase that was only used for
  bitcoin after the split, so the Restore warning stays.
- **Address formats are shared between chains.** No address can reveal its chain.
  Wrong-chain protection therefore comes from the `xbt:` scheme and the server
  check, not from address validation.
- **Send review:** the full address is shown (never truncated), along with the
  amount, fee, and a "Replay-safe" badge. An address that came from the clipboard
  is flagged with "Pasted — check every character," because clipboard-swapping
  malware is common. The review shows a "To yourself" notice when the
  destination is one of the wallet's own addresses. When the fee is 10% or more
  of the amount, or the rate is above 200 sat/vB, it asks for a second
  confirmation. Wren never builds a payment above 1,000 sat/vB. Server error
  text is never shown; Wren uses its own fixed messages.

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
  a row, with a "Search further" option for wallets with gaps. **Claim requires
  Tor.** The scan and the broadcast run on their own Tor circuit and Electrum
  session, never the one the Wren wallet uses, so the server can't link the
  claimed wallet to the Wren wallet. If Tor is off, Claim says "Claim needs Tor.
  Turn on Tor in Settings." The server sees the scanned addresses (as script
  hashes), never the xpub. The server must prove pre-split coins with a
  checkpoint proof; coins it can't prove are listed as not claimable.
- **Supported wallet types:** only the script types the pinned fork firmware can
  sign with SIGHASH_UNIFIED. Native SegWit (`bc1q`) is expected; legacy (`1…`),
  nested SegWit (`3…`), and Taproot (`bc1p`) must be confirmed against the
  firmware during planning. Unsupported types are named before scanning starts,
  never discovered halfway through. An export that includes any other type
  (multisig, Taproot with scripts) is refused as a whole, naming the type.
- **Unsigned claim:** every input's PSBT sighash field is set to SIGHASH_UNIFIED.
  All found coins are swept; there is no change output. Coins worth less than the
  fee to move them, unconfirmed coins, coins not yet mature and coins the server
  couldn't prove are left out and listed with the reason. Each claim transaction
  pays a fresh address in the Wren wallet.
- **Verification before broadcast.** All must pass, or nothing is sent:
  1. Every signature uses SIGHASH_UNIFIED.
  2. Every signature is valid against the fork's signature digest.
  3. The inputs are exactly the coins found in the scan.
  4. The destination and amount are exactly what the user reviewed.
  5. The fee is exactly what the user chose.
- **Unsafe signature:** if any signature could be valid on the BTC chain (a
  normal signature, or a 0x21 byte over a normal digest), Wren sends nothing,
  wipes the scanned QR data, and shows the "Stop" screen (§5, Claim step 8).
  Signatures that are simply wrong get "A signature doesn't check out. Nothing
  was sent."
- **Many coins:** the claim is split so that each transaction's QR stays small
  enough for the SeedSigner to scan (a 50 kB PSBT limit to start, set by device
  testing), and so no transaction exceeds standard size. The transactions are
  signed one after another. The review screen says how many there are before
  signing starts.

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
  passphrase field under "Advanced," a history scan with the BTC-seed check
  (§4), then the Wren PIN setup (Create, step 4). Restoring needs a verified
  server connection; nothing is saved until the check passes.
- **Home:**
  - The spendable XBT balance as the centerpiece on a frosted panel, with a sats
    toggle. Unconfirmed incoming payments appear on a separate "Incoming
    (unconfirmed)" line and become spendable only once confirmed. The wallet's
    own unconfirmed change counts as spendable.
  - Send and Receive buttons, plus a "Claim XBT" entry.
  - A recent activity list.
  - A status line showing the route actually in use (Tor or direct), the server
    connection and Tor start-up progress, with a warning when the server's latest
    block is more than 2 hours old.
  - No USD value. XBT markets are too thin for a reliable price.
- **Receive:** QR code and address, copy and share buttons, and a fresh address
  each time, until 10 shown addresses are still unpaid. From then on Wren shows
  the oldest unpaid address again, so a restore from the 12 words always finds
  every payment. An optional amount builds an `xbt:` link.
- **Send:**
  1. Scan a QR code or paste an address, with the clipboard flag.
  2. Enter an amount, or choose "Send all."
  3. Choose a fee: slow, normal, or fast from server estimates, or custom (1 to
     1,000 sat/vB). If the server's estimates look unreasonable, Wren uses the
     minimum fee and says so.
  4. Review, with the Replay-safe badge.
  5. Confirm on Wren's native confirm sheet with the Wren PIN, or Face
     ID/fingerprint if turned on.
  6. Result: "Sent". Or "Sent, but the server did not confirm it" (Wren keeps
     rebroadcasting; *Speed up* stays available). Or "The server rejected it:
     <reason>" (the payment stays listed; *Speed up* re-sends the same coins, so
     it can never pay twice).
- **Speed up:** offered only on a pending payment signed by this wallet's own
  key, including one whose result was "not confirmed" or "rejected". It replaces
  the payment by re-spending the same coins at a higher fee (RBF).
- **Claim XBT** (rules in §4.1). Cancel at any step wipes the xpub and sends
  nothing.
  1. Intro cards: what twins are (one sentence), why claiming needs a hardware
     signer, "Wren will never ask for your seed words. Anyone who does is a
     scammer," and a link to the SeedSigner guide.
  2. Connect: scan the xpub QR shown on the SeedSigner.
  3. Scan: shows the XBT and coin count found, and any coins that can't be
     claimed, with the reason. If none: "**This server found no XBT for this
     account.**" It then lists the reasons it may have missed coins: a different
     wallet type or account, a missing passphrase, coins past the first
     addresses ("Search further"), coins received after the split, coins not yet
     confirmed. Then: "This doesn't prove the wallet has none." Then stop.
  4. Destination: the user's Wren wallet by default, with a fresh address for
     each claim transaction. "Send elsewhere" accepts any XBT address by scan or
     paste, shown in full, with the clipboard flag.
  5. Fee: slow, normal, fast, or custom.
  6. Review: sweep of all claimable coins, no change, the Replay-safe badge, and
     "This moves XBT only. Your BTC stays where it is." It also says: "These
     coins are spent together, which shows they belong to one wallet, on both
     chains."
  7. Sign. First, a "**Before you sign**" card: "On your SeedSigner, look for the
     line **Unified sighash (0x21)**. If it says **Standard sighash**, don't
     sign. Press back. Your signer needs the fork firmware. See the guide." Then
     Wren shows an animated QR; the SeedSigner scans it, shows the details on its
     own screen, and signs; Wren scans the signed QR back.
  8. Verify and broadcast (§4.1). If a signature is unsafe, Wren shows the
     "Stop" screen:
     **Stop. This signature isn't safe.** Your signer made a normal bitcoin
     signature. Wren did not send it. But the signature still exists, on your
     signer's screen and in the QR code. Anyone who copied it could use it to
     move these coins to {destination}, on the BTC chain and on the XBT chain.
     To make it useless: (1) update your SeedSigner to the fork firmware (see
     the guide); (2) sign this claim again and finish it, which moves your XBT;
     (3) then move your BTC to a new address with the wallet you normally use
     for BTC. Until you've done step 3, keep your signer and this phone with
     you. [Open the guide] [Sign again]
  9. Success: "**Done. The {N} coins listed are now split: their BTC and XBT are
     separate.**" Then any coins not claimed, each with its reason (not yet
     confirmed, worth less than the fee, too large for QR, couldn't be proven).
     Then: "**Claim before you spend BTC from this wallet.** Coins this scan
     didn't find are not split, and spending their BTC could also move their
     XBT." The xpub is wiped.
- **SeedSigner guide:** a plain-language guide, in the app and on the website:
  what to buy, how to flash and verify the pinned fork firmware, and how to
  claim.
- **Activity detail:**
  - Confirmation count and transaction ID. Incoming unconfirmed items carry an
    "Unconfirmed — not spendable yet" tag.
  - An optional "Open in explorer" link, with a warning that the explorer will
    see the transaction.
- **Settings:**
  - Server: the default project server, or a custom one (host, port, SSL/onion).
    Unencrypted servers only as onion addresses. For a custom SSL server with a
    self-signed certificate, Wren shows the certificate's SHA-256 fingerprint
    (colon-separated hex, as `openssl` shows it) for the user to compare with
    their node; if the server's key later changes, Wren stops and asks before
    trusting the new one. Tor is on by default; turning it off asks for
    confirmation and never happens automatically.
  - View recovery phrase, which always requires the Wren PIN.
  - Units: XBT or sats.
  - App lock: change the Wren PIN or switch to a longer PIN, and turn Face
    ID/fingerprint on or off. All require the current PIN.
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

The app matches the website (wrenwallet.app). The palette is the red family used
by the XBT community and the Shrike wallet, not BTC orange. It was derived from
the original green design by keeping each color's lightness and relative
strength and turning the hue to red, so the depth and fade carry over.

- **Dark mode (primary):**
  - Background gradient from vermilion `#B83921` at the top-left, through
    crimson `#571521`, to deep oxblood `#380D14` at the bottom-right, with a
    soft ember glow (`#FF6040` at about 30%) in the top-left corner.
  - Frosted glass panels: a translucent rose fill (`#D2B9B6` at about 14-35%
    opacity), a backdrop blur, and a 1 px light border at about 22% white.
  - Accents:
    - Ember coral `#FF7F5F` for highlighted words and focus rings.
    - Ember red `#F0453A` with white text for the primary action button.
    - Gold `#FFD666` only for warnings ("Stay safe", the Claim "Stop" screen),
      so a warning never looks like a normal action.
  - Type: white `#FFFFFF` primary, white at 88% (`#FFFFFFE0`) secondary,
    raised from 70% for easier reading. System font, SF Pro on iOS and Roboto
    on Android, so Dynamic Type and font scaling work everywhere.
- **Light mode:** a warm off-white background (`#F7EFED` to `#E5DEDD`), white
  frosted panels, deep wine `#49121C` type (`#6C212B` secondary), and a deeper
  red accent `#C01D26` with white text on buttons.
- **Hero object:** on Home, a glossy 3D red coin (highlight `#FFC2B0`, body
  `#F0453A`, edge `#B7282F`, shadow `#61141F`) carrying the Wren logo: a
  faceted, low-poly wren, standing upright and alert, in cream-to-rust facets.
  The sharp facets are a nod to Shrike; the calm, rounded bird says Wren is
  peaceful and social. It is static, with a subtle idle shimmer that respects
  Reduce Motion.
- **App icon and splash:** the same red coin and wren. The square icon fills
  the frame with the red gradient and the wren.
- **Components:**
  - Large rounded panels (radius 24) and pill buttons.
  - One primary action per screen.
  - Blur via `expo-blur`, with a solid fallback (`#4E161C` at 94%) when the OS
    has reduced transparency turned on.
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
- **Replay proof, on every build (required CI check):** every transaction the
  engine signs that spends a pre-split coin, including all four claim types, is
  mined on a Bitcoin Core regtest node that shares the pre-split history, and
  **must be rejected by consensus** with the expected script error. As a
  control, the same transaction signed normally must be accepted there.
- **Claim tests, on every build (required CI check):**
  - **Signer compatibility:** real SeedSigner fork-firmware exports and signed
    PSBTs (captured from a device) are imported and verified, and the firmware's
    own PSBT parser and signer, run without its screen, sign each claim type in
    the local test chain. The full device or emulator run happens at Gate B and
    before release.
  - **Refusals:** the engine must refuse to broadcast when a signature is unsafe
    (normal, or BTC-valid), an output was changed after review, an input was
    added or removed, or the fee was altered. Unsafe signatures made by real
    software that doesn't know the fork (Bitcoin Core) must be refused for every
    claim type.
  - **QR round trip:** large animated BC-UR PSBTs survive encode, scan, and
    decode intact. QR codes claiming absurd sizes, or that never complete, are
    refused.
  - **Wipe:** after finish, cancel, or a force-quit mid-claim, no trace of the
    xpub remains in app storage.
  - **Supported types:** every script type Claim accepts is signed and verified
    end to end; every other type is refused before scanning.
- **Mainnet data rules in the test chain:** the local test chain turns on the
  fork's data limits and long coinbase maturity, so tests run under mainnet
  rules.
- **Wrong-chain proof:** pointing the engine at a main-chain Electrum server (for
  example a stock electrs) must fail to connect, and sending must stay
  impossible.
- **Header-chain tests:** weaker, equal-work and invalid competing branches are
  refused and the stored chain is unchanged; a server below activation is
  refused; a fabricated pre-split confirmation is never counted.
- **Server-trust tests:** an invented unconfirmed spend of a wallet coin is
  refused; *Speed up* refuses any transaction the wallet key did not sign; fake
  incoming payments are never spendable; absurd server fees fall back to the
  minimum; a lost broadcast reply never leads to paying twice.
- **BTC-seed check:** a restored phrase with pre-split history is never saved;
  history found later allows only one send-all.
- **No secrets on disk:** no seed or key bytes are ever written to Wren's files.
- **App lock tests:**
  - Wrong-PIN waits survive an app restart and a force-quit.
  - Biometrics can never view the recovery phrase or change the PIN.
  - Enrolling a new face or fingerprint turns the shortcut off.
  - A wallet from an earlier install is never deleted without the typed
    confirmation.
  - The saved seed copied to another phone cannot be unlocked with the right PIN.
  - Obvious PINs are refused, at every PIN length.
  - A wrong PIN still counts if the app is closed during the check.
  - Changing the phone's date or time does not shorten a wait.
  - The app's JavaScript cannot send, view the recovery phrase, or change the
    PIN without the native PIN pad or biometrics.
  - Wren locks on leaving the app and after 5 minutes idle.
- **App tests:** unit tests for screen logic, plus Maestro end-to-end flows on
  the iOS simulator and Android emulator covering onboarding, the backup check,
  receive, send, and settings.
- **Real devices:** a developer iPhone and an Android test phone. These cover
  the Wren PIN, biometrics, camera scanning, and secure storage.
- **Real-chain trial:** a small round trip between Wren Wallet and Shrike on the
  live fork chain, verified on a developer-run fork node.
- **Claim trials:**
  - At Gate B: a SeedSigner (privkeyio unified-sighash firmware, real device or
    emulator) claims from a test wallet on the local test chain, running the
    full flow on a real phone. Its exports and signed PSBTs become the test
    fixtures.
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
| Arti onion-client maturity | SSL direct mode as a fallback the user chooses (never automatic). Tor start-up gives up after 2 minutes with a clear message. The actual route is shown on Home. |
| App Store rejection | Organization account; no swaps or fiat; clear non-custodial description. |
| Thin XBT market confuses newcomers | No USD display; self-custody cards in onboarding. |
| Claim depends on one community firmware developer (privkeyio's SeedSigner unified-sighash build) | Pin a verified firmware version; the guide shows how to verify its signature; add other signers if their makers support SIGHASH_UNIFIED. |
| User forgets the Wren PIN | No reset, by design. Restore from the 12 words; onboarding says so at PIN setup. Sending is locked until the backup check passes, so anyone who can send has confirmed their words. |
| A 6-digit PIN has only 1,000,000 combinations | Each guess needs this phone's security chip, so a copy of Wren's storage or a backup is useless elsewhere. Someone holding the locked phone gets 5 tries, then waits up to 24 hours per try. But someone who can run their own code inside Wren on that phone (a jailbroken or rooted phone, or a tampered app) skips the waits and could try every 6-digit PIN in about a week. Users can choose a longer PIN (8 to 12 digits, or with letters), which raises this to years. Argon2id cost and the wait schedule are reviewed at Gate B and in the external review. |
| Fake Wren apps ask users for their seeds | Official links only; open source; "Wren never asks for your seed" in the app and on the website; claiming is hardware-signer only, so a real Wren never asks. |
| Claim scan reveals old BTC addresses to the fork server | Only during an actual claim; Tor required, on a separate session the server can't link to the Wren wallet; xpub never leaves the phone; stated in the privacy policy. |
| One server shows a privately mined branch with fake confirmations | Full header verification, minimum total work and most-work selection limit what it can fake; stale-tip warning; Nest asks for 6 confirmations (Nest plan). A second-server cross-check is deferred past v1. |
| A signer makes a normal (BTC-valid) claim signature | Pre-sign check on the SeedSigner screen; Wren refuses and wipes it, and tells the user to finish the claim and then move their BTC. |
