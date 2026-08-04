# Bivium

**Non-recourse, fixed-rate, no-liquidation lending.**

Bivium lets you borrow stablecoins against BTC or ETH at a fixed rate, for a fixed term, with **no
liquidation and no margin calls**. You may repay strictly before maturity and reclaim your collateral.
From maturity onward repayment is closed; unpaid debt contributes its collateral to the market's pooled
settlement basket. The protocol is **immutable and oracle-free**: settlement reads only whether the loan
was repaid, never a price feed.

The same market has two sides:

- **Borrowers** post BTC/ETH and draw USDC. They get downside-protected financing that can never be
  liquidated.
- **Lenders** supply USDC and hold fungible market credit. At maturity, holders claim a pro-rata share of
  loan tokens contributed by repaid debt and collateral contributed by unpaid debt.

## Where to start

- **[Using the app](using-the-app.md)** — a walkthrough of Basic Markets and Portfolio, plus the Pro
  Trade workspace.
- **[Protocol overview](protocol-overview.md)** — how the core works: the offer model, repay-or-deliver
  settlement, why there's no oracle and no liquidation, and how rates are set.
- **[Security](security.md)** — trust model, what to understand before lending, and token assumptions.

> Bivium is an unaudited proof of concept. The fresh chain-and-core-bound release described here is
> pre-release; no addresses for it are announced here. Do not use it with real funds.
