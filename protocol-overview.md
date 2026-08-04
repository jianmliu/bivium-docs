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

A credit claim is **not** a claim paired to one borrower. Credit is fungible within an exact market and
represents a pooled settlement claim. A borrower may repay only while `block.timestamp < maturity`,
contributing loan tokens and freeing their collateral. From `block.timestamp >= maturity`, repayment is
closed and claims are available: unpaid debt contributes its collateral, while debt repaid before the
cutoff contributes loan tokens. Each credit holder receives a pro-rata share of that two-token basket.

This is why Bivium needs **no oracle and no liquidation**. The collateral is not a margin buffer to be
sold against a price feed on the way down; it is simply the *second settlement currency*, delivered as-is
if the borrower walks away. "No liquidation" and "dual-currency settlement" are the same design fact.
Nothing revalues or liquidates a position mid-term. The maturity cutoff fixes which assets contribute
to the pooled claim without consulting a price.

## The economics: option-like exposure

The payoff has **physically-settled put-like** economics:

- The **borrower has long-put-like protection** — they can repay before the cutoff to recover collateral
  or remain unpaid and contribute that collateral to settlement. That is why they cannot be liquidated.
- **Credit holders have short-put-like exposure** — their fungible claims may settle partly in loan
  tokens and partly in a pro-rata share of collateral from all unpaid debt in the market.

You don't have to think in options to use Bivium, but it helps explain the assignment exposure. The
quoted rate is not pure option premium: it can include time value, compensation for assignment/default
risk, liquidity conditions, and maker spread. A higher displayed rate alone does not establish a
monotonic probability of collateral delivery.

## Markets

A market's identity is exactly eight fields: `chainId`, `bivium` (the core contract address),
`loanToken`, `collateralToken`, `maturity`, `strike`, `allowPartialRepay`, and `gate`. The first two
bind the market to one chain and one core deployment. Claims are fungible only *within that exact
identity*, so concentrating activity on a standard set of maturities and strikes builds depth without
making positions portable across chains or cores.

The complete flat `Offer` is `chainId`, `bivium`, `loanToken`, `collateralToken`, `maturity`, `strike`,
`allowPartialRepay`, `gate`, `maker`, `buy`, `tick`, `maxUnits`, `maxAssets`, `start`, `expiry`, `group`,
and `ratifier`, in that order. Its eight-field prefix is the market identity. `tick` encodes the price
on the core's grid; there is no separate price field. The full tuple is always hashed as the ratification
commitment, so changing any field, including the chain or core, changes it. Signature-based ratifiers
verify a maker signature over that commitment; manager, curve, and other on-chain ratifiers can attest
their policy without a maker signature.

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

The core is **immutable** (no protocol administrator, pause, or upgrade), has **no oracle** (settlement reads
only repaid-or-not), and has **no liquidation engine**. Pricing is delegated to pluggable attestors that
can authorize economically consequential fills within their configured scope; user-authorized operators
also retain their granted capabilities. A borrower posts collateral once, at a fixed
strike; nothing revalues or liquidates it mid-term. See **[Security](security.md)** for the full trust
model, the lender-side risk you must understand, and the per-market token assumptions.
