# Bitcoin Collateral Needs Credit Markets — Article Design

## Goal

Add an original research article to the Bivium GitBook that explains, from Bivium's own design principles, why Bitcoin-backed lending benefits from fixed-term, market-priced, tradable credit claims.

The article is conceptual documentation, not a product announcement. It must help readers understand why Bivium uses its particular market structure without implying that disabled or future functionality is available.

## Editorial position

The article will make a Bivium-native argument. It will not cite, paraphrase closely, or structure itself around the Alpen or Bitcoin Magazine articles that informed the initial discussion. It will not position Bivium as an implementation of another protocol's architecture.

The central thesis is:

> Bitcoin can serve as high-quality collateral, but a scalable credit market also needs explicit term structure, standardized claims, market-based pricing, secondary transferability, and clearly bounded settlement risk.

Claims about lower funding costs, longer maturities, or deeper liquidity will be expressed as possible market outcomes when the required conditions exist, not as guarantees.

## Audience

The primary audience is a technically literate Bitcoin, DeFi, or credit-market reader who understands collateralized borrowing but may not yet distinguish loan origination from a functioning credit market.

The article should remain understandable without reading the smart contracts or using the Development Preview first. Links to the protocol overview, app guide, and security page will provide progressively deeper detail.

## Article structure

The new `bitcoin-credit-markets.md` page will use this argument sequence:

1. **Bitcoin is collateral; lending alone is not a market.** Establish the difference between originating an isolated loan and producing an asset that can be priced and transferred.
2. **Term structure must be explicit.** Explain why maturity creates comparable funding instruments and a clear repayment cutoff.
3. **Credit must be standardized.** Introduce DCN as fungible credit only within one exact market domain, rather than as a claim on a specific borrower.
4. **Origination and secondary trading share one model.** Show how Borrow, Lend, Buy, and Sell can use the same Offer and fill semantics.
5. **Markets, not the core, set the rate.** Explain tick-based signed quotes, price discovery, and the limits of displayed APR.
6. **Repay-or-deliver replaces liquidation.** Derive the no-oracle and no-liquidation properties from physical two-asset settlement at maturity.
7. **Risk changes form rather than disappearing.** Describe collateral delivery, collateral quality, assignment exposure, liquidity, token-wrapper, and smart-contract risk.
8. **Trust boundaries must be explicit.** Separate the immutable core from quote authority, relayer availability, market-domain validation, and collateral representation.
9. **Current availability.** Link the Development Preview and state that only its visible, enabled workflows are documented; it is unaudited and non-production.
10. **Conclusion.** Restate that Bivium's contribution is market structure for fixed-term Bitcoin-backed credit, not a promise of cheap or risk-free borrowing.

## Terminology

- Use **Bitcoin** for the asset and monetary network; use **BTC collateral** when discussing the represented asset posted in a market.
- Define **DCN** as fungible market credit that participates in repay-or-deliver settlement.
- Use **market** for the exact eight-field chain-and-core-bound identity.
- Use **settlement basket** only for the combined loan-token and delivered-collateral accounting at maturity.
- Avoid language suggesting that a claim is portable across markets, chains, or core deployments.

## Explicit exclusions

The article will not document or imply availability of:

- Pool or LP workflows;
- deposits, subscriptions, redemption queues, or manager settlement;
- vaults or curators;
- protocol-set utilization curves;
- oracle-triggered liquidations;
- structured products, securitization, rehypothecation, or borrowing against DCN;
- production contracts or addresses.

References to standardization and secondary trading apply only to enabled DCN and signed-offer behavior. They do not imply a future financing layer.

## Accuracy and safety boundaries

- Bivium is described as fixed-rate, fixed-term, non-recourse, oracle-free, and without forced liquidation.
- Repayment is allowed strictly before maturity; claims apply from maturity onward.
- Unpaid collateral is delivered into market-level settlement rather than sold using a price feed.
- DCN holders have short-put-like physical-delivery exposure and may receive a mixture of loan token and collateral token.
- Signed offers and relayer discovery are not described as custody-free guarantees; the core's on-chain checks and the relayer's liveness role must remain distinct.
- The article must link to the Security page for token, authorization, collateral-wrapper, and unaudited-code risks.

## GitBook integration

- Create `bitcoin-credit-markets.md` at the repository root.
- Add `Bitcoin credit markets` to `SUMMARY.md` immediately after Introduction and before Using the app.
- Add a short link from `README.md` under Where to start.
- Preserve the existing Development Preview warning and avoid duplicating app instructions in the research article.

## Validation

The completed change will be checked for:

- a complete argument matching all ten article sections;
- no Pool, LP, vault, curator, subscription, deposit, or redemption feature claims;
- no contract addresses or production-deployment claims;
- consistency with `protocol-overview.md`, `using-the-app.md`, and `security.md`;
- valid local Markdown links and GitBook navigation;
- clean Markdown whitespace and readable heading hierarchy;
- original phrasing with no direct quotations from the reference articles.
