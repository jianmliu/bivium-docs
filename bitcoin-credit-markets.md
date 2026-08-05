# Bitcoin Collateral Needs Credit Markets

Bitcoin can serve as high-quality collateral without automatically producing a credit market. A
bilateral loan can originate capital, but it does not by itself create a claim that other
participants can price, transfer, or compare with a claim of the same term. Collateral
quality matters, yet it does not create term liquidity on its own.

Bivium approaches BTC- and ETH-backed credit as fixed-rate, fixed-term, non-recourse
market activity. The important question is not only whether a borrower can obtain loan
tokens against collateral. It is whether the resulting credit has a clear term, a shared
identity, a quoted price, and a defined settlement outcome. For the mechanics behind
that model, see the [protocol overview](protocol-overview.md).

## A term is part of the instrument

Specifying a maturity gives the claim a fixed horizon and makes same-term claims
comparable. It also establishes a single repayment cutoff: a borrower may repay only
while the current time is strictly before maturity. At and after maturity, repayment
closes.

That cutoff does more than set a date on a loan. It lets participants compare claims
with the same remaining term and know when their settlement exposure changes. Different
maturities are different instruments, even when they use the same loan token and the
same represented BTC collateral. A market may develop useful depth at a given term, but
neither the term nor the collateral guarantees that outcome.

## Standardized credit has a narrow domain

In Bivium, a dual-currency note (DCN) is fungible credit only within one exact market
identity. That domain is: `chainId`, `bivium` (the core contract address), `loanToken`,
`collateralToken`, `maturity`, `strike`, `allowPartialRepay`, and `gate`.

Every field matters. A change to the chain or core deployment changes the domain; so
does a change to either token, maturity, strike, repayment policy, or gate address. The
`gate` field is a domain-bound access-policy address, or zero for unrestricted access;
when non-zero, that address may also expose lifecycle hooks for debt origination and
repayment. DCN is therefore not a general claim on a borrower and is never portable
across those domains. Within one exact domain it represents the same market-level
settlement claim, which is what makes transfer and comparison possible without
pretending that unlike credit is interchangeable.

## One claim, one Offer model

Borrow, Lend, Buy, and Sell are different ways to move the same market claim. They use
the same Offer and fill model:

- Lend acquires credit.
- Borrow creates debt and posts the required collateral as the position moves below
  zero.
- Buy and Sell transfer existing credit between participants.

The offer carries a `tick`, which encodes price on the core's grid. The interface can
derive an APR from that price for display, but APR is not an independent on-chain term.
The immutable core does not set a rate. It checks the submitted market domain, the
maker-authorized ratifier and its result, offer timing and capacity, and the lifecycle
conditions applicable to the fill. New debt issuance is permitted only before maturity;
a fill that purely transfers existing credit can remain possible at or after maturity,
subject to offer timing, available credit and liquidity as applicable, authorization,
gate policy, and the other execution checks. A displayed APR does not establish fair
value, available liquidity, or a likely settlement result. Those depend on actual
quotes, counterparties, market terms, and conditions at execution.

This separation is deliberate. A maker authorizes a ratifier, and the core delegates
offer ratification to that contract. A signature-based ratifier verifies a maker
signature over the offer commitment, while another configured on-chain ratifier can
attest policy without a maker signature. In the current Development Preview, signed
quotes can be used for offer discovery, but signing is not a universal property of an
Offer or of ratification. The core does not implement offer-signature verification; it
does separately verify EIP-712 signed authorization grants, which can confer scoped or
full operator authority, including fund-moving capabilities. Market participants or
configured quote authorities make economically consequential pricing choices. A
relayer can help participants discover offers, but discovery is not execution: the core
rechecks the submitted domain, maker authorization of the ratifier, ratification result,
offer timing and capacity, and the lifecycle conditions applicable to that fill.

## Repay or deliver

Before maturity, a borrower can repay the fixed debt and recover collateral. That right
ends strictly at maturity. Debt repaid before the cutoff contributes loan tokens to the
market-level settlement basket. Debt left unpaid contributes its collateral as-is.
From maturity onward, DCN holders can claim their pro-rata share of the resulting loan
token and collateral token amounts.

This is physical delivery, not a mid-term collateral sale. A DCN holder consequently
has short-put-like exposure: repayment can produce loan-token proceeds, while unpaid
debt can produce collateral-token delivery at the market's terms. It is not a promise
that a quoted yield compensates for that exposure.

The settlement rule also explains Bivium's core properties. The core does not need an
oracle to decide what collateral is worth, because it does not reprice collateral on the
way to maturity. It has no forced liquidation engine and no margin call: there is no
mid-term price threshold that compels a borrower to close. The choice is bounded by the
term—repay before the cutoff and recover collateral, or let the collateral enter
settlement after it.

## Risk changes form

Removing forced liquidation does not remove risk. It changes the risk the credit holder
accepts. A holder must be prepared for assignment through physical delivery and for a
two-token settlement result rather than only loan-token repayment. The economic quality
of the collateral matters, as does the quality of any BTC representation or wrapper:
an EVM market settles in its configured token, not necessarily in native BTC.

Other material risks include:

- whether there is enough executable liquidity at an acceptable quote;
- the authority and scope of any signer or other quote authority;
- relayer liveness, censorship, delay, or ranking of discovered offers;
- correct validation of the exact market domain before acting;
- gate policy and any cached lifecycle hooks, which can revert or make external calls;
- token behavior and the assumptions required of each configured token; and
- smart-contract, integration, authorization, and interface risk.

The core invokes `beforeTake`, `afterTake`, `beforeRepay`, and `afterRepay` when the
domain's gate advertised hook support at the market's first touch. The reentrancy guard
and core accounting checks constrain that surface, but they do not eliminate
external-call, integration, or origination-and-repayment liveness risk.

The [Security guide](security.md) explains these boundaries in greater depth, including
the distinction between a quote authority's economic influence and direct custody
authority. No displayed rate removes the need to assess the collateral, domain, and
counterparty terms.

## Keep the trust boundaries visible

Bivium's core is immutable and oracle-free: it records the fixed market terms and
applies repay-or-deliver accounting rather than deciding market prices. That boundary
does not make every surrounding component equally trustless. Quotes express the
judgment of their makers or configured authorities. A relayer is a discovery and
liveness component, not the source of execution truth. Client and core domain checks
must prevent a familiar-looking asset pair from being treated as the intended market.
And collateral representation carries its own custody, bridge, or wrapper assumptions.

The practical discipline is to validate that the full eight-field domain is the market a
user intends to use. A familiar name such as “BTC credit” is not sufficient
identification. The core binds execution to the submitted domain, but it cannot infer
whether that domain matches a user's intent. That domain is the identity of fungible
DCN; the full Offer adds the transaction terms a participant accepts and is not part of
the fungible claim identity.

## Current availability

The [Bivium Development Preview](https://dev.bivium.pages.dev) is the place to inspect
the currently visible, enabled workflows. It is unaudited, non-production software for
testing only. Do not use real funds.

This article describes the market design; it does not repeat the interface steps. For
the available screens and their operating cautions, read [Using Bivium](using-the-app.md).
If a workflow is not visible and enabled in the Preview, this documentation does not
present it as available.

## Market structure is the point

Bitcoin-backed credit becomes more legible when its term, claim identity, price, and
settlement are explicit. That structure can make credit transferable and comparable;
under suitable participation and quoting conditions, it may also support useful
liquidity. It does not promise inexpensive borrowing, deep markets, or risk-free yield.

Bivium's contribution is this market structure: fixed terms, standardized
domain-bound DCN, ratified offer pricing, and repay-or-deliver settlement. The result is a way
to reason clearly about Bitcoin collateral and credit risk—not a guarantee that either
will be cheap or safe.
