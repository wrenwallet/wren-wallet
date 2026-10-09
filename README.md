# Wren Wallet™

A simple, self-custody mobile wallet for **XBT**, the coin of the BLAKE2b fork
chain. iOS and Android.

> **Status: in design. There is no app to download yet.**
> Any app or download that claims to be Wren Wallet today is not ours.
> Wren will never ask for your recovery phrase.

## What Wren is being built to do

- **Hold and spend XBT on your phone.** Your keys stay on your device. No
  accounts, no KYC, no data collected.
- **Claim your XBT safely.** If you held BTC before the split at block
  961,632, you may also hold XBT at the same addresses. Wren lets you move it
  using a hardware signer (SeedSigner), so your BTC seed never touches a phone.
- **Stay on the right chain.** Every signature uses `SIGHASH_UNIFIED`, so it is
  invalid on the BTC chain. Wren uses its own `xbt:` payment links and refuses
  to talk to BTC servers.
- **Keep your network private.** Tor is built in and on by default.

XBT is not BTC. They are separate coins on separate chains.

## Project documents

- [Design spec](docs/superpowers/specs/2026-09-24-wren-wallet-design.md)
- [Engine implementation plan](docs/superpowers/plans/2026-09-28-wren-engine.md)
- [Fork research](docs/superpowers/research/)
- [Future ideas](docs/future-ideas.md)

## Review

Every stage is reviewed by outside bitcoin developers before code is written,
again at the signing milestone, and by a formal security review before any
public release. See [SECURITY.md](SECURITY.md) to report a problem.

## License

Code: [MIT](LICENSE). The names and logo are not covered by the license; see
[TRADEMARKS.md](TRADEMARKS.md).

Wren Wallet is non-custodial software provided as is, with no warranty. It is
not financial advice. You are responsible for your keys and your coins.

Website: https://wrenwallet.app
