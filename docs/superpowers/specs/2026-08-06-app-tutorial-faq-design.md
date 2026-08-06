# App Tutorial and FAQ Design

## Goal

Restructure the Bivium product documentation into an end-to-end app tutorial and a separate FAQ. The information architecture may follow the useful pattern of introducing product terms before step-by-step actions and collecting common questions afterward, but all rules and wording must come from Bivium's deployed behavior and protocol design.

## Source of truth

The current `bivium-frontend` `origin/dev` behavior, the Development Preview configuration, and the domain-bound core documentation are the sources of truth. A control that is dormant in source code but not enabled in the Preview is not documented as available.

The Binance Lite Loan page informed only the information hierarchy. Its custody model, fixed service fee, collateral yield, auto-repayment, overdue interest, LTV monitoring, and post-term liquidation rules do not apply to Bivium and must not appear as Bivium behavior.

## Information architecture

### `using-the-app.md`

The existing page will become a linear task guide:

1. **Before you start** — Development Preview, Sepolia, test assets, wallet connection, and Header domain checks.
2. **Understand the market terms** — loan token, represented BTC/ETH collateral, strike, maturity, APR, face, and dual-currency note (DCN).
3. **Borrow** — select an active configured market, enter the amount, review APR/face/collateral/strike/maturity, approve collateral when required, submit Borrow, and verify the Loan row.
4. **Manage a Loan** — read debt, locked collateral, lifecycle state, and maturity in Portfolio.
5. **Repay and reclaim collateral** — approve the loan token if needed, repay strictly before maturity, then reclaim collateral when the UI exposes the action.
6. **Lend and receive DCN** — use Lend now against an executable ask, review terms, submit the fill, and verify the DCN row.
7. **Trade in Pro** — select the exact market, take liquidity or place an enabled order, inspect the positions tape, and cancel a resting Order.
8. **Claim matured DCN** — review Estimated proceeds, distinguish loan-token and collateral-token estimates, submit Claim, and explain that final amounts use execution-time state.
9. **Understand unavailable states** — matured markets, missing quotes, relayer unavailability, and Portfolio market-domain validation failure.
10. **Continue to FAQ and Security** — route conceptual questions and risk detail to focused pages.

Every action sequence will distinguish a visible button from a protocol condition. For example, an active market and executable quote are prerequisites for Borrow or Lend; the guide must not imply that every configured market always has liquidity.

### `faq.md`

The new FAQ will answer:

1. Does Bivium use LTV, margin calls, or price-triggered liquidation?
2. When can a borrower repay?
3. What happens when debt is unpaid at maturity?
4. Is displayed APR a service fee or a guaranteed return?
5. What is DCN and where is it fungible?
6. Can DCN be transferred or traded after maturity?
7. Why can Claim return both loan token and collateral token?
8. Are Estimated proceeds guaranteed?
9. Why does Portfolio unavailable not mean the account is empty?
10. What can a relayer failure affect?
11. Which collateral assets and markets are supported?
12. Is partial repayment supported?
13. Can real funds be used in the Development Preview?
14. Are older-core offers and positions migrated?

Answers will be concise and link to the protocol or security page for deeper detail. Questions whose answer depends on the exact configured market, such as partial repayment, will explicitly direct users to the market terms instead of making a global promise.

## Navigation changes

- Keep `Using the app` after `Bitcoin credit markets` in `SUMMARY.md`.
- Add `FAQ` immediately after `Using the app` and before `Protocol overview`.
- Add a concise FAQ link under `README.md` → `Where to start`.
- Add reciprocal links between the tutorial and FAQ where helpful.

## Safety and accuracy boundaries

- Bivium repayment is permitted strictly before maturity; from maturity onward Claim is available and repayment is closed.
- Pure secondary transfers of existing credit may remain possible after maturity when offer timing, credit availability, and other execution conditions allow; the guide must not promise post-maturity liquidity.
- The core does not use an oracle or forced liquidation. Unpaid collateral participates in physical repay-or-deliver settlement.
- Displayed APR is derived from offer price and term, not a protocol-set service fee or guaranteed return.
- Estimated proceeds are current-state previews, not transaction guarantees.
- The relayer affects discovery and liveness, not core custody, while the client and user must still validate the intended market domain.
- BTC/ETH collateral may be represented by external tokens or wrappers whose issuer, bridge, redemption, basis, and liquidity risks remain external assumptions.
- The Development Preview is unaudited, non-production, and for test assets only.

## Explicit exclusions

The tutorial and FAQ will not document or imply availability of:

- Pool or LP workflows;
- deposits, subscriptions, redemption queues, or manager settlement;
- vaults or curators;
- auto-repayment;
- overdue penalty interest;
- post-term LTV monitoring or liquidation;
- collateral yield or Earn-account behavior;
- repayment with arbitrary alternative assets;
- production contract addresses.

## Screenshots and video boundary

This documentation update is text-first. It will not include screenshots captured from matured markets or unconnected wallet states, because those images would quickly become stale and would not demonstrate an executable flow. Screenshots may be added later after active markets and a prepared test wallet are available.

## Validation

The completed documentation will be checked for:

- all ten tutorial sections and all fourteen FAQ questions;
- consistency with the current frontend actions and protocol lifecycle;
- no unavailable feature claims or Binance-specific behavior;
- conditional wording around liquidity, active markets, partial repayment, and post-maturity transfers;
- no contract addresses or production claims;
- valid local Markdown links and correct GitBook navigation order;
- clean Markdown whitespace and readable heading hierarchy;
- no stale screenshots or instructions that require MetaMask inside the in-app browser.
