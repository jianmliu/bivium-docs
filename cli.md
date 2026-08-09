# Command-line interface

The Bivium CLI (`bivium`) is a command-line client and TypeScript SDK covering the protocol's full
user surface: market identity and state, maker funding and offer signing, borrowing, repayment,
collateral reclaim, settlement claims, and secondary trading of market credit (DCN). It exists for
developers, market operators, and automated acceptance runs — everything the web app does at the
protocol layer, scriptable from a terminal.

The CLI shares the protocol's status: it targets test deployments only, is unaudited, and must not
be used with real funds. It is a client, not a service — it holds no funds, runs no server, and
signs only with a key you provide locally.

## Deployment profiles

Every invocation is scoped to a **profile file** that pins the deployment it is allowed to talk to:
chain id, core contract, signature ratifier, RPC endpoint, an ABI-lineage tag, and an allowlist of
token symbols with their exact decimals. Profiles are how the CLI avoids the most dangerous failure
mode in a domain-bound protocol: signing or submitting against the wrong chain, the wrong core, or a
core whose encoding differs from what the client assumes.

Before any state-changing command runs, the CLI calls the core's `computeId` with a fixed canary
tuple and requires the result to match its own local hash for the profile's declared lineage. A
mismatch — wrong core, wrong lineage, wrong chain — stops the command before any transaction is
signed. Reads and writes never fall back to guessed parameters; a profile that cannot be verified is
an error, not a warning.

Profile files carry deployment addresses; this documentation does not. Obtain or construct the
profile for a given test deployment from that deployment's own release materials.

## Command groups

| Group | Commands | What it covers |
|---|---|---|
| Market | `market id`, `market state` | Compute a market id locally and cross-check it on-chain; read the market's aggregate state |
| Reads | `read position`, `read credit`, `read liquidity` | A single account's debt/collateral, DCN balance, and resting lender liquidity |
| Maker | `maker set-ratifier`, `maker fund`, `maker withdraw-liquidity`, `maker make-offer` | Register the quote authority, escrow lender liquidity, and sign resting offers (limit orders) |
| Borrow | `borrow quote`, `borrow execute` | Preview exact principal, collateral, and implied APR for a resting bid, then fill it |
| Lifecycle | `repay`, `reclaim`, `claim` | Repay strictly before maturity, withdraw released collateral, claim the settlement basket from maturity onward |
| Trading | `book list`, `trade buy`, `trade sell`, `order list`, `order cancel` | The DCN secondary market: aggregated depth, market-order sweeps, and order management |
| Testnet | `mock mint` | Mint valueless mock test assets — only for tokens the profile explicitly marks mintable |

A signed offer travels as a small JSON file: the full offer fields, the commitment hash, and the
maker's EIP-712 ratification signature. Any consumer of such a file recomputes the commitment from
the fields and rejects the file on mismatch, and checks its chain and core against the active
profile. The file is portable; the chain remains the only authority on whether it can execute.

## Trading semantics

- **Limit orders** are signed resting offers (`maker make-offer`). They cost no gas to create and
  are cancelled on-chain by exhausting the offer's consumption budget (`order cancel`); delisting
  from any order-book relayer is best-effort on top of that on-chain authority.
- **Market orders** (`trade buy`, `trade sell`) sweep one or more resting offers in a single atomic
  transaction. The CLI plans the sweep with the same rounding the core applies, shows the plan
  (per-offer fills, total cost or proceeds, worst tick), and after execution verifies that the
  credit and cash balance changes equal the plan exactly. A tick limit bounds the worst acceptable
  price.
- **Buying DCN** fills a resting ask and can only transfer credit the ask's maker actually holds;
  it never originates new debt. **Selling DCN** fills a resting bid from credit you hold. As
  described in the [FAQ](faq.md), transfers of existing credit may remain possible at or after
  maturity, but nothing guarantees quotes, counterparties, or an exit at any time.
- **Order books are discovery, not truth.** The CLI can read and publish offers through the same
  relayer wire protocol the web app uses, or exchange signed-offer files directly. Remaining
  capacity always comes from on-chain state, every relayer response is re-verified field by field,
  and a relayer failure is reported as exactly that — never rendered as an empty book.

## What the CLI enforces

The same discipline the protocol documents elsewhere, applied client-side:

- **Exact integer arithmetic only.** Strikes, prices, and amounts never pass through floating
  point; inputs that need more precision than a token has are rejected rather than rounded.
- **Core rounding is mirrored, not approximated**, and pinned by tests against values returned by
  live deployments.
- **Simulation before every transaction**, so protocol errors surface by name before gas is spent.
- **Exact allowances** — approvals cover the precise amount of one action, never unlimited.
- **Balance-delta postconditions** — after a fill, funding, or sweep, the CLI verifies the actual
  token movements equal the plan, and treats any difference as a failure.
- **Ratification prechecks** — an offer is only written or accepted if the on-chain ratifier
  confirms the signature for its commitment.
- **Keys stay local**: signing keys are read from an environment variable, are never accepted as
  command-line arguments, and are never logged.

## What the CLI does not do

It does not custody funds, guarantee liquidity, or relax any protocol condition: repayment still
closes at maturity, displayed APR remains a derived display value and not a promised return, and a
market with no executable offers is just as unavailable from the CLI as from the app. Commands that
would publish to an order-book relayer refuse to run when the active profile's lineage does not
match what that relayer speaks.

## Relationship to the web app

The CLI and the web app are alternative clients over the same protocol. They share the relayer wire
protocol, so against the same deployment domain — same chain, core, and ratifier — an offer
published from one is visible to and fillable from the other. Settlement truth is always the core
contract; neither client can see or produce an order the other could not verify on-chain.

An experimental command group for whole-vault (ERC-1155) collateral exists for local development
against in-progress contracts. It is not part of the Development Preview and is not documented here
beyond this note.
