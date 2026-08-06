# Using Bivium — step by step

Bivium is fixed-rate, fixed-term, non-recourse lending with no price-triggered liquidation. This guide
covers only workflows enabled in the current **[Development Preview](https://dev.bivium.pages.dev)**.
An active market and an executable quote are required for Borrow or Lend; appearing in the market list
does not guarantee liquidity.

For lifecycle questions, see the [FAQ](faq.md). For the underlying mechanics, see the
[protocol overview](protocol-overview.md).

---

## 1. Before you start

You need a wallet connected to the Sepolia test network and the test assets required by the selected
market. Use test assets only.

1. Open the **[Bivium Development Preview](https://dev.bivium.pages.dev)** in a browser that supports
   your wallet.
2. Check the Header before approving or signing. It shows the environment, chain, shortened Bivium core
   address, and release digest prefix.
3. Confirm the wallet network matches the displayed chain.
4. Confirm the full market terms before every action. A familiar asset pair is not enough: a market and
   its offers belong to one exact chain and core.

The Preview is unaudited and non-production. This guide publishes no contract addresses and must not be
used as authority for a deployment address.

## 2. Understand the market terms

Each listed market has a specific set of terms:

- **Loan token** — the asset the borrower receives and may use to repay, such as test USDC.
- **Collateral token** — the represented BTC or ETH asset posted by the borrower. Its wrapper or issuer
  has risks separate from Bivium.
- **Strike / floor** — the fixed conversion boundary used to determine required collateral and physical
  delivery exposure.
- **Maturity** — the cutoff. New borrowing and repayment require a pre-maturity state; claims apply from
  maturity onward.
- **Face** — the fixed loan-token amount owed by the borrower.
- **APR** — a display derived from the offer price and remaining term. It is not a protocol-set service
  fee, a promise of liquidity, or a guaranteed return.
- **Dual-currency note (DCN)** — fungible credit within that exact market domain. At maturity it claims a
  pro-rata share of loan token from repaid debt and collateral token from unpaid debt.

## 3. Borrow

Borrow is available only when the selected market is active and has an executable bid.

1. Open **Basic → Markets** and select the exact strike and maturity you intend to use.
2. Choose **Borrow** and enter the desired amount.
3. Review the fixed APR, proceeds, face owed, required collateral, strike, and maturity.
4. If requested, approve only the collateral amount needed for the action.
5. Submit **Borrow** and review the wallet transaction before confirming it.
6. After confirmation, open **Portfolio** and verify the new **Loan** row.

If the market is matured, has no executable quote, or fails domain validation, do not submit an
alternative transaction based only on a matching asset symbol.

## 4. View and manage a Loan

In **Portfolio**, a Loan row shows:

- debt or delivered face;
- locked or reclaimable collateral;
- strike and maturity;
- current lifecycle state;
- the contextual action currently allowed.

Portfolio uses the bounded market list configured for the Preview. If required configuration cannot be
validated, it shows an unavailable state rather than treating the account as empty.

## 5. Repay and reclaim collateral

Repayment is allowed strictly before maturity. At and after maturity, repayment is closed.

1. Open the Loan row in **Portfolio** and confirm it is still repayable.
2. If requested, approve the required loan token.
3. Select **Repay**, review the fixed face, and confirm the transaction in the wallet.
4. When collateral is shown as reclaimable and **Reclaim** is available, submit that separate action.
5. Verify the updated Loan state after confirmation.

Do not assume that the availability of a button remains unchanged while a wallet confirmation is open;
the app checks current chain time and lifecycle conditions again before submission where applicable.

## 6. Lend and receive DCN

Lend now is available only when an active market has an executable ask.

1. Open **Basic → Markets** and select the exact market.
2. Choose **Lend now** and enter the desired amount.
3. Review the offer price, displayed APR, term, strike, and DCN face.
4. Approve the loan token if requested, then submit the fill and confirm it in the wallet.
5. Open **Portfolio** and verify the **DCN** row.

DCN is a physically settled market claim, not a savings balance or guaranteed-yield product. Holding it
can result in receiving collateral token at maturity.

## 7. Trade and manage Orders in Pro

Switch to **Pro** for the order book, chart, order ticket, and shared positions tape.

1. Select the exact configured market.
2. Take available liquidity or place an order when the corresponding order control is enabled.
3. Review `tick`-derived price and APR, size, side, expiry, and market before signing.
4. Check the **Order** row in the positions tape or Portfolio.
5. Use **Cancel** to consume the remaining on-chain capacity of a resting order; the app also attempts to
   remove it from relayer discovery.

Existing credit may remain transferable after maturity when offer timing, available credit, and all
other execution conditions allow. This does not guarantee a quote, counterparty, or post-maturity exit.

## 8. Claim matured DCN

From maturity onward, a DCN row may expose **Claim**.

1. Review **Estimated proceeds** in the action area.
2. Check the loan-token estimate, which represents the holder's share of debt repaid before maturity.
3. Check the collateral-token estimate, which represents the holder's share of collateral delivered by
   unpaid debt.
4. Submit **Claim** and confirm the transaction in the wallet.
5. Verify the received assets and updated DCN balance after confirmation.

Estimated proceeds use current contract state and refresh while the page is open. They are not a
guarantee: final amounts use the state when the transaction executes. If the preview cannot be loaded,
treat the result as unknown rather than zero.

## 9. Understand unavailable states

- **Matured market** — new borrowing is closed. Claims may be available to DCN holders.
- **No executable quote** — the market may be configured but currently has no fillable bid or ask.
- **Relayer unavailable** — order discovery, ranking, or cancellation delisting may be delayed. The
  relayer does not hold core funds, and the core still checks execution conditions.
- **Portfolio unavailable** — required market configuration or domain validation failed. Retry and
  independently verify on-chain state; this is not evidence that the account has no commitments.
- **Resting orders unknown** — if relayer reads fail, do not infer that a signed order disappeared.

## 10. Continue learning

- **[FAQ](faq.md)** — repayment, DCN, claims, data availability, and release limitations.
- **[Protocol overview](protocol-overview.md)** — market identity, offers, and repay-or-deliver settlement.
- **[Security](security.md)** — authorization, collateral, token, wrapper, and unaudited-code risks.

> Development Preview and unaudited software. Verify the Header, use test assets only, and confirm every
> wallet transaction yourself.
