# Security

## Status

**Bivium is an unaudited proof of concept. Do not use it with real funds.**

It has not been audited, formally verified, or run through a bug bounty. A legacy Sepolia deployment
may exist; the fresh domain-bound release described here has not been deployed or promoted to
production. The test suite (unit + invariant) checks intended behaviour and core solvency properties,
but that is **not** a security guarantee.

## Provenance

- `src/Bivium.sol`, `src/interfaces/*`, `src/libraries/*`, the pluggable `src/ratifiers/*`
  (`AuctionRatifier`, `CurveRatifier`, `GuardedCurveRatifier`, `MerkleSignatureRatifier`,
  `SetterRatifier`, `SignatureRatifier`), `src/periphery/*` (`CreditFeeRouter`,
  `IntentSettlementRouter`, `WethGateway`), and `src/gates/*` (`AllowlistGate`)
  are original, clean-room code written from a written spec. They do not derive from any third-party (BUSL/GPL)
  source.
- `DualCurrencyPoolManager` and the vault contracts are optional pool/vault surfaces. They are not the
  stable default execution path and should not be inferred to be enabled from their presence in the source.
- The only production dependency is **OpenZeppelin Contracts** (MIT), including its ERC-20 and ERC-3156
  interfaces, `SafeERC20`, `ERC20`, `ReentrancyGuard`, `ReentrancyGuardTransient`, `ECDSA`, `Math`,
  `Ownable`, and `IERC165`. Token transfers and signature verification (only in ratifier and router
  peripherals; the core runs no signature scheme) use these building blocks rather than bespoke code.

## Trust model

- **Immutable, no admin.** There is no owner, role, pause, or upgrade path. A deployed market cannot
  be altered, frozen, or drained by any privileged party — and a bug cannot be patched in place.
- **No oracle.** Settlement depends only on whether a borrower repaid before maturity. There is no
  price feed to manipulate.
- **Explicit callback and flash surfaces.** Callback-enabled variants of `fill`, `fund`, and `repay`
  support defined integration interfaces, and the core also exposes ERC-3156 flash lending. The
  reentrancy surface is bounded by those explicit interfaces, call ordering, reentrancy guards, and
  accounting checks; it is not absent.
- **Lender optionality.** Borrowers hold an economic option at maturity (repay or let the collateral
  be delivered). Lenders accept that a default yields collateral instead of loan tokens. This is a
  product outcome, not bad debt.

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
- **Ratifiers carry no fund power.** Naming a `ratifier` is a *separate* registry (`setRatifier` /
  `isRatifier`), read only by the `view` `_ratify`. A ratifier can attest a maker's offers but can
  **never** move the maker's funds — even a malicious/compromised ratifier is bounded to "ratifies an
  offer within the maker's already-posted, `consumed`-capped budget," not theft. (A full operator is
  *not* automatically a ratifier, and vice-versa.)
- **Quoting keys are off the path.** Signature-based ratifiers keep their *own* signer registries, so a hot
  quoting key signs attestations only and holds no core authorization (the M-01-safe pattern).
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

### Release and operating controls

The relayer separates offers physically by chain, core, and market, and ignores legacy or domainless
records. Before acting, the manager validates the full market parameters. The keeper fails closed
until its release marker and manifest bind the expected core, manager, ratifier, and pools.

A release manifest pins source, artifact, and client digests, but independent deployment verification
is still required. Development must remain active for at least seven days **and** cover at least one
complete market maturity cycle; these are independent gates and may overlap. Only after both pass are
the artifacts reproduced for Main. No fresh-release addresses are published here, and that release is
not claimed to have been deployed or promoted to production.

This is an ABI-breaking transition: the fresh core, frontend, and keeper do not migrate or accept
legacy offers as the new encoding. An old signature may still preflight as `RATIFIED` against its exact
old commitment, but that does not make it executable on the current core. Old positions stay on their
legacy core, so users need a legacy read-and-exit path until settlement.

### Lender-side risk: adverse selection in physical-delivery yield

This is an **economic** risk, not a contract bug — but it is the single most important thing a lender must
understand, so it is stated here explicitly.

A lender holding credit is **short a put** on the collateral: the APR is the premium for being assigned the
collateral if the borrower defaults (delivers instead of repaying). High APR is therefore **not** free yield —
it is the market pricing a high probability and/or low recovery of assignment:

- **Soft case (legitimate).** On good collateral (e.g. BTC/ETH), a higher APR means a higher chance you are
  assigned — a strike near spot, or a volatile asset. This is normal option premium: fine **iff you would
  genuinely be happy to own that collateral at `strike − premium`.**
- **Hard case (adverse selection / fraud).** With permissionless, oracle-free markets, a borrower can mint a
  worthless token, name it as collateral, advertise a high APR to attract lenders, draw real loan tokens, and
  at maturity **deliver the junk** — extracting the principal. The high APR was bait. This is the classic
  "market for lemons": the borrower has private information about both their intent to default and the
  collateral's true value.

**Rule of thumb:** `APR ≈ risk-free rate + assignment-risk premium`. The larger the premium, the more the
market is telling you that you will likely end up holding the collateral. Dual-currency yield is only sound
when the delivered asset is one you **want to hold**, at a strike you **would buy at**, and the asset is
**real, liquid, and not mintable/riggable by the borrower** — which is why traditional dual-currency products
only offer blue-chip assets.

**Where the defense lives.** The core is deliberately neutral — it does not (and will not) judge collateral
quality, exactly as Uniswap will not stop you LP-ing a honeypot. The defense is at the **edges**: fund only
vetted `(collateral, strike)` markets via a **curator vault** (ERC-4626) or a market list, and surface in the
UI a decomposition of APR into time-value vs. assignment-risk premium with a warning that high APR means high
assignment probability. See the lender-side risk discussion in the [protocol overview](protocol-overview.md).

## Token assumptions (MUST hold per market)

Bivium accounts token amounts exactly. A market is only safe if **both** of its tokens are standard
ERC-20s:

- **No fee-on-transfer / deflationary tokens** — the contract assumes the amount sent equals the
  amount received.
- **No rebasing / balance-changing tokens** — balances must change only on explicit transfers.
  (A non-rebasing yield-bearing wrapper such as an ERC-4626 share is fine and is the recommended way
  to keep idle pool liquidity productive.)
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

## Properties checked by tests

Invariant tests (`test/Bivium.invariant.t.sol`) fuzz random sequences of every fund-moving action
and assert:

- **Loan solvency** — the contract's loan balance always exactly backs withdrawable lender liquidity
  plus the unclaimed repaid-loan pool.
- **Credit conservation** — the sum of holder credit equals originated face minus claimed face (no
  credit can be minted from nothing).
- **Collateral backing** — delivered collateral still owed to holders is fully backed, and claims
  never exceed the delivered pool.

Symbolic proofs (`test/BiviumSymbolic.t.sol`, via [Halmos](https://github.com/a16z/halmos)) verify —
for *all* inputs, not sampled — the repay-or-deliver state-machine guards: `fill` and `repay` always
revert at/after maturity, and `claim` always reverts before it.

Static analysis ([Slither](https://github.com/crytic/slither), `slither . --fail-high`) reports zero
findings (one "arbitrary from in transferFrom" is a triaged false positive: the `from` is gated by the
contract's authorization check).

These do not cover: gas-griefing, cross-market interactions beyond a single market, or economic/MEV
behaviour. The borrow-side arithmetic (strike collateralization, rate bound) is checked by fuzzing, not
symbolically — SMT solvers struggle with the 256-bit multiply/divide. All `mulDiv` use OpenZeppelin's
full-precision `Math.mulDiv` (512-bit intermediate), so the products do not overflow prematurely, and
`claim` distributes pools via cumulative rounding, leaving no stranded dust once fully redeemed.

## Before any production use

1. Independent security audit(s).
2. Extended fuzzing / formal verification of the accounting invariants.
3. Fresh-release Development deployment and a public bug bounty.
4. Per-market review that the chosen tokens satisfy the assumptions above.

## Reporting a vulnerability

The fresh domain-bound release has no production deployment. If you find an issue, please open a
private report to the repository owner rather than a public issue.
