# From an option-backed index to fixed-rate lending

Bivium can be understood as a restricted, lending-oriented form of the **P/N decomposition** used
to describe an option-backed index. The decomposition is useful for explaining who bears the market
risk, but Bivium deliberately does not expose both legs as freely traded instruments.

The resulting product is simpler: lenders hold the protected credit leg, while a borrower keeps the
right to repay a fixed amount and recover their collateral. To the user, this is fixed-rate,
fixed-maturity, non-recourse borrowing rather than an options trading interface.

> This page gives an economic interpretation of the current protocol and then describes a possible
> Aave v4 integration. It does not introduce a new Core interface, token, oracle, or settlement rule.

## The P/N decomposition

For a collateral asset `X`, strike `K`, and maturity `T`, the conceptual split is:

```text
X = P + N

P(T) = min(X(T), K)
N(T) = max(X(T) - K, 0)
```

- **P** is the protected leg. Its upside is capped at `K`, while it remains exposed to the collateral
  falling below `K`.
- **N** is the residual upside and redemption leg. It represents the value above `K` and the economic
  right to recover the collateral by paying the fixed redemption amount.

In a full option-backed index design, multiple P and N legs could be independently held, traded, and
aggregated across assets, strikes, and maturities. Bivium narrows that design to one lending market and
restricts what happens to N.

## Bivium's simplifying restriction

When a borrower posts BTC or ETH and raises stablecoin funding:

- the lender side acquires the **P-like fungible market credit**;
- the borrower retains the **N-like right to repay before the maturity cutoff and reclaim collateral**;
- the N-like right is not sold into a separate public market;
- if the borrower does not repay before the cutoff, that right expires and the collateral enters the
  market's pooled settlement basket.

N is therefore best understood as a **redemption credential**, not as a second freely circulating
asset. In the current Core it is not a separately minted token. It is represented by ownership and
control of the borrower's negative market position together with the protocol's repayment rule.

This restriction turns the P/N decomposition into a lending product:

| Option interpretation | Bivium lending interpretation |
|---|---|
| Deposit collateral `X` | Post BTC or ETH collateral |
| Sell the P leg | Issue and sell fixed-maturity credit |
| Retain the N leg | Keep the right to repay and recover collateral |
| Exercise the redemption right | Repay the fixed debt before maturity |
| Let N expire | Do not repay; deliver collateral into settlement |

## Fixed rate from the traded discount

Suppose a borrower issues `u` units of credit and sells them for price `p` loan tokens per unit. The
borrower receives `p * u` today and must contribute `u` loan tokens to recover the collateral before
maturity.

```text
cash received today = p * u
fixed repayment      = u
term return          = 1 / p - 1
```

For example, selling `80,000` units at `p = 0.875` gives the borrower `70,000` stablecoins today. The
amount required to close the debt is fixed at `80,000` stablecoins. The lender's maximum contractual
stablecoin proceeds are also `80,000`, subject to the market's actual pooled settlement.

The rate is fixed by the trade price. It is not set by utilization, an interest-rate model, or an
oracle. The quoted discount compensates lenders for time, liquidity, and the possibility of receiving
collateral instead of the loan token.

## Repay or deliver

Bivium does not continuously compare collateral value with debt value. There is no health factor,
margin call, or price-triggered liquidation.

Before maturity, the borrower may repay the fixed debt and recover the posted collateral. From
maturity onward, repayment is closed. Repaid positions contribute loan tokens to settlement; unpaid
positions contribute collateral. Credit holders claim a pro-rata share of that pooled two-token
basket.

Consequently:

- the borrower has no obligation beyond the posted collateral;
- the lender has capped loan-token upside and accepts collateral-delivery risk;
- a fall in BTC or ETH is an economic loss borne by the credit side, not a Core insolvency detected by
  an oracle;
- the absence of liquidation follows from physical, dual-currency settlement.

This is economically similar to **fixed-rate, fixed-term, non-recourse collateralized lending with an
embedded option**, while remaining mechanically a fungible market-credit protocol rather than a set
of bilateral loans.

## What P is—and is not

In this interpretation, P is the protected side of the collateral decomposition and maps most closely
to Bivium market credit. It is not, by itself:

- a stablecoin;
- a guaranteed-yield instrument;
- a claim guaranteed to settle entirely in the loan token;
- a perpetual asset independent of collateral, strike, and maturity.

A strategy may buy P-like credit with GHO and present the result as a GHO-denominated yield product.
That yield is an upper-layer strategy outcome, not the definition of P. If collateral is delivered
below the strike, the strategy can lose GHO-denominated value.

## Why the model fits Aave

The simplified model follows a recognizable Aave product path:

```text
borrower posts BTC or ETH
        -> receives stablecoin funding at a fixed price
        -> keeps the right to repay a fixed amount
        -> recovers collateral or delivers it at maturity
```

The user-facing concepts remain collateral, principal, fixed rate, maturity, repayment, and
redemption. P/N remains an internal economic model that explains the risk transfer.

An Aave v4 integration can therefore be framed as a proposed **Fixed-Term Option Lending Spoke**, not
as the issuance of a new stablecoin or as a generic option-backed index market.

### Proposed responsibilities of the Spoke

The Spoke would provide the Aave-facing market and account layer while preserving Bivium's settlement
semantics:

- register approved collateral, loan-token, strike, and maturity combinations;
- escrow collateral outside shared Hub liquidity by default;
- route lender capital into fills only when a matching fixed-rate offer executes;
- record the borrower's redemption authority in their account;
- expose fixed repayment and rollover actions;
- settle unpaid positions by delivery rather than by price-triggered liquidation;
- isolate each market's exposure through caps and explicit collateral assumptions.

The proposed Spoke must not represent the borrower's position as ordinary Aave variable debt. Ordinary
Hub debt is an unconditional obligation to restore the borrowed asset, whereas the Bivium borrower may
choose not to repay and surrender the collateral instead.

### GHO's role

GHO is a natural loan, quote, and settlement token for such a Spoke:

- lenders pay GHO to acquire P-like credit;
- borrowers receive GHO when issuing credit;
- borrowers use a fixed amount of GHO to recover collateral;
- rollover can close one maturity and fund the next in a single account workflow.

This does not make P a form of GHO. It makes GHO the cash leg of a BTC- or ETH-backed fixed-term credit
market.

### Safe liquidity boundary

The clean initial integration is maker-funded rather than Hub-debt-funded. A lender may keep idle GHO
productive in Aave and authorize it to be pulled atomically when an offer fills, but the Bivium Spoke
does not call `Hub.draw` to fund the borrower.

That distinction is essential:

```text
Hub debt       = an unconditional obligation to restore GHO
Bivium credit  = a claim that may settle in GHO, collateral, or both
```

Using a shared Hub credit line would transfer BTC or ETH tail risk to Hub liquidity providers unless a
separate first-loss layer guaranteed restoration in GHO. Such a credit line is a possible later risk
product, not a requirement for the fixed-term Spoke and not part of the current protocol.

## Product summary

Bivium applies one decisive restriction to the broader P/N model: **P can circulate, while N remains
with the borrower as the redemption credential**. That restriction converts an option decomposition
into an accessible lending flow:

- lenders buy fixed-maturity, collateral-deliverable credit;
- borrowers obtain fixed-rate, non-recourse financing;
- rates are discovered through discounted credit trading;
- settlement is repay-or-deliver;
- no price oracle or mid-term liquidation is required by the Core.

This is the conceptual bridge from an option-backed index primitive to an Aave-aligned fixed-rate
lending product.
