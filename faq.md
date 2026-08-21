# Frequently asked questions

This FAQ describes the current Bivium Development Preview and the domain-bound core lifecycle. For
button-by-button instructions, see [Using the app](using-the-app.md).

## 1. Does Bivium use LTV, margin calls, or price-triggered liquidation?

No. Bivium does not read a price oracle, calculate a maintenance LTV, issue margin calls, or sell
collateral because its market price changed. A borrower either repays strictly before maturity or leaves
the debt unpaid. Unpaid collateral is then delivered into the market's physical settlement for DCN
holders.

This removes price-triggered liquidation; it does not remove collateral, smart-contract, wrapper, or
market risk. See [Security](security.md).

## 2. When can a borrower repay?

Repayment is permitted only while `block.timestamp < maturity`. At and after maturity, repayment is
closed. Check the market maturity and the current Loan action before opening a wallet confirmation.

## 3. What happens when debt is unpaid at maturity?

The borrower keeps the loan proceeds and does not recover the collateral securing the unpaid debt. That
collateral participates in repay-or-deliver settlement for fungible credit holders in the same market.
There is no auction or oracle-priced sale inside the core.

## 4. Is the displayed APR a service fee or a guaranteed return?

Neither. Price is encoded by the offer's `tick`; the app derives APR from that price and the remaining
term. It is a comparison display, not a protocol-set fee, forecast, or guaranteed yield. The realized
economic result also depends on execution price, whether debt is repaid, collateral received, and the
value and liquidity of the assets.

## 5. What is DCN, and where is it fungible?

A dual-currency note (DCN) is Bivium market credit. It is fungible only within one exact market identity:
chain, Bivium core, loan token, collateral token, maturity, strike, partial-repayment policy, and gate.
A similar symbol, strike, or maturity on another chain or core is a different claim.

## 6. Can DCN be transferred or traded after maturity?

Existing credit can be transferred after maturity when the submitted action, offer timing, available
credit, authorization, and other execution conditions allow. Whether a buyer, seller, or executable
quote exists is a separate liquidity question. The app does not promise a post-maturity exit.

## 7. Why can Claim return both loan token and collateral token?

Debt repaid before maturity contributes loan token to settlement. Debt left unpaid contributes its
collateral token. A DCN holder claims a pro-rata share of both settlement assets, so either amount may be
zero or positive depending on the market's final state.

## 8. Are Estimated proceeds guaranteed?

No. Estimated proceeds are current-state previews that refresh while the app is open. The Claim
transaction uses execution-time state. If the preview cannot be loaded, treat the proceeds as unknown,
not zero.

## 9. Why does Portfolio unavailable not mean my account is empty?

Portfolio displays positions only when it can safely resolve configured markets and validate their
domains. A configuration, network, or validation failure blocks that conclusion. Retry, verify the Header
and wallet network, and independently check the intended core before assuming a commitment is absent.

## 10. What can a relayer failure affect?

The relayer affects off-chain offer publication and discovery. Failure can hide, delay, or mis-rank
orders and may prevent the app from confirming that a signed order was delisted. It does not custody core
funds or bypass on-chain offer, domain, capacity, authorization, and lifecycle checks.

## 11. Which collateral assets and markets are supported?

Use only the markets listed by the current Development Preview configuration. Each entry defines the
exact loan token, collateral token, maturity, strike, and remaining domain fields. A token symbol alone
does not establish support, and the list can change between test releases. Represented BTC or ETH also
carries external issuer, bridge, redemption, basis, and liquidity assumptions.

## 12. Is partial repayment supported?

It depends on the exact market's `allowPartialRepay` parameter. Do not infer the policy from another
market or release. Review the selected market terms and the amount presented by the app before approving
loan token or submitting Repay.

## 13. Can I use real funds in the Development Preview?

No. The Preview is an unaudited, non-production test environment. Use Sepolia test assets only, verify
the Header domain, and confirm every wallet transaction yourself. No contract address published by an
unverified source should be treated as authoritative.

Mock GHO and Mock USDC in the multi-loan Sepolia candidate are separate, valueless test assets. The
borrowed token address is part of the market identity, so repayment must use that original loan token;
matching dollar-like names or symbols are not interchangeable.

## 14. What is a whole-vault market, and why can't I type an amount?

Its collateral is an indivisible Bitcoin vault, so the vault's size fixes the loan's face rather than the
other way round. You select vaults; the panel derives the face. A single borrow also fills exactly one
lender quote rather than sweeping the book, so a vault whose face exceeds the best resting bid cannot be
borrowed against. See [Bitcoin vault markets](bitcoin-vault-markets.md).

## 15. What is the difference between vaultBTC and TBVBTC?

`vaultBTC` represents one specific vault and is **not transferable** — it is an internal credential that
moves only along the vault lifecycle's own lanes, and its holder can reclaim that exact vault. `TBVBTC`
is an ordinary transferable ERC-20 representing a share of the pool of vaults on the keeper side; it is
what the collateral leg of settlement pays, what its own market takes as collateral, and what the
redemption book escrows.

Converting a vault to TBVBTC exchanges a claim on *your* vault for a claim on the pool. While no keeper
has settled the vault, the original depositor can reverse it with an equal amount of TBVBTC — including
for a vault delivered by default.

## 16. If I hold TBVBTC, am I guaranteed native bitcoin?

No. What the protocol can ensure is that the **claim** does not disappear: escrowed redemption orders
remain backed and are cancellable after their deadline, the token stays transferable and usable as
collateral, and the vault's original depositor can buy it back while it is unsettled. Converting that
claim into bitcoin requires a keeper to front-pay and, on mainnet, payment verification. In the
Development Preview no bitcoin moves at all and fills are simulated. See [Security](security.md).

## 17. Are offers and positions from an older core migrated?

No. The chain-and-core-bound format does not convert legacy signatures or move positions between core
deployments. Older positions remain associated with their original core and require a compatible legacy
read-and-exit path until their lifecycle completes.

### Learn more

- **[Using the app](using-the-app.md)** — Borrow, Lend, Portfolio, Pro, Repay, and Claim steps.
- **[Bitcoin vault markets](bitcoin-vault-markets.md)** — whole-vault collateral and the two tokens.
- **[Protocol overview](protocol-overview.md)** — market identity, pricing, and settlement mechanics.
- **[Security](security.md)** — trust boundaries and asset assumptions.
