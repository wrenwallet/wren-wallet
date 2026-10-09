# Wren — Future Ideas and Watch List

Ideas for later phases. None of these are in Wren Wallet v1. Each entry says
what it is, why it matters, what blocks it today, and when to revisit.

---

## XBT Lightning (seen 2026-09-29)

**What:** Paperclip Pool opened a Lightning LSP for XBT
(lightning.paperclippool.xyz/liquidity). It sells inbound liquidity in 0.05,
0.10, and 0.25 XBT channels for a fee of about 1%, with a minimum term of about
90 days (12,960 blocks), paid on-chain after 3 confirmations. It is
non-custodial and labeled experimental. It requires node software
"v26.06.8-blake2b.5" (it looks like a Core Lightning fork; not confirmed) with
custom feature bits: BLAKE2b 512, unified signature 514/515, unified channel
type 514.

**Why it matters:** instant payments for Wren Nest, especially local pickups
where waiting for several confirmations is awkward.

**Blockers today:**
- One LSP, and brand-new custom software with no track record.
- Lightning inside a phone wallet (for example LDK) needs fork patches for
  BLAKE2b headers and unified-signature channels, and adds channel management,
  online-to-receive, and watchtower work.
- XBT is a minority chain where reorgs are cheap, which adds risk to channel
  closes.
- **Wrong-chain invoices (answered in the Gate A review):** XBT and BTC
  Lightning share the genesis hash (the chain hash in BOLT12 and gossip) and the
  address prefixes, so BOLT11 invoices on both start `lnbc`. The prefix and chain
  hash can't tell the chains apart, and an invoice's `f` fallback address is
  valid on both chains. If Wren adds Lightning, it must require the fork's even
  BLAKE2b feature bit (512) in the invoice's own features, refuse invoices
  without it, and ignore `f` fallbacks.
- **Still open:** whether the fork's Lightning software sets bit 512 in
  invoices, or only in node and connection features. Ask the fork's Lightning
  developers.

**Revisit when:** there is more than one LSP, the fork's Lightning software has
a track record, and invoices are confirmed to carry the BLAKE2b feature bit.

**Update 2026-10-09:** invoices and BOLT 12 offers carry compulsory bit 512
since privkeyio CLN `.5` (2026-09-28). A phased path is proposed for review in
[`superpowers/research/2026-10-09-xbt-lightning.md`](superpowers/research/2026-10-09-xbt-lightning.md):
Phase 0 spec rules (v1 recognizes Lightning strings and explains they are not
supported), Phase 1 connect-your-own-node (REST over Tor, NWC) with one-tap
Wren Nest payments, Phase 2 embedded LDK using a third-party LSPS2 provider.
Wren itself runs no Lightning infrastructure in any phase.

## x402 payments (looked at 2026-09-29)

**What:** an open standard, started by Coinbase and now run by the x402
Foundation, that reuses HTTP "402 Payment Required." A server answers "pay X
to Y," the client pays and retries, and a facilitator verifies and settles. It
is built for small automatic payments, mostly by AI agents, and mainly uses
USDC on fast chains. Block added main-chain Bitcoin Lightning support around
2026-09-25.

**Fit for Wren Nest:** not for v1. It conflicts with current decisions:
on-chain XBT is too slow for it, stablecoins are deferred until legal review,
the facilitator sits between buyer and seller (Nest pays sellers directly),
and Nest buyers are people making one-off purchases, not machines.

**Possible later use:** borrow the 402 pattern for **promoted-listing
payments** (fresh address per payment, backend watches the chain). With XBT
Lightning, adapt x402's Lightning scheme for instant payment.

**Revisit when:** XBT Lightning is mature (above), or AI shopping agents
become common.

## Tax help for sellers (looked at 2026-10-06)

**What Square does:** Square merchants price in dollars and sales tax is added
to the dollar total as usual; the customer pays in bitcoin over Lightning. By
default Square converts each payment to dollars instantly, so the merchant
sees an ordinary dollar sale. If the merchant keeps bitcoin, the dollar value
at receipt is income and becomes the cost basis, and Square tracks gains with
FIFO. Square sends the merchant and the IRS a Form 1099-DA, and the app keeps
every bitcoin transaction with a CSV export.

**Why Wren can't copy it:** Square can file 1099s because it is a custodial
broker that holds funds, does the conversion, and has every merchant's tax ID.
Wren is non-custodial with no KYC, and Nest never touches funds, so there is
nothing to file and no one to file for. There is also no automatic conversion
to dollars in v1, so every Nest sale is like Square's "keep the bitcoin" case.
XBT trades only on thin exchanges (NeoxEX, NonKYC), so a dollar value at
receipt is an estimate. No small-purchase crypto tax exemption is law yet:
proposals range from fees-only (House) to $300 (Lummis, Senate) to $600
(an industry group that includes Block).

**Idea (later phase, no KYC):**
- Tax export in Wren Wallet: a CSV of every receive and send with date, XBT
  amount, txid, fee, and a dollar estimate from a named price source, for the
  user's own tax software or accountant.
- Optional "sale" label and note on a receive, so Nest sellers can separate
  business income from personal transfers.
- Clear in-app wording: taxes are the user's responsibility, and Wren reports
  nothing to anyone.

**Revisit when:** the Nest legal review happens, or a de minimis exemption
becomes law (that would change what the export needs to show).

**Sources:** Square Bitcoin Tax Guide
(squareup.com/us/en/the-bottom-line/managing-your-finances/square-bitcoin-tax-guide),
squareup.com/us/en/bitcoin, cointracking.info/blog/crypto-de-minimis-tax-exemption.
