# Using Bivium — a walkthrough

Bivium is **non-recourse, fixed-rate, no-liquidation** lending. You post BTC or ETH as collateral, draw
USDC, and may repay strictly before maturity to reclaim the collateral. From maturity onward repayment
is closed; unpaid debt contributes collateral to market settlement. There is no liquidation engine or
margin call.

The current **[Development Preview](https://dev.bivium.pages.dev)** has two experiences over the same
configured markets and signed orders:

- **Basic** provides guided **Markets** and **Portfolio** screens.
- **Pro** provides a single **Trade** workspace with the book, chart, order ticket, and positions tape.

This guide documents only features enabled in the Development Preview. For the underlying mechanics,
see the [protocol overview](protocol-overview.md).

---

## 0. Verify the Preview, then connect

1. Open the **[Bivium Development Preview](https://dev.bivium.pages.dev)**.
2. **Check the Header before approving or signing.** Its deployment identity shows the environment,
   chain, shortened Bivium core address, and release digest prefix. Confirm they match the environment
   you intended to use. A familiar token pair or market name is not enough: every market and offer
   belongs to one exact chain and core.
3. **Choose Basic or Pro**, connect your wallet, and confirm that its network matches the displayed
   chain.
4. Use only test assets supplied for the selected Development environment.

The Preview is a test frontend for an unaudited release. It is not production, this guide publishes no
contract addresses, and it must not be used with real funds.

---

## 1. Basic — Markets

Open **Markets** to compare the series listed by the current deployment configuration. Each series is
identified by its loan token, collateral token, strike, and maturity. Select a market to open its action
dock:

- **Borrow** fills the best executable signed maker bids for that exact market. Review the fixed APR,
  proceeds, face owed, collateral, maturity, and strike before approving collateral and filling.
- **Lend now** buys resting DCN credit from signed asks. The APR locks when the fill lands; you may sell
  the DCN in Pro before maturity or hold it for repay-or-deliver settlement.
- **Rate order** appears only when the RFQ/intent lane is configured. It lets a borrower name acceptable
  terms for solvers to fill; signing an intent is not itself an executed loan.

The signed order book is the default liquidity source, with RFQ/intents available only when configured.
Before submitting an action, re-check the Header domain and full market terms. Repay the fixed face while
`block.timestamp < maturity` to free collateral. At and after maturity, repayment is closed.

---

## 2. Basic — Portfolio

**Portfolio** is a unified tape of the connected account's enabled commitments and activity:

- **Loan** rows show debt, locked or reclaimable collateral, lifecycle state, and the available Repay or
  Reclaim action.
- **DCN** rows show fungible market credit and expose Claim from maturity onward for a pro-rata share of
  loan tokens from repaid debt and collateral from unpaid debt.
- **Order** rows show the remaining size of signed bids or asks after on-chain consumption and allow
  cancellation.

Portfolio reads the bounded list of markets configured for the Preview. This lets it inspect known
markets even when automatic discovery is unavailable, while preventing data from an unvalidated market
domain from being treated as safe.

The summary reports debt or delivered face, collateral locked, and resting orders. If the relayer is
unavailable, the order count may be shown as `unknown`. If required market configuration or domain
validation fails, Portfolio shows an unavailable state and Retry action. An unavailable view is **not**
an empty account: retry and independently verify on-chain state before concluding that a commitment is
absent.

### Claiming matured DCN

When a DCN can be claimed, its action area shows **Estimated proceeds** before the Claim transaction:

- the loan-token row estimates the holder's share contributed by debt repaid before maturity;
- the collateral-token row estimates the holder's share contributed by debt left unpaid at maturity.

The preview is read from current contract state and refreshes while the page is open. It is an estimate,
not a guaranteed execution result; the transaction uses the state available when it executes. If the
estimate cannot be loaded, do not infer that the claim is worth zero. Retry or verify the claim directly
before submitting.

---

## 3. Pro — Trade

Switch to **Pro** for the exchange workspace. Choose the exact configured market, then use the unified
signed book to take liquidity or rest a limit order. Bids and asks use one tick grid: `tick` encodes the
price and APR is derived for display. Pro also includes the enabled Loan, DCN, and Order position tape
used by Portfolio.

A relayer can hide, delay, or mis-rank discovery, but it does not hold user funds. The core re-checks
the offer, market domain, capacity, and other execution conditions on-chain.

---

## Migration and legacy positions

Legacy signatures are not converted into new offers. Positions created on an older core remain on that
core and require a compatible legacy interface until completion. Do not assume a position migrated
because another app release shows the same assets, strike, and maturity.

## Key terms

- **Floor / strike** — the price at which collateral is delivered or assigned.
- **Maturity** — the fixed cutoff. Repayment and new borrowing require `block.timestamp < maturity`;
  claims apply from `block.timestamp >= maturity`.
- **Rate / APR / tick** — the fixed borrowing cost or lending yield. The offer carries a `tick`, not a
  free-form price field.
- **DCN** — fungible market credit that claims a pro-rata share of the market's two-token settlement.
- **No liquidation** — there is no maintenance margin or liquidator; settlement occurs at maturity.

## Learn more

- **[Protocol overview](protocol-overview.md)** — markets, offers, and repay-or-deliver settlement.
- **[Security guide](security.md)** — domain checks, trust model, lender risk, and token assumptions.

> Development Preview and unaudited software. Verify the Header and do not use real funds.
