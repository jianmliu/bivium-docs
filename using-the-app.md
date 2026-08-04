# Using Bivium — a walkthrough

Bivium is **non-recourse, fixed-rate, no-liquidation** lending. You post BTC or ETH as collateral, draw
USDC, and at maturity either **repay** and reclaim the collateral or **walk away** and deliver it. There
is no liquidation engine or margin call.

The app has two experiences over the same markets and orders:

- **Basic** provides guided **Markets** and **Portfolio** screens.
- **Pro** provides a single **Trade** workspace with the book, chart, order ticket, and a positions tape.

For how the protocol works, see the [protocol overview](protocol-overview.md).

---

## 0. Verify the release, then connect

1. **Check the Header before approving or signing.** Its deployment identity shows the environment,
   chain, shortened Bivium core address, and the first 12 characters of the release digest. Confirm all
   four match the release you intended to use. A familiar token pair or market name is not enough:
   every market and offer belongs to one exact chain and core.
2. **Choose Basic or Pro** in the Header, then connect your wallet and confirm its network matches the
   displayed chain.
3. **Use faucets only when the selected development environment provides them.** Follow the links and
   mock-token controls shown by that environment.

The domain-bound release described here is pre-release; this guide publishes no fresh-release
addresses and does not claim it has been promoted to production. A legacy Sepolia deployment may
still exist, but its offers, signatures, and positions are not part of the new core.

---

## 1. Basic — Markets

Open **Markets** to compare available loan-token/collateral series by strike and maturity. Select a
market to open its action dock:

- **Borrow** fills the best executable signed maker bids for that exact market. Review the fixed APR,
  proceeds, face owed, collateral, maturity, and strike before approving collateral and filling.
- **Lend now** buys resting DCN credit from signed asks. The APR locks when the fill lands; you may sell
  the DCN in Pro before maturity or hold it for repay-or-deliver settlement.
- **Rate order** appears only when the RFQ/intent lane is configured. It lets a borrower name acceptable
  terms for solvers to fill; signing an intent is not itself an executed loan.

The stable default is the signed CLOB, with RFQ/intents available only when configured. The guarded
curve/pool-manager lane is disabled by default. **Deposit** and LP subscription appear only in an
explicitly configured Development or experimental build; do not assume a pool bid, pool liquidity, or
LP deposit is available.

Before submitting any action, re-check that the Header domain is the intended one. At maturity a
borrower either repays the fixed face and reclaims collateral, or leaves the collateral for delivery.

---

## 2. Basic — Portfolio

**Portfolio** is one unified tape of the connected account's commitments and activity:

- **Loan** rows show debt, locked or reclaimable collateral, lifecycle state, and the available Repay
  or Reclaim action.
- **DCN** rows show directly held credit and expose Claim after maturity.
- **Order** rows show remaining signed bids or asks after on-chain consumption and allow cancellation.
- **LP** rows and their withdraw/request/redeem actions appear only when the optional pool surface is
  configured and the account actually holds pool shares.

The summary reports LP deposited, debt/delivered face, collateral locked, and resting orders. If the
relayer or market-domain validation is unavailable, treat missing order or position data as unknown,
not as evidence that a commitment disappeared.

---

## 3. Pro — Trade

Switch to **Pro** for the exchange workspace. Choose the exact market, then use the unified signed book
to take liquidity or rest a limit order. Bids and asks use one tick grid: `tick` encodes the price; APR
is a display derived from it. Pro also includes the same position tape used by Portfolio.

The pool's guarded bid is included only in a build that explicitly enables the pool lane and only when
it is funded and operational. Otherwise the executable book is signed CLOB/RFQ liquidity. A relayer can
hide or delay discovery, but the current core still checks the offer and market before execution.

---

## Migration and legacy positions

Legacy signatures are not converted into new offers. Positions created on an older core remain on that
core and need a legacy read-and-exit path until settlement; this guide does not claim that path is
already deployed. Do not assume a position migrated because another app release shows the same assets,
strike, and maturity.

## Key terms

- **Floor / strike** — the price at which collateral is delivered or assigned.
- **Maturity** — the fixed end date. Repayment and new borrowing occur before it; claims occur after it.
- **Rate / APR / tick** — the fixed borrowing cost or lending yield. The offer carries a `tick`, not a
  free-form price field.
- **DCN** — the direct credit claim that settles to the market's repay-or-deliver basket.
- **No liquidation** — there is no maintenance margin or liquidator; settlement occurs at maturity.

## Learn more

- **[Protocol overview](protocol-overview.md)** — markets, offers, and repay-or-deliver settlement.
- **[security guide](security.md)** — domain checks, trust model, lender risk, and token assumptions.

> Pre-release and unaudited. Verify the Header; do not use with real funds.
