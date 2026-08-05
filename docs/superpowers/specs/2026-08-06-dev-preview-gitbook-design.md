# Dev Preview GitBook Alignment Design

## Goal

Align the public Bivium GitBook with the functionality that is actually enabled in the current dev Preview deployment. The documentation must not present Pool or other disabled workflows as available product features.

## Source of truth

The documentation is derived from the current `bivium-frontend` `origin/dev` behavior and its Preview configuration. The public Preview URL is `https://dev.bivium.pages.dev`.

The deployed interface, enabled navigation, configured market list, and reachable user actions take precedence over older design documents or code paths that remain in the repository but are not enabled for users.

## Documentation scope

The GitBook will describe:

- the dev Preview as a test environment rather than a production deployment;
- the enabled top-level navigation and the Markets, Portfolio, and Pro experiences;
- listed markets as the bounded market universe used by the interface;
- Portfolio coverage for enabled Loan, direct DCN, and resting Order records;
- the available lifecycle actions for those records, including repay, collateral reclaim, order cancellation, and matured DCN claims where applicable;
- the estimated loan-token and collateral-token proceeds shown before a matured DCN claim;
- safe failure behavior when required market-domain validation or external data cannot be established;
- existing unaudited-software, test-asset, and configuration-change warnings.

## Explicit exclusions

The GitBook will not document Pool, LP deposit or withdrawal, subscription pools, pool discovery, pool settlement, LP redemption, or any other feature that is not enabled in the current dev Preview.

Implementation details may still contain dormant Pool-related code. Those details are not user-visible product behavior and will not be presented as available. Protocol explanations that require discussing internal pool identifiers will use neutral terms such as market or position identifier and will not imply an enabled Pool product.

## Files and responsibilities

- `README.md`: identify the documentation as covering the current dev Preview and preserve the non-production warning.
- `using-the-app.md`: document only enabled screens, records, actions, claim proceeds, loading behavior, and user-facing limitations.
- `protocol-overview.md`: retain the market-domain and offer-safety model while removing claims that disabled Pool/LP workflows are available.
- `security.md`: keep assurance boundaries and explain safe handling of invalid or unavailable market configuration without advertising disabled Pool features.
- `SUMMARY.md`: change only if navigation labels need to match the revised page contents.

## Content model

The user guide will follow the interface journey:

1. Open the dev Preview and verify that it is a test environment.
2. Connect a wallet on the supported test network.
3. Select one of the markets listed by the deployment configuration.
4. Use Markets or Pro for the enabled trading and borrowing flows.
5. Use Portfolio to review Loan, DCN, and Order records.
6. Use the contextual action shown for a record.
7. For a matured DCN claim, review the estimated proceeds before submitting the transaction.

## Safety and error wording

Documentation must distinguish an unavailable view from an empty account. If the application cannot validate the market domain or load a required source, the user must not infer that a position is absent. The guide will advise retrying and independently verifying on-chain state before acting.

Estimated claim proceeds are previews, not guarantees. Final amounts are determined by the contract state when the transaction executes.

## Validation

The completed documentation will be checked by:

- searching all published Markdown pages for Pool, LP, subscription, redeem, settle, and related terms;
- manually reviewing every remaining match and retaining it only when it is a necessary protocol disclaimer rather than an available-feature claim;
- checking that the Preview URL and all internal GitBook links resolve syntactically;
- comparing enabled feature statements against current `bivium-frontend` `origin/dev` source;
- reviewing the final diff for production claims, contract addresses, disabled functionality, and stale screenshots or instructions.

## Publication boundary

This change updates the GitBook repository only. It does not change frontend deployment configuration, deploy contracts, enable Pool functionality, or publish contract addresses.
