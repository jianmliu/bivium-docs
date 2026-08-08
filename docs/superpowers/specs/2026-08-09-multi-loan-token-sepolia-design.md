# Multi-Loan-Token Sepolia Design

## Status

Design specification for review. This document does not describe a deployed feature.

## Goal

Allow one Bivium deployment to expose fixed-rate markets denominated in more than one loan token,
starting with **Mock GHO** and **Mock USDC** on Sepolia. GHO is the default product path; USDC remains
available as a compatibility and comparison market.

The change must preserve Bivium's existing market isolation and repay-or-deliver semantics. It must not
make a position repayable in a token other than the loan token selected when that position was opened.

## Product decision

The Sepolia interface will support these loan-token choices:

| Loan token | Decimals | Role |
|---|---:|---|
| Mock GHO | 18 | Default Sepolia loan, quote, repayment, and cash-settlement token |
| Mock USDC | 6 | Secondary compatibility market |

The application presents GHO first. It does not describe P-like Bivium credit as GHO or as a
stablecoin. GHO and USDC are the cash legs used to purchase, repay, and settle market credit.

Mock GHO must use 18 decimals. A six-decimal token labelled `GHO` is explicitly prohibited because it
would hide the decimal-boundary failures that a later real-GHO integration must handle.

## Non-goals

This work does not:

- change Bivium Core storage, offer encoding, market hashing, repayment, or claim logic;
- allow one position to borrow one token and repay another;
- combine GHO and USDC deposits in one pool share or calculate a cross-token NAV;
- add a stablecoin swap, peg oracle, or automatic GHO/USDC conversion;
- use Aave v4 `Hub.draw` to fund Bivium borrowers;
- claim that mock-token Sepolia behavior proves production GHO integration;
- remove historical USDC markets or rewrite historical documentation and tests merely for terminology.

## Existing protocol capability

Bivium Core already treats `loanToken` as part of the exact on-chain `MarketParams` identity:

```text
protocolMarketId = keccak256(abi.encode(
    loanToken,
    collateralToken,
    maturity,
    strike,
    allowPartialRepay,
    gate
))
```

Consequently, the following are different, non-fungible markets even if every other field matches:

```text
(GHO,  WETH, maturity, strike, ...)
(USDC, WETH, maturity, strike, ...)
```

`Bivium.computeId` contains exactly those six fields; it does not include `chainId` or the Core address.
Credit, debt, repayment balances, collateral delivery, and claims remain isolated by that on-chain ID.
No Core migration is required.

Off-chain systems must not treat the 32-byte ID as globally unique. The deployment-qualified identity
derived from the manifest and used in URLs, databases, logs, and idempotency scopes is:

```text
qualifiedMarketKey =
  "eip155:" + canonicalChainId + "/" + lowercaseCoreAddress + "/" + lowercaseProtocolMarketId

example:
  eip155:11155111/0x1111111111111111111111111111111111111111/0xaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
```

`canonicalChainId` is base-10 with no sign or leading zeroes; the address is exactly 40 lowercase hex
digits after `0x`; the ID is exactly 64 lowercase hex digits after `0x`. The key is **derived-only** and
is not stored as an independent manifest field or included separately in `manifestHash`. Every consumer
derives it with the same shared golden vectors after validating the manifest. This namespace prevents
records from different chains or Core deployments from colliding without changing the Core hash. Offer
authorization and replay protection remain governed by their existing commitment and signature-domain
rules; this spec does not change either format.

## Required invariant

For every position and every resulting credit claim:

```text
opening loan token == repayment loan token == repaid settlement token
```

An unpaid position may contribute its configured collateral token instead, as under the current
repay-or-deliver rule. No router, frontend, solver, pool, or account abstraction layer may silently
substitute GHO for USDC or USDC for GHO.

## Architecture

```text
                         Bivium Core (unchanged)
                                  |
              +-------------------+-------------------+
              |                                       |
       WETH / GHO markets                       WETH / USDC markets
              |                                       |
      GHO maker inventory                     USDC maker inventory
              |                                       |
       GHO pool instance                       USDC pool instance
              +-------------------+-------------------+
                                  |
                     token-aware frontend/router
```

The design uses five boundaries:

1. Core remains token-agnostic and isolates markets by `loanToken`.
2. Each pool or vault accepts exactly one loan token.
3. Frontend actions are driven by the selected market's token metadata.
4. Solver capital, limits, approvals, and mock minting are accounted per token.
5. Keeper pricing reads loan-token decimals from the configured market or pool rather than assuming
   six-decimal USDC.

## Token policy and registry

Each service consumes an explicit asset registry covering both loan and collateral assets. An asset
record contains:

```ts
type AssetConfig = {
  address: Address;
  symbol: string;
  decimals: number;
  roles: Array<"loan" | "collateral">;
  mock: boolean;
  mintable: boolean;
};
```

The address is the canonical key. Symbol and decimals are metadata, not identity. Services must reject:

- duplicate addresses with conflicting metadata;
- duplicate `(chainId, address)` entries;
- a market whose loan or collateral address is absent or lacks the corresponding role;
- configured decimals that disagree with on-chain `decimals()`;
- a production configuration that marks a token mintable.

Sepolia uses an explicit, bounded list rather than accepting arbitrary ERC-20 addresses. Initial loan
assets use six or eighteen decimals; supported collateral assets use eight or eighteen decimals.
Fee-on-transfer, rebasing, ERC-777-style callback, and balance-mutating tokens are out of scope.

### Canonical deployment manifest

There is one source artifact:

```text
vault-contracts-bivium/deployments/sepolia/multi-loan-v1.json
```

It is generated after broadcast from transaction receipts and read-only chain calls, committed with the
deployment, and never hand-edited. Its minimum schema is:

```ts
type DeploymentManifestV1 = {
  schemaVersion: 1;
  chainId: 11155111;
  core: Address;
  coreDeploymentBlock: string;       // canonical uint encoded as decimal text
  assets: Array<{
    address: Address;
    symbol: string;
    decimals: number;
    roles: Array<"loan" | "collateral">;
    mock: boolean;
    mintable: boolean;
  }>;
  markets: Array<{
    key: string;
    id: Hex;
    loanToken: Address;
    collateralToken: Address;
    maturity: string;                // canonical uint encoded as decimal text
    strike: string;                  // canonical uint encoded as decimal text
    allowPartialRepay: boolean;
    gate: Address;
    visibility: "product" | "acceptance";
    executionPath: "core-erc20" | "vaultbtc-whole-lot";
    supportedLotAtoms: Array<string>; // non-empty for vaultbtc; empty for generic ERC-20
    liquiditySources: Array<"rfq" | "manager-pool">;
    manager: Address | null;
  }>;
  managers: Array<{
    address: Address;
    loanToken: Address;
    collateralToken: Address;
    deploymentBlock: string;
    managerAbiVersion: 1 | 2;        // 1 = USDC() legacy; 2 = loanToken()
    poolIds: Array<Hex>;              // each is a protocol market ID owned by this manager
  }>;
  defaultMarketId: Hex;              // candidate public default; must reference a product market
  manifestHash: Hex;
};
```

All unordered collections have one canonical order before RFC 8785 serialization:

- `assets`: lowercase address bytes ascending;
- `markets`: protocol market ID bytes ascending, then UTF-8 `key` ascending;
- `managers`: lowercase address bytes ascending;
- asset `roles`: `loan`, then `collateral`;
- market `liquiditySources`: `rfq`, then `manager-pool`;
- market `supportedLotAtoms`: numeric ascending after canonical-uint validation;
- manager `poolIds`: protocol market ID bytes ascending.

Addresses are normalized to lowercase bytes for comparison and sorting, then serialized in checksum
form. No generator may preserve discovery, receipt, object-map, or CLI argument order. The generator
rejects duplicate addresses, keys, IDs, manager memberships, or pool IDs and recomputes every protocol
market ID using the Core's exact six-field
`keccak256(abi.encode(MarketParams))`. It validates `chainId` and Core address as the containing
deployment namespace and constructs a qualified key separately; neither value is inserted into
`Bivium.computeId`.
`manifestHash` is SHA-256 of RFC 8785 canonical JSON with the `manifestHash` field omitted. The generator
writes both the JSON and hash; consumers never calculate a hash over arbitrary formatting.

The deployment pipeline supplies the **same byte-for-byte manifest** and expected hash to frontend,
solver, and keeper. Service-specific environment variables may provide secrets and RPC URLs, but may
not redefine token or market fields. Legacy `NEXT_PUBLIC_USDC`, `MOCK_USDC`, and manager-address variables
are migration inputs to the generator only, not independent runtime sources after this release.

Before enabling writes, every consumer validates:

1. schema version, expected manifest hash, `chainId`, and Core address;
2. every asset has code, on-chain `decimals()` equals the manifest, and its market use matches its role;
3. every protocol market ID recomputes exactly from its six manifest fields and the derived qualified
   key matches the canonical format above;
4. every manager's collateral and loan-token getters match the manifest using its declared ABI version;
5. every declared manager `poolId` resolves through `marketParams(poolId)`, recomputes to that ID, uses
   that manager as `gate`, and appears in exactly one manifest market with `manager-pool` liquidity;
6. every `manager-pool` market names its manager and membership; any market whose `gate` equals a
   declared manager names that manager even if only RFQ liquidity is enabled; all other markets use
   `manager: null`;
7. every market passes the arithmetic validator for its declared execution path and supported lot sizes;
8. `defaultMarketId` exists and has `visibility: "product"`.

Solver and keeper reject startup on failure. The frontend may render a configuration-error screen, but
keeps transaction actions disabled until validation succeeds. All three log the same `manifestHash` so
Sepolia acceptance can prove that they operated on one deployment definition.

## Deployment design

### Mock contracts

Deploy two separate mock tokens:

```text
MockGHO  = MockERC20Decimals("GHO", "GHO", 18)
MockUSDC = MockERC20Decimals("USD Coin", "USDC", 6)
```

Both expose the same test-only permissionless `mint(address,uint256)` surface currently expected by
the faucet and solver. Deployment and UI must label them as test assets.

### Market matrix

The initial Sepolia deployment creates the bounded matrix:

| Collateral | GHO | USDC |
|---|---|---|
| WETH | Required and default | Required |
| WBTC or vaultBTC | Enable only when that collateral deployment is live | Enable only when live |

For the same human strike, the raw strike differs by loan-token decimals. Deployment scripts must use
the decimal conversion formula rather than copy raw strike values between GHO and USDC markets:

```text
strike = humanPrice * 10^loanDecimals * 1e36 / 10^collateralDecimals
```

That formula constructs the market strike; it is not an assertion that every generic fill has an exact
integer collateral ratio. For newly issued Core debt, the actual escrow rule is:

```text
collateralRaw = ceil(issuedDebtRaw * STRIKE_SCALE / strike)
```

The ceiling rounds against the borrower, as documented by Core. Exact
`issuedDebtRaw * STRIKE_SCALE == collateralRaw * strike` equality is required only by an execution path
that explicitly validates dust-free sizes, such as `vaultbtc-whole-lot` below.

`humanPrice` is a **canonical decimal string**, never a JavaScript `number` or floating-point value.
Every implementation parses it into loan-token atoms first and then uses integer arithmetic only:

```text
priceAtoms = parseUnits(humanPrice, loanDecimals)
strike     = priceAtoms * 1e36 / 10^collateralDecimals
```

If the division has a remainder, configuration is rejected rather than rounded implicitly. Frontend
code must not use `Math.round(humanPrice * 10 ** loanDecimals)`; that expression exceeds
`Number.MAX_SAFE_INTEGER` for ordinary 18-decimal GHO prices and can derive a different market ID from
the deployment script.

One canonical golden-vector file defines at least WETH/GHO, WETH/USDC, vaultBTC/GHO, and
vaultBTC/USDC examples. Contract deployment tests generate the expected raw strikes, market IDs, and
Core `issuedDebtRaw -> ceil(collateralRaw)` results at both exact and rounding boundaries; frontend and
solver tests consume the same literal vectors and must produce byte-for-byte identical values. Runtime
quote display may use floating point, but no value derived through floating point may enter market
identity, calldata, signing, allowance, balance, or settlement calculations.

### Execution-path arithmetic

Decimal-correct strike construction is necessary but not sufficient. The manifest declares each
market's execution path, and the generator applies that path's admissibility rules.

For `core-erc20`, the generator and quote preview use the ceiling formula above. Golden vectors cover
both an exact division and a one-atom remainder at supported minimum and representative borrow sizes,
plus the corresponding repay calldata. UI collateral requirements must show the ceiling result rather
than reverse the displayed human strike with floating-point arithmetic.

For `vaultbtc-whole-lot`, configuration is accepted only when all of these hold:

```text
allowPartialRepay == false
collateral asset decimals == 8
strike % STRIKE_SCALE == 0

units = lotAtoms * strike / STRIKE_SCALE
units > 0
ceil(units * STRIKE_SCALE / strike) == lotAtoms
```

The last two equations are checked for every supported vault lot size declared by the deployment. They
mirror `BiviumVaultApp`'s whole-group escrow invariant and prevent a market that computes a valid ID but
reverts every actual vault borrow because of dust. Deployment, frontend, and solver consume the same
execution-path validator and literal lot/strike round-trip vectors. The solver's current six-decimal
loan-token restriction for vaultBTC must be replaced by these path rules so an 18-decimal GHO market is
accepted only when it is genuinely executable.

The script emits the versioned deployment manifest defined above. Downstream configuration must not
infer a GHO market by replacing the USDC address in an existing market.

### Pool isolation

`DualCurrencyPoolManager` instances remain single-loan-token managers:

```text
GHO manager  -> GHO deposits  -> GHO-denominated markets only
USDC manager -> USDC deposits -> USDC-denominated markets only
```

A manager must reject a market whose `loanToken` differs from its configured loan asset. GHO and USDC
must never share pool shares, deposit accounting, liquidity caps, or settlement accounting.

The current `USDC()` getter and internal USDC naming are legacy single-token assumptions. The canonical
`bivium-core` repository owns `DualCurrencyPoolManager`, even though consumers such as
`vault-contracts-bivium` receive it through a vendored dependency. The implementation therefore makes
an additive **periphery** change upstream and then updates the vendored revision:

```solidity
function loanToken() external view returns (IERC20) {
    return USDC;
}
```

Existing storage, constructor ABI, and `USDC()` remain unchanged for deployed-manager compatibility.
New deployments and callers use `loanToken()`. The manifest records `managerAbiVersion`; a caller may
fall back to `USDC()` only for a manifest-declared legacy manager, never because a generic
`loanToken()` call unexpectedly failed. Internal variable and event renaming is not required in this
release.

This changes a periphery contract in `bivium-core`; it does **not** change the Bivium Core market
contract, offer encoding, market hash, or settlement semantics.

## Frontend design

### Market catalog

Replace the single `USDC` constant as the only loan-token source with token and market records generated
from the validated manifest. Do not maintain a second hand-authored frontend market list. Construct
display rows from the manifest's exact market fields and hide `visibility: "acceptance"` rows from the
normal picker.

The selected `MarketRow.loan` is the only source for:

- symbol and icon;
- decimals and parsing;
- balances and allowances;
- approval target inputs;
- borrow proceeds and repayment labels;
- lend amount and claim previews;
- intent and offer market parameters.

Components must not use `MARKETS[0]` to determine a global loan symbol. Portfolio rows and order rows
derive metadata from their own market IDs. Unknown historical token addresses are displayed by shortened
address and raw decimals only after metadata is resolved safely; they must not be labelled GHO or USDC
by guesswork.

### Market selection

The Basic market picker adds a loan-asset selector above or alongside collateral selection:

```text
Borrow asset
  GHO   default
  USDC
```

Changing the loan asset selects a different market. It does not mutate an existing draft position in
place. Any amount, quote, approval state, or executable-depth result derived from the previous market is
cleared and recomputed.

Market keys include the loan symbol or address, for example `eth-gho` and `eth-usdc`. URL or local-state
selection must fall back deterministically to the configured default market when a referenced market is
not deployed.

### Faucet

The Sepolia faucet lists each configured mintable mock token separately:

```text
Get Mock GHO
Get Mock USDC
```

Each amount is parsed with that token's decimals. The faucet never exposes `mint` for a non-mock token,
even if a contract at that address happens to implement a compatible function.

### Copy

Replace transaction-path hard-coding such as “repaid in full, in USDC” with the selected market's
symbol. Protocol-level documentation may continue to use USDC in historical examples, but current
Sepolia instructions identify GHO as the default and disclose that both assets are mocks.

## Solver design

### Per-token capital state

Every solver job records its `loanToken`. Capital state is keyed by token address:

```text
wallet balance[token]
funded liquidity[token]
reserved capital[token]
minted today[token]
pending exposure[token]
daily settled volume[token]
allowance[token][spender]
```

Capital in one token cannot satisfy a requirement in another token.

The current solver discovers work by polling a paginated intent feed, so the first implementation uses
one **feed dispatcher Durable Object** plus **one execution coordinator and maker signer per loan
token**. It does not assume an HTTP request arrives for each intent.

The scheduled worker ticks the single dispatcher. The dispatcher is the only owner of the upstream feed
cursor and performs this durable ingestion transaction for every page:

1. parse each intent and recompute its protocol market ID;
2. map `(chainId, core, marketId)` to the validated manifest and obtain `loanToken`;
3. write a token-address-keyed outbox entry for each valid intent, or a reasoned quarantine entry for an
   invalid/unknown intent;
4. persist the next feed cursor in the same Durable Object transaction as those entries;
5. deliver pending outbox entries at least once to the execution coordinator keyed by
   `(chainId, core, loanToken)`.

Cursor advancement depends on durable local outbox/quarantine persistence, not on token-coordinator
health. A GHO circuit therefore cannot stop ingestion or dispatch of later USDC intents. Outbox delivery
is retryable and idempotent; the token coordinator acknowledges only after saving the job. Duplicate
delivery is absorbed by the qualified market-scoped intent key. A token coordinator with an open circuit
continues accepting jobs into its own durable queue but does not execute them until an audited reset.

The dispatcher has no signer, capital ledger, approval state, or execution circuit. A malformed intent
is quarantined with cursor, payload hash, and reason rather than retried forever. If the dispatcher
cannot persist the complete page atomically, it does not advance the cursor. A bounded outbox-size limit
may pause new feed ingestion as an infrastructure safeguard, but already-dispatched healthy token
coordinators continue processing their queues.

Each token execution coordinator owns its own:

- lifecycle store and job queue;
- circuit-breaker state and reset audit log;
- `RiskBook`, pending and daily-volume counters;
- mock-mint counters and policy;
- capital reservations and allowances;
- maker address, signer, and nonce sequence.

Distinct maker signers avoid cross-coordinator transaction-nonce races. A GHO coordinator may trip its
circuit without preventing the USDC coordinator from accepting or settling work. There is no process-
global capital circuit in the first implementation; only infrastructure failures that make request
validation impossible may cause the HTTP edge to reject all tokens.

Dispatcher and coordinator keys are derived from lowercase address bytes, never symbols. Job keys
include `chainId`, Core address, protocol market ID, and intent identity. A later shared-signer design
would require a separate global nonce allocator and is out of scope.

### Generic mock-mint action

Rename the semantic action from `mint_mock_usdc` to `mint_mock_token`. The action operates only on the
job's configured loan token and only when that token's registry entry has `mock=true` and
`mintable=true`.

Configuration changes from one global `MOCK_USDC` and one global maker to an explicit per-token map.
Each entry includes:

- token address;
- maker address and signer-secret binding;
- mint enabled flag;
- per-transaction mint cap;
- per-day mint cap.

The worker rejects ambiguous or missing mint policy. Production mode rejects every enabled mock-mint
entry.

### Quotes and signatures

The complete offer already commits to `loanToken`; no signature format changes are required. Quote
caches, consumption groups, idempotency keys, and lifecycle job keys must include the exact market ID so
that otherwise identical GHO and USDC requests cannot collide.

Pricing parameters may initially be shared, but funding APR, inventory spread, exposure limits, and
minimum order sizes are configurable per loan token. A future real-GHO market must not inherit USDC's
peg or liquidity assumptions implicitly.

## Keeper design

The keeper stops treating the manager's asset as semantically USDC. For each manager it reads the
generic configured loan token and its on-chain decimals, then derives human strike from the market:

```text
humanStrike = strike * 10^collateralDecimals
              / (1e36 * 10^loanDecimals)
```

The keeper validates that every pool returned by a manager uses that manager's loan token. It rejects
mixed-token pool configuration rather than publishing anchors using the wrong decimal scale.

Operationally, GHO and USDC managers may share one keeper process. Failures, last-poke status, and
metrics are keyed by manager and loan-token address so a GHO failure does not suppress healthy USDC
updates.

## Router and account behavior

A router may offer a later atomic refinance path:

```text
repay old GHO market -> release collateral -> open USDC market
```

That path is not part of this first implementation. The first version requires users to close and open
positions explicitly. EIP-7702 account automation may batch those existing actions later, but it cannot
change the loan token of an open market position.

## Aave v4 boundary

Mock GHO integration is an application and market test; it does not require or imply a Hub credit line.
Maker capital funds Bivium trades. A later Aave v4 adapter may keep idle GHO or USDC productive and pull
the selected token atomically when a matching offer fills.

Any such adapter is configured independently per Aave Hub asset ID and per Bivium loan-token address.
It must not use shared `Hub.draw` debt to absorb Bivium's collateral-delivery risk. Supporting real GHO
requires separate governance, token-address, liquidity, and risk review.

## Failure handling

The system fails closed in these cases:

- token metadata is missing or conflicts with on-chain decimals;
- the selected market is not deployed;
- a pool's loan token differs from its manager's configured token;
- a solver job references an unregistered token;
- mock minting is requested for a non-mock token;
- an approval or balance was calculated for a previously selected market;
- a quote, intent, or offer resolves to a different loan token than the current market;
- a claim token cannot be mapped to verified metadata.

A loan-token failure does not alter or pause unrelated markets in the Core. Frontend and automation may
disable the affected market while continuing to show independently healthy markets.

## Migration and compatibility

No on-chain Core migration is required. Existing USDC market IDs, offers, positions, credit, and claims
remain valid.

Migration occurs at the deployment and service layers:

1. add and test the upstream periphery `loanToken()` getter, then update the vendored revision;
2. deploy 18-decimal Mock GHO;
3. deploy every candidate product and short-lived acceptance manager/pool required by this release;
4. create every candidate GHO and retained USDC product market plus both acceptance markets;
5. generate one candidate manifest containing that complete topology, with the GHO product market as
   `defaultMarketId`;
6. deploy token-aware frontend, dispatcher, token coordinators, and keeper to a non-public candidate
   environment using the exact candidate manifest hash;
7. complete the real-transaction acceptance sequence and final product-market smoke checks against that
   same hash;
8. promote the byte-identical service builds and manifest to the public Preview without deploying a new
   market, changing `defaultMarketId`, or regenerating the manifest.

The existing public Preview continues using its prior USDC release until step 8. “GHO becomes default
only after acceptance” is therefore a release-promotion rule, not a post-acceptance mutation of the
candidate manifest. Any address, market, manager, default, or manifest change invalidates the evidence
and requires the full acceptance sequence to run again.

The manifest generator may read legacy environment variables during one transition release and must log
their deprecated mapping. Runtime frontend, solver, and keeper configuration must not read them as a
second source of token or market truth. No fallback may silently map a missing GHO field to USDC.

## Testing strategy

### Contract and deployment tests

- deploy Mock GHO with 18 decimals and Mock USDC with 6;
- create otherwise equivalent GHO and USDC markets and assert different market IDs;
- verify decimal-correct raw strikes represent the same human floor;
- verify Core collateral uses ceiling division at exact and one-atom-remainder boundaries;
- fund and execute one borrow, repay, and claim path in each token;
- verify cross-token repayment fails;
- verify GHO and USDC pool managers reject the other token's market;
- verify new managers expose `loanToken()` and legacy `USDC()` as the same address;
- verify balances and settlement remain conserved independently per token.

### Manifest tests

- generate identical canonical JSON and `manifestHash` from identical deployment facts;
- permute assets, markets, managers, roles, liquidity sources, and pool IDs and still generate identical
  canonical bytes and hash;
- reject duplicate asset addresses, market keys, market IDs, manager memberships, and pool IDs;
- reject a market ID that does not recompute from the exact manifest fields;
- derive canonical qualified-market-key golden vectors without storing a second manifest field;
- reject asset-role, asset-decimal, manager-token, or manager-pool disagreement against Sepolia
  read-only calls;
- reject an acceptance market as `defaultMarketId`;
- prove frontend, solver, and keeper fixtures consume the same manifest bytes and expected hash.

### Frontend tests

- show only markets with configured token and collateral addresses;
- default to GHO while allowing USDC selection;
- clear stale quote and approval state when loan token changes;
- parse and format GHO at 18 decimals and USDC at 6 decimals;
- reproduce every canonical raw-strike and market-ID golden vector without floating-point input;
- reject a vaultBTC lot/strike combination that fails the whole-lot round trip;
- build market parameters with the selected loan-token address;
- show token-correct borrow, repay, lend, faucet, portfolio, and claim copy;
- resolve multiple loan tokens in the same portfolio without using `MARKETS[0]` metadata;
- prevent mint controls for non-mock tokens.

### Solver tests

- persist a complete mixed-token feed page and its next cursor atomically into dispatcher outbox entries;
- replay outbox delivery without duplicating a token-coordinator job;
- quarantine an invalid or unknown-token intent without dropping later valid intents on the page;
- hold the feed cursor when page persistence fails, while allowing already-dispatched jobs to execute;
- reserve capital independently for GHO and USDC jobs;
- prevent GHO balance from funding a USDC job and vice versa;
- trip and reset the GHO coordinator circuit while the USDC coordinator continues settling;
- use distinct maker nonces without cross-token collisions;
- key lifecycle and idempotency records by exact market ID;
- mint only the job token under its own caps;
- enforce 18-decimal GHO and 6-decimal USDC atomic boundaries;
- accept an executable vaultBTC/GHO configuration and reject a dust-producing one;
- recover one token's failed job without corrupting the other token's reservations.

### Keeper tests

- derive the same human strike for equivalent GHO and USDC markets;
- reject manager/market loan-token mismatch;
- poke GHO and USDC pools independently;
- preserve service for one token when the other token's configuration fails.

### Sepolia acceptance

Acceptance uses dedicated, short-lived WETH/GHO and WETH/USDC markets whose maturity is fixed to the
same timestamp at least two hours after deployment. These are real Sepolia markets and transactions;
the test waits for wall-clock maturity and does not use fork time-warp. The two-hour minimum leaves at
least the manager's one-hour pricing floor plus operational time for funding and fills.

Acceptance-market keys include `acceptance` and their exact maturity so they cannot collide with or
become the default product markets. The deployment manifest marks them `visibility: "acceptance"`.
They are visible to the acceptance runner and Portfolio but excluded from the normal market picker.
Offer expiry must be before maturity. If either pre-maturity leg is not confirmed with at least 30
minutes remaining, both markets are abandoned and redeployed with a new maturity; the test must not
race the cutoff.

The acceptance artifact names distinct deployer/curator, GHO solver-maker, GHO borrower, USDC LP, and
USDC borrower accounts. It executes two explicit paths against the same Core:

### Path A — GHO RFQ solver, repaid road

1. The application mints Mock GHO needed for the solver's capital and the borrower's later repayment;
   the borrower wraps Sepolia ETH into WETH collateral.
2. The GHO solver dispatcher ingests a GHO intent, durably routes it to the GHO coordinator, and the GHO
   maker signs and funds an offer for the acceptance WETH/GHO market. No pool bid is included in this
   fill.
3. The borrower executes the solver offer through the intent settlement path and receives Mock GHO.
4. Assert the offer, intent, logs, balances, allowances, principal, face amount, strike, and APR all use
   the manifest's GHO address and 18-decimal scale.
5. Before the cutoff, the borrower repays the exact GHO face, withdraws the released WETH, and records
   the repayment and collateral-withdrawal transaction hashes.
6. After real Sepolia maturity, the solver-maker claims its credit and receives the expected GHO leg
   with zero delivered WETH for this isolated repaid position.

### Path B — USDC manager pool, delivered-collateral road

1. The application mints Mock USDC to the LP; the USDC borrower wraps Sepolia ETH into WETH collateral.
2. The curator creates the acceptance pool under the USDC manager. Assert its manifest membership,
   market ID, loan token, collateral token, strike, maturity, and gate.
3. The LP approves and calls `deposit`; assert ERC-6909 shares and Core liquidity. The curator calls
   `startPool`, closing subscription.
4. The keeper reads the USDC manager from the manifest and submits valid signed buy/sell anchors when
   `requireAnchor` is enabled. Assert the accepted ticks and freshness state on-chain.
5. The borrower takes the manager's exact `bid(poolId, units)` with no solver leg, receives 6-decimal
   Mock USDC, and leaves the position unpaid. Assert the LP requests redemption with `requestRedeem`
   while the pool is active.
6. After the same real Sepolia maturity, a permissionless caller invokes `settle(poolId)`. The LP calls
   `claimRedeem(poolId)` and receives the expected delivered WETH basket leg. Assert manager credit and
   liquidity were drained into the settlement basket and conservation holds within rounding dust.

Finally, Portfolio must show the GHO repaid history and USDC delivered history with distinct qualified
market keys. Logs from frontend, dispatcher, both token coordinators, and keeper must report the same
manifest hash and must never substitute one loan-token address for the other.

Before promotion, execute a minimum-size borrow followed by immediate repay and collateral withdrawal
on the final GHO **product** market selected by `defaultMarketId`. Verify its qualified key, execution
path, liquidity source, token precision, and transaction logs against the candidate manifest. Retained
USDC product markets receive a read/write smoke check appropriate to their enabled liquidity source.
These product checks do not wait for product maturity; the short-lived acceptance markets provide the
real maturity coverage.

Transaction hashes for every step, both protocol market IDs and qualified keys, pool ID, maturity,
observed block timestamps, pre/post balances, share supply, claimed amounts, and rounding dust are
recorded in a dated acceptance artifact. Fork tests remain required for fast regression coverage but
cannot substitute for this wall-clock sequence. The artifact records the candidate `manifestHash`,
service build identifiers, `defaultMarketId`, and product-smoke transaction hashes. Promotion must reuse
those exact artifacts; an acceptance market can never be the default picker market.

## Observability

Logs and metrics include `chainId`, `marketId`, `loanToken`, `loanSymbol`, and manager where applicable.
At minimum, monitor:

- quotes and fills by loan token;
- available, reserved, and funded capital by loan token;
- mock mint volume and cap failures by loan token;
- keeper poke success and age by manager and loan token;
- frontend configuration rejection by token address;
- repayment and settlement volume by loan token.

Addresses, not symbols, are used for aggregation keys.

## Documentation updates

After implementation and Sepolia acceptance:

- update the Development Preview introduction to name Mock GHO as the default loan token;
- update the app tutorial and faucet instructions for both mock assets;
- replace current-feature USDC-only language with market-specific loan-token language;
- retain explicit examples where USDC is intentionally the example asset;
- add a warning that Mock GHO has no value and is not production GHO;
- document that a position can only be repaid in its original loan token.

## Implementation boundaries by repository

| Repository | Required change |
|---|---|
| `bivium-core` | Add the backward-compatible `DualCurrencyPoolManager.loanToken()` periphery getter; no change to the Bivium Core market contract |
| `vault-contracts-bivium` | Update the pinned `bivium-core` vendor revision; deploy Mock GHO and token-correct markets/managers |
| `bivium-frontend` | Multi-token catalog and selection; token-derived amounts, approvals, faucet, portfolio, and copy |
| `bivium-solver` | Token-keyed capital lifecycle, generic mock mint policy, token-specific risk configuration |
| `bivium-keeper` | Generic loan-token discovery, decimal conversion, token-keyed operation and metrics |
| `bivium-docs` | Publish user-facing behavior only after deployment acceptance |
| `vault-contracts-zcb-aave-v4` | No first-release Hub credit integration; update only adapters or deployment notes actually used by the accepted Sepolia path |

## Acceptance criteria

The feature is complete when:

- the same Core deployment supports live GHO and USDC market IDs;
- Mock GHO uses 18 decimals and Mock USDC uses 6 decimals;
- every user and automation action derives token address, symbol, and decimals from its exact market;
- frontend, solver, and keeper validate and report the same canonical manifest hash;
- protocol market IDs match the Core six-field hash while off-chain records use the deployment-qualified
  `(chainId, core, id)` namespace;
- GHO and USDC capital, pools, caps, lifecycle records, and settlements are isolated;
- mixed-token feed ingestion cannot skip one token or let one token's circuit block the other's queue;
- a GHO solver circuit failure does not stop the USDC coordinator;
- existing USDC positions and flows continue to work;
- no Core contract or market-hash format changes;
- the full contract, frontend, solver, and keeper test suites pass;
- the Sepolia real-transaction acceptance sequence passes;
- final GHO and retained USDC product-market smoke transactions pass under the candidate manifest;
- the public promotion uses the exact accepted manifest hash and service build identifiers;
- GHO is made the public default only after acceptance evidence is recorded.
