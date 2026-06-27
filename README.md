# Bivium

**Non-recourse, fixed-rate, no-liquidation lending.**

Bivium lets you borrow stablecoins against BTC or ETH at a fixed rate, for a fixed term, with **no
liquidation and no margin calls**. At maturity you either repay and reclaim your collateral, or walk
away — deliver the collateral at a price you chose up front, and keep the borrowed funds. The protocol
is **immutable and oracle-free**: settlement reads only whether the loan was repaid, never a price feed.

The same market has two sides:

- **Borrowers** post BTC/ETH and draw USDC. They get downside-protected financing that can never be
  liquidated.
- **Lenders** supply USDC and earn a fixed yield — the premium a borrower pays for that protection. If a
  borrower walks away, the lender receives the collateral instead of repayment.

## Where to start

- **[Using the app](using-the-app.md)** — a step-by-step walkthrough of Borrow, Earn, and Positions.
- **[Protocol overview](protocol-overview.md)** — how the core works: the offer model, repay-or-deliver
  settlement, why there's no oracle and no liquidation, and how rates are set.
- **[Security](security.md)** — trust model, what to understand before lending, and token assumptions.

> Bivium is currently a testnet preview and is unaudited. Do not use it with real funds.
