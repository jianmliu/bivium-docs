# Protocol overview

Bivium is a fixed-rate, fixed-term credit protocol with one defining property: a loan settles
**repay-or-deliver**, with no liquidation and no oracle. This page explains how that works and why the
design follows from it.

## A loan is a trade

A position on a Bivium market is a single signed number: `position = credit − debt`.

- Holding **+credit** means you lent — you hold a claim that pays out at maturity (a zero-coupon-style
  claim).
- Holding **−debt** means you borrowed — you issued that claim and posted collateral to back it.

Every action is the same operation — move `units` of the claim for `units × price` of the loan token:

- **Lend** = buy the claim.
- **Borrow** = sell a claim you create, posting collateral.
- **Sell / buy on the secondary** = transfer an existing claim between holders.

Because of this, lending, borrowing, and secondary trading all run through one order type (an *offer*)
and one settlement call (a *fill*). The only thing that distinguishes borrowing is that the seller's
position crosses below zero into debt — which is the single point where collateral is escrowed.

## Repay-or-deliver settlement (no oracle, no liquidation)

A credit claim is **not** a plain fixed payout. At maturity the borrower chooses:

- **Repay** the loan token → reclaim the collateral; the lender is paid in the loan token, or
- **Deliver** → the collateral is handed to the credit holder instead.

A rational borrower repays only when the collateral is worth more than the amount owed, so the lender
receives `min(face, collateral)` — **physically settled, in kind, with no price feed.**

This is why Bivium needs **no oracle and no liquidation**. The collateral is not a margin buffer to be
sold against a price feed on the way down; it is simply the *second settlement currency*, delivered as-is
if the borrower walks away. "No liquidation" and "dual-currency settlement" are the same design fact.
Nothing happens mid-term regardless of price — every position settles exactly once, at maturity.

## The economics: a loan is an option

The payoff above is exactly a **physically-settled put option** on the collateral:

- The **borrower is long a put** — they have the right to "sell" their collateral to the lender at the
  agreed strike (by walking away) if it falls below that level. That is their downside protection, and
  why they can never be liquidated.
- The **lender is short that put** — they earn a premium (the loan's yield) for standing behind it, and
  may end up owning the collateral at the strike.

You don't have to think in options to use Bivium, but it explains the mechanics: the **rate** a borrower
pays is the option premium, and a higher rate means a higher chance the lender ends up holding the
collateral.

## Markets

A market's identity is exactly eight fields: `chainId`, `bivium` (the core contract address),
`loanToken`, `collateralToken`, `maturity`, `strike`, `allowPartialRepay`, and `gate`. The first two
bind the market to one chain and one core deployment. Claims are fungible only *within that exact
identity*, so concentrating activity on a standard set of maturities and strikes builds depth without
making positions portable across chains or cores.

The complete flat `Offer` is `chainId`, `bivium`, `loanToken`, `collateralToken`, `maturity`, `strike`,
`allowPartialRepay`, `gate`, `maker`, `buy`, `tick`, `maxUnits`, `maxAssets`, `start`, `expiry`, `group`,
and `ratifier`, in that order. Its eight-field prefix is the market identity. `tick` encodes the price
on the core's grid; there is no separate price field. The full offer is hashed and signed, so changing
any field, including the chain or core, changes its commitment.

Bivium markets focus on liquid, blue-chip collateral — **BTC and ETH** — because that collateral is deep
and the risk is straightforward to price and hedge. The settlement window aligns to the standard
08:00 UTC options expiry.

## How rates are set

Price is expressed on a **logistic tick grid** and shown to you as an APR. Each tick moves by a fixed
`ln(1.005)` step in log-odds space, giving roughly 0.5% relative resolution in the discount/rate near
par; the maximum tick is pinned to par. Every order, human or automated, lands on the same ladder, so
quotes are directly comparable.

How a given market's rate is *justified* is pluggable, without changing the core:

- **Signed quotes (RFQ / order book).** A lender (or a market-maker quoting on their behalf) signs an
  offer at a chosen rate; a borrower fills it.
- **An optional on-chain curve.** When explicitly configured and funded, a pool can quote a rate as a
  function of utilization. The stable app configuration disables this lane and uses signed CLOB offers,
  with RFQ/intents available when configured; a curve is not guaranteed liquidity.

In every case the rate is one number — read as a tick, an APR, or an upfront premium — and the immutable
core simply checks that a fill respects it. The core never sets prices and holds no special pricing power.

## Releases and compatibility

The chain-and-core-bound format is an ABI-breaking release: it requires a fresh core, frontend, and
keeper deployment. Legacy offers and signatures are not migrated and are not accepted as the new
`Offer` encoding. A public check against an old ratifier can still return `RATIFIED` for the exact old
commitment; that only describes the old signature and is not evidence that the offer can execute on
the current core. A missing or wrong domain prevents new-core execution.

Positions also do not migrate between cores. A legacy read-and-exit path must remain available until
old positions settle; this documentation does not claim that such a UI is live. A legacy Sepolia
deployment may exist, but the fresh domain-bound release described here has not been deployed or
promoted to production.

## Trust model in one paragraph

The core is **immutable** (no owner, no admin, no pause, no upgrade), has **no oracle** (settlement reads
only repaid-or-not), and has **no liquidation engine**. Pricing is delegated to pluggable attestors that
can validate an offer but can never move a user's funds. A borrower posts collateral once, at a fixed
strike; nothing revalues or liquidates it mid-term. See **[Security](security.md)** for the full trust
model, the lender-side risk you must understand, and the per-market token assumptions.
