# Security

## Status

**Bivium is an unaudited proof of concept. Do not use it with real funds.**

It has not been audited, formally verified, or run through a bug bounty. The current Development
Preview is available for testing, but the domain-bound release has not been promoted to production and
no contract addresses are published here. The test suite (unit + invariant) checks intended behaviour
and core solvency properties, but that is **not** a security guarantee.

## Provenance

- `src/Bivium.sol`, `src/interfaces/*`, `src/libraries/*`, the pluggable `src/ratifiers/*`
  (`AuctionRatifier`, `CurveRatifier`, `GuardedCurveRatifier`, `MerkleSignatureRatifier`,
  `SetterRatifier`, `SignatureRatifier`), `src/periphery/*` (`CreditFeeRouter`,
  `IntentSettlementRouter`, `WethGateway`), and `src/gates/*` (`AllowlistGate`)
  are original, clean-room code written from a written spec. They do not derive from any third-party (BUSL/GPL)
  source.
- The only production dependency is **OpenZeppelin Contracts** (MIT), including its ERC-20 and ERC-3156
  interfaces, `SafeERC20`, `ERC20`, `ReentrancyGuard`, `ReentrancyGuardTransient`, `ECDSA`, `Math`,
  `Ownable`, and `IERC165`. Token transfers and signature verification (only in ratifier and router
  peripherals; the core runs no signature scheme) use these building blocks rather than bespoke code.

## Trust model

- **Immutable core, no protocol administrator.** There is no protocol owner, pause, or upgrade path, so
  no protocol administrator can drain or rewrite a market and a bug cannot be patched in place. Users
  can still grant full or scoped operator authority, and makers can authorize economically powerful
  quote authorities, with the blast radii described below.
- **No oracle.** Settlement depends only on whether a borrower repaid before maturity. There is no
  price feed to manipulate.
- **Explicit callback and flash surfaces.** Callback-enabled variants of `fill`, `fund`, and `repay`
  support defined integration interfaces, and the core also exposes ERC-3156 flash lending. The
  reentrancy surface is bounded by those explicit interfaces, call ordering, reentrancy guards, and
  accounting checks; it is not absent.
- **Pooled repay-or-deliver settlement.** Repayment is permitted strictly before maturity. From maturity
  onward it is closed: debt repaid before the cutoff contributes loan tokens, unpaid debt contributes
  collateral, and fungible credit holders claim the combined basket pro rata.

### Authorization & the fund-custody critical path

The **core** is the sole persistent custodian of market escrow and accounting balances. Periphery such
as `IntentSettlementRouter` and `WethGateway` can pull, receive, or wrap assets transiently within an
atomic call, but does not hold them as long-term market custody. A component can put a user's funds at
risk through a core bug, a periphery bug while assets are in flight, or operator powers granted by the
user. The authorization model limits those paths:

- **Capability-scoped operators.** `grantAuthorization(operator, capabilities, expiry)` hands an
  operator only specific `CAP_*` bits (one per fund-moving function) and an optional expiry, instead of
  full custody. `IntentSettlementRouter` receives `CAP_FILL` only. `WethGateway` intentionally
  needs `CAP_FILL` plus `CAP_WITHDRAW_COLLATERAL`; other operators should receive the minimum capability
  set for their workflows. An operator's blast radius is limited to its granted capabilities, expiry,
  and the checks on those actions. `setAuthorization(x, true)`
  remains as the explicit full-custody grant (`ALL_CAPS`, no expiry) — use it only for fully trusted
  accounts.
- **Quote authority differs from direct custody authority.** A registered ratifier or signer has no
  direct operator or withdrawal capability merely by being a quote authority. It can nevertheless
  approve hostile-price bids or asks that fills execute. Under signer compromise, the signer can create
  new offers with hostile caps and fresh consumption groups, so those signer-chosen values are not a
  trustworthy loss bound. The on-chain ceiling is the maker liquidity available to bids or maker credit
  available to asks, together with the core-recorded consumption and caps of offers that actually execute.
  A compromised quote authority can therefore cause economic loss even though it cannot call withdrawals
  directly. Revoke the signer/ratifier and stop publication promptly on suspected compromise. (A full
  operator is not automatically a ratifier, and vice versa.)
- **Order book / relayer / frontend are off the path.** Offer discovery and matching are entirely
  off-chain and custody-free; `fill` re-checks every economic invariant on-chain, so a
  compromised relayer can censor or mis-rank (a liveness issue) but cannot move funds.
- **Gates are access-policy only.** A market `gate` is consulted via `view` (`canIncreaseCredit/Debt`)
  and never authorized; a faulty gate can wrongly permit/deny credit or debt increases but cannot
  extract escrowed funds.

Core and periphery authorization tests cover scoped grants, expiry, and the limited capabilities used
by the intent router and WETH gateway.

### Chain and core domain boundary

Every market is identified by exactly `(chainId, bivium, loanToken, collateralToken, maturity, strike,
allowPartialRepay, gate)`, where `bivium` is the intended core address. An offer repeats those eight
fields as its prefix, and the full flat offer is always hash-committed for ratification. Signature-based
ratifiers verify maker signatures over that commitment; managers, curves, and other on-chain ratifiers
can attest policy without a maker signature. Changing either the chain or core therefore changes both
the market ID and offer commitment. Operating a separate ratifier for each core is useful hygiene, but
the domain-bearing core is the primary replay boundary.

On the first touch of an uncreated market, the core checks `chainId`, `bivium`, and the maximum
time-to-maturity (`10 * 365 days`) before token, gate, or hook probes. These checks are creation-only. If
the runtime chain ID later changes, an untouched old-domain market cannot be created, while a market
already touched remains live for its normal lifecycle: fills, repayment, claims, collateral and
liquidity withdrawal, and credit transfers. A chain-ID change is not an automatic trading halt for
existing markets.

Ratification receives the economic taker represented by the action, which may differ from the account
that submitted the transaction. A public `isRatified` call is only a preflight: the core remains the
authority on the represented parties, offer commitment, remaining capacity, market state, and all
other execution checks.

### Required fresh-release promotion controls

The relayer separates offers physically by chain, core, and market, and ignores legacy or domainless
records. Before acting, the client validates the full market parameters. The keeper's runtime release
marker is implemented and fails closed on a mismatch.

The following are required promotion policy for the fresh release, not a claim that every control is
already deployed or enforced: the release manifest must bind the correct core and ratifier and pin
source, artifact, and client digests; independent deployment verification is required;
Development must remain active for at least seven days **and** cover at least one complete market
maturity cycle; and artifacts must then be reproduced for Main. Manifest binding and address updates
remain pending release work. The Development Preview frontend is deployed for testing, but no
fresh-release contract addresses are published here and the release is not promoted to production.

This is an ABI-breaking transition: the fresh core, frontend, and keeper do not migrate or accept
legacy offers as the new encoding. An old signature may still preflight as `RATIFIED` against its exact
old commitment, but that does not make it executable on the current core. Old positions stay on their
legacy core, so users need a legacy read-and-exit path until settlement.

### Lender-side risk: adverse selection in physical-delivery yield

This is an **economic** risk, not a contract bug — but it is the single most important thing a lender must
understand, so it is stated here explicitly.

A lender holding fungible market credit has short-put-like exposure to the collateral delivered by
unpaid debt. The quoted APR can combine time value, assignment/default-risk compensation, liquidity
conditions, and maker spread. High APR is therefore **not** free yield and, by itself, is not a calibrated
or monotonic default probability:

- **Soft case (legitimate).** On good collateral (e.g. BTC/ETH), rate compensation may reflect strike,
  volatility, term, liquidity, and assignment exposure. It is suitable only if you would genuinely be
  happy to receive that collateral through the pooled settlement.
- **Hard case (adverse selection / fraud).** With permissionless, oracle-free markets, a borrower can mint a
  worthless token, name it as collateral, advertise a high APR to attract lenders, draw real loan tokens, and
  at maturity **deliver the junk** — extracting the principal. The high APR was bait. This is the classic
  "market for lemons": the borrower has private information about both their intent to default and the
  collateral's true value.

Dual-currency yield is only sound when the delivered asset is one you **want to hold**, at a strike you
would accept, and the asset is
**real, liquid, and not mintable/riggable by the borrower** — which is why traditional dual-currency products
only offer blue-chip assets.

**Where the defense lives.** The core is deliberately neutral — it does not (and will not) judge collateral
quality, exactly as a neutral exchange will not stop you providing liquidity to a honeypot. The defense
is at the **edges**: fund only
vetted `(collateral, strike)` markets via a **curator vault** (ERC-4626) or a market list, and surface the
components and uncertainty behind a displayed APR rather than presenting it as a default probability.
See the lender-side risk discussion in the [protocol overview](protocol-overview.md).

## Token assumptions (MUST hold per market)

Bivium accounts token amounts exactly. A market is only safe if **both** of its tokens are standard
ERC-20s:

- **No fee-on-transfer / deflationary tokens** — the contract assumes the amount sent equals the
  amount received.
- **No rebasing / balance-changing tokens** — balances must change only on explicit transfers.
- **No transfer hooks (e.g. ERC-777)** — although CEI + `nonReentrant` defend against reentrancy,
  hook tokens are out of the supported set.

Creating a market with a non-conforming token is unsafe and is the deployer's responsibility.

### Collateral asset (per market)

Beyond the ERC-20 conformance above, the **economic** quality of the collateral is what the lender's short-put rests on
(see adverse selection, above). The intended universe:

- **Primary: BTC, then ETH.** Deep, liquid, blue-chip, and — critically — **hedgeable on a continuous options market**
  (Deribit for crypto; CME/IBIT increasingly in-band for BTC), so a solver can offset its assigned-put risk 1:1 and the
  asset is not mintable/riggable by the borrower. These are the adverse-selection-*safe* collateral. BTC leads; ETH is
  the second blue-chip.
- **Long-tail / niche collateral is deliberately not supported.** Supporting it safely requires exactly the machinery
  Bivium omits — a price-oracle suite, collateral-utility compartments, and token wrappers (the breadth design used by oracle-based, long-tail lenders). Bivium's scope is the opposite: **depth on one or two blue-chips, oracle-free.** A niche
  asset can only ever be a bespoke/off-grid market at the curator's explicit risk.
- **Wrapper choice matters.** Deribit/CME hedge in **native BTC**, but EVM collateral is necessarily a wrapped/vault
  token. Prefer **trustless-vault native BTC/ETH** (a Babylon/BitVM-style self-custodial vault) over custodial WBTC/cbBTC:
  this swaps *custodian-credit* depeg for smaller *bridge/challenge-window* risk. The vault's peg-out delay does **not**
  affect Bivium's settlement (the core settles in the EVM token; the delay bites only at terminal redemption to L1) and
  is intermediated off-protocol by a **front-pay** liquidity service — but the residual native-vs-vault-token basis is a
  tail the Deribit hedge cannot remove.
- **Extension: institution-custodied tokenized blue-chips / RWAs** (equities, treasuries) on a permissioned/institutional
  venue, where custody is regulated and the hedge is listed equity/rates options. This rides the tokenized-collateral
  thesis and keeps the collateral adverse-selection-safe by construction (permissioned issuance), at the cost of a
  permissioned venue.

Assets with no deep options market to hedge, or that the borrower can mint/rig, must stay bespoke/off-grid and are the
curator's explicit risk.

## Assurance scope at the pinned source revision

These statements describe Bivium core revision
`02a1730d94f5d192f0ef81c5ebcc3ebe497321d9`. They must be refreshed when the release commit changes.

The stateful invariant handler in `test/Bivium.invariant.t.sol` fuzzes one market through funding,
primary origination by filling a lender bid, partial repayment, collateral and liquidity withdrawal,
credit transfer and claim, maturity progression, and wrong-chain/wrong-core creation probes. It asserts:

- **Loan solvency** — the contract's loan balance always exactly backs withdrawable lender liquidity
  plus the unclaimed repaid-loan balance.
- **Credit conservation** — the sum of holder credit equals originated face minus claimed face (no
  credit can be minted from nothing).
- **Collateral backing** — pooled collateral still owed to holders is fully backed, and claims
  never exceed the delivered collateral allocation.

Secondary fills, callback-enabled variants, and ERC-3156 flash loans are outside that stateful handler.
Some paths have separate unit tests (including secondary fills, the fill callback, and flash-loan
transfer-accounting reverts), but this invariant suite does not establish their sequence-level safety;
fund/repay callback combinations and broader flash behavior remain assurance gaps.

Symbolic assurance is limited for the new eight-field market boundary. Local verification migrated
`test/BiviumSymbolic.t.sol` to the exact domain-bound repayment selector and Halmos proved that a positive
repayment reaches the core's `MaturityPassed` rejection at or after maturity. The harness still does not
establish the fill maturity boundary, so no symbolic fill-maturity claim is made.

Static analysis ([Slither](https://github.com/crytic/slither),
`slither . --foundry-compile-all --fail-high`) completes without a high-severity failure. Lower-severity
detector output remains review input rather than a claim of zero findings.

The current tests also do not establish gas-griefing resistance, comprehensive cross-market behavior,
or economic/MEV safety. The borrow-side arithmetic (strike collateralization and rate bounds) is covered
by unit/fuzz tests, not a current symbolic proof. All `mulDiv` use OpenZeppelin's
full-precision `Math.mulDiv` (512-bit intermediate), so the products do not overflow prematurely, and
`claim` distributes both settlement assets via cumulative rounding, leaving no stranded dust once all
credit is claimed.

## Before any production use

1. Independent security audit(s).
2. Extended fuzzing / formal verification of the accounting invariants.
3. A complete Development observation period and a public bug bounty.
4. Per-market review that the chosen tokens satisfy the assumptions above.

## Reporting a vulnerability

The fresh domain-bound release has no production deployment. If you find an issue, please open a
private report to the repository owner rather than a public issue.
