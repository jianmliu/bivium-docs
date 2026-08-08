# Multi-Loan-Token Sepolia Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Run GHO- and USDC-denominated Bivium markets on one Sepolia Core, with token-isolated automation, a canonical deployment manifest, a GHO-default frontend, and real repay/deliver acceptance evidence.

**Architecture:** Keep `Bivium` and its six-field `computeId` unchanged. Add a backward-compatible generic getter to the pool-manager periphery, generate one canonical manifest for all assets/markets/managers, and make frontend, solver, and keeper consume that manifest. Solver feed ingestion is separated from token-specific execution; GHO and USDC use different coordinators, signers, risk books, and circuits.

**Tech Stack:** Solidity 0.8.28/Foundry, TypeScript, viem, Cloudflare Workers/Durable Objects, Next.js, Node test runner, Sepolia.

---

## Repository and branch boundaries

Create an isolated `agent/multi-loan-token` worktree in every modified repository before executing its first task. Never modify the existing dirty `bivium-frontend` checkout.

| Repository | Work |
|---|---|
| `bivium-core` | Add `DualCurrencyPoolManager.loanToken()` and tests |
| `vault-contracts-bivium` | Update pinned Core; exact arithmetic helpers; Mock GHO deployment; manifest generator and acceptance scripts |
| `bivium-solver` | Manifest loading; dispatcher outbox; per-token coordinators, signers, limits, minting |
| `bivium-keeper` | Manifest loading; multi-manager/token scheduling and decimal-safe strike conversion |
| `bivium-frontend` | Manifest-backed catalog; multi-manager market runtime; GHO/USDC selection and faucet/copy |
| `bivium-docs` | Track this plan, acceptance artifact, and publish user docs after acceptance |

## Shared release gates

Do not advance to the next phase unless the current phase's tests pass and its repository has a focused commit. Do not deploy candidate contracts until all local and Sepolia-fork tests pass. Do not promote the Preview unless the candidate manifest hash and build IDs match the real-transaction acceptance artifact.

### Task 1: Add the generic pool-manager loan-token getter upstream

**Files:**
- Modify: `bivium-core/src/vaults/DualCurrencyPoolManager.sol`
- Modify: `bivium-core/test/vaults/DualCurrencyPoolManager.t.sol`

- [ ] **Step 1: Write the failing compatibility test**

Add to the existing manager test contract:

```solidity
function test_loanToken_matchesLegacyUsdcGetter() public view {
    assertEq(address(manager.loanToken()), address(usdc));
    assertEq(address(manager.loanToken()), address(manager.USDC()));
}
```

- [ ] **Step 2: Run the focused test and verify it fails**

Run:

```bash
forge test --match-test test_loanToken_matchesLegacyUsdcGetter -vvv
```

Expected: compile failure because `loanToken()` does not exist.

- [ ] **Step 3: Add the additive getter without changing storage or constructor ABI**

Add beside the immutable getters:

```solidity
function loanToken() external view returns (IERC20) {
    return USDC;
}
```

- [ ] **Step 4: Run manager and full Core tests**

```bash
forge test --match-path test/vaults/DualCurrencyPoolManager.t.sol -vvv
forge test
```

Expected: all tests pass; no storage, constructor, or `USDC()` ABI removal.

- [ ] **Step 5: Commit upstream**

```bash
git add src/vaults/DualCurrencyPoolManager.sol test/vaults/DualCurrencyPoolManager.t.sol
git commit -m "feat: expose generic pool loan token"
```

### Task 2: Update the consumer pin and add arithmetic golden vectors

**Files:**
- Modify: `vault-contracts-bivium/lib/bivium-core` gitlink
- Create: `vault-contracts-bivium/script/lib/MultiLoanMath.sol`
- Create: `vault-contracts-bivium/test/MultiLoanMath.t.sol`
- Create: `vault-contracts-bivium/deployments/fixtures/multi-loan-golden-v1.json`

- [ ] **Step 1: Update the Core submodule to Task 1's reviewed commit**

Checkout the exact upstream commit inside `lib/bivium-core`; record it with:

```bash
git -C lib/bivium-core rev-parse HEAD
```

- [ ] **Step 2: Write failing exact-math tests**

Create tests covering:

```solidity
function test_strikeFromHumanAtoms_wethGho() public {
    assertEq(MultiLoanMath.strike(3000e18, 18), 3000e36);
}

function test_collateralForDebt_roundsUp() public {
    uint256 strike = 3000e36;
    assertEq(MultiLoanMath.collateralForDebt(1e18, strike), 333333333333334);
}

function test_vaultBtcRejectsOffGridStrike() public {
    vm.expectRevert(MultiLoanMath.NonWholeLotStrike.selector);
    MultiLoanMath.validateVaultBtc(1e36 + 1, 1e8);
}
```

- [ ] **Step 3: Run the tests and verify missing-library failure**

```bash
forge test --match-path test/MultiLoanMath.t.sol -vvv
```

- [ ] **Step 4: Implement pure integer helpers**

```solidity
library MultiLoanMath {
    uint256 internal constant SCALE = 1e36;
    error NonIntegralStrike();
    error NonWholeLotStrike();

    function strike(uint256 priceAtoms, uint8 collateralDecimals) internal pure returns (uint256) {
        uint256 denominator = 10 ** collateralDecimals;
        if (mulmod(priceAtoms, SCALE, denominator) != 0) revert NonIntegralStrike();
        return priceAtoms * SCALE / denominator;
    }

    function collateralForDebt(uint256 debtAtoms, uint256 strike_) internal pure returns (uint256) {
        return (debtAtoms * SCALE + strike_ - 1) / strike_;
    }

    function validateVaultBtc(uint256 strike_, uint256 lotAtoms) internal pure returns (uint256 units) {
        if (strike_ % SCALE != 0) revert NonWholeLotStrike();
        units = lotAtoms * strike_ / SCALE;
        if (units == 0 || collateralForDebt(units, strike_) != lotAtoms) revert NonWholeLotStrike();
    }
}
```

Use `Math.mulDiv` if fuzz tests show multiplication can overflow for supported bounds.

- [ ] **Step 5: Generate literal GHO/USDC × WETH/vaultBTC vectors**

The JSON must contain canonical decimal strings for `priceAtoms`, `strike`, representative `debtAtoms`, ceiling `collateralAtoms`, and vault lot round trips. Do not generate any identity value through JavaScript `number`.

- [ ] **Step 6: Run tests and commit**

```bash
forge test --match-path test/MultiLoanMath.t.sol -vvv
forge test
git add lib/bivium-core script/lib/MultiLoanMath.sol test/MultiLoanMath.t.sol deployments/fixtures/multi-loan-golden-v1.json
git commit -m "feat: add exact multi-loan market arithmetic"
```

### Task 3: Define and validate the canonical deployment manifest

**Files:**
- Create: `vault-contracts-bivium/scripts/manifest/schema.ts`
- Create: `vault-contracts-bivium/scripts/manifest/canonical.ts`
- Create: `vault-contracts-bivium/scripts/manifest/generate.ts`
- Create: `vault-contracts-bivium/scripts/manifest/verify.ts`
- Create: `vault-contracts-bivium/scripts/manifest/manifest.test.ts`
- Create: `vault-contracts-bivium/deployments/sepolia/.gitkeep`
- Create: `vault-contracts-bivium/package.json`
- Create: `vault-contracts-bivium/tsconfig.json`
- Create: `vault-contracts-bivium/package-lock.json`

- [ ] **Step 1: Bootstrap the minimal manifest toolchain and add failing tests**

Create a private Node package using pinned versions of TypeScript, `tsx`, `viem`, an RFC 8785/JCS implementation, and the Node type definitions. Add `test` and `typecheck` scripts; commit the generated lockfile. Do not add application/runtime dependencies unrelated to deployment tooling.

Tests must assert:

```ts
assert.equal(qualifiedMarketKey(11155111, CORE, ID),
  `eip155:11155111/${CORE.toLowerCase()}/${ID.toLowerCase()}`);
assert.equal(hashManifest(permutationA), hashManifest(permutationB));
assert.throws(() => validateManifest(duplicatePoolId));
assert.throws(() => validateManifest(acceptanceAsDefault));
```

Also permute assets, markets, managers, roles, liquidity sources, lot sizes, and pool IDs.

- [ ] **Step 2: Run the manifest tests and verify they fail**

```bash
npm test -- scripts/manifest/manifest.test.ts
```

- [ ] **Step 3: Implement `DeploymentManifestV1` and derived qualified keys**

Use the exact schema in the approved spec. `qualifiedMarketKey` is derived only:

```ts
export const qualifiedMarketKey = (chainId: number, core: Address, id: Hex) =>
  `eip155:${chainId}/${core.toLowerCase()}/${id.toLowerCase()}` as const;
```

- [ ] **Step 4: Implement complete canonical sorting and RFC 8785 hashing**

Normalize all addresses before sorting; sort every array defined by the spec; hash canonical JSON with `manifestHash` omitted; then write the final object with the hash inserted.

- [ ] **Step 5: Implement chain-backed verification**

Using viem, verify chain ID, Core bytecode, token decimals, six-field `computeId`, manager getter selected by `managerAbiVersion`, manager collateral, each `poolId`, `marketParams(poolId)`, gate, execution-path constraints, and default visibility.

- [ ] **Step 6: Run tests and commit**

```bash
npm test -- scripts/manifest/manifest.test.ts
npm run typecheck
git add package.json package-lock.json tsconfig.json scripts/manifest deployments/sepolia/.gitkeep
git commit -m "feat: add canonical deployment manifest"
```

### Task 4: Deploy Mock GHO and the complete candidate topology

**Files:**
- Create: `vault-contracts-bivium/script/DeployMultiLoanSepolia.s.sol`
- Create: `vault-contracts-bivium/test/DeployMultiLoanSepolia.t.sol`
- Reuse: `vault-contracts-bivium/test/mocks/MockERC20.sol`
- Create during deployment: `vault-contracts-bivium/deployments/sepolia/multi-loan-v1.json`

- [ ] **Step 1: Write a failing deployment test**

The test must deploy `MockERC20Decimals("Mock GHO","GHO",18)` and retained 6-decimal Mock USDC, then assert WETH/GHO and WETH/USDC product markets, two short-lived acceptance markets, separate managers, correct gates, and GHO product `defaultMarketId`.

- [ ] **Step 2: Run the focused test and verify failure**

```bash
forge test --match-path test/DeployMultiLoanSepolia.t.sol -vvv
```

- [ ] **Step 3: Implement the deployment script with explicit phases**

The script must:

```text
deploy assets -> deploy managers -> create product pools -> create +2h acceptance pools
-> seed only configured pool liquidity -> emit receipt facts -> stop
```

Use `vm.envString` canonical price strings parsed into atoms; do not accept floating-point shell values.

- [ ] **Step 4: Generate the manifest from receipts and read-only calls**

Run the TypeScript generator after broadcast. It must not accept hand-authored addresses as final facts.

- [ ] **Step 5: Run contract tests and a Sepolia-fork dry run**

```bash
forge test
forge script script/DeployMultiLoanSepolia.s.sol:DeployMultiLoanSepolia \
  --fork-url "$SEPOLIA_RPC_URL" -vvv
```

Expected: no broadcast; generated candidate topology validates on the fork.

- [ ] **Step 6: Commit deployment support, not live addresses yet**

```bash
git add script/DeployMultiLoanSepolia.s.sol test/DeployMultiLoanSepolia.t.sol
git commit -m "feat: deploy multi-loan Sepolia topology"
```

### Task 5: Add manifest-backed solver configuration

**Files:**
- Create: `bivium-solver/src/manifest.ts`
- Create: `bivium-solver/src/manifest.test.ts`
- Modify: `bivium-solver/src/config.ts`
- Modify: `bivium-solver/src/config.test.ts`
- Modify: `bivium-solver/src/executionPort.ts`
- Modify: `bivium-solver/src/pricePreflight.test.ts`

- [ ] **Step 1: Write failing manifest and 18-decimal vaultBTC tests**

Assert config rejects a hash mismatch and token metadata overrides. Replace the current expectation that vaultBTC requires six loan decimals with:

```ts
assert.doesNotThrow(() => validateExecutionPath(vaultBtcGhoMarket));
assert.throws(() => validateExecutionPath({ ...vaultBtcGhoMarket, strike: OFF_GRID }));
```

- [ ] **Step 2: Run tests and verify failures**

```bash
npm test -- src/manifest.test.ts src/config.test.ts src/pricePreflight.test.ts
```

- [ ] **Step 3: Load one immutable manifest and per-token secret policy**

Remove `MARKETS_JSON` and `MOCK_USDC` as runtime market sources. Parse `DEPLOYMENT_MANIFEST_JSON` plus `EXPECTED_MANIFEST_HASH`; keep RPC URLs and signer secrets service-specific. Resolve the signer secret by lowercase loan-token address and verify it derives the configured maker.

- [ ] **Step 4: Replace six-decimal vaultBTC restriction with shared path validation**

Use literal golden vectors and integer-string parsing. Never convert identity or atomic amounts to `number`.

- [ ] **Step 5: Run solver checks and commit**

```bash
npm run check
git add src/manifest.ts src/manifest.test.ts src/config.ts src/config.test.ts src/executionPort.ts src/pricePreflight.test.ts
git commit -m "feat: load multi-loan solver manifest"
```

### Task 6: Split solver ingestion from token execution

**Files:**
- Create: `bivium-solver/src/dispatcher.ts`
- Create: `bivium-solver/src/dispatcher.test.ts`
- Modify: `bivium-solver/src/coordinator.ts`
- Modify: `bivium-solver/src/coordinator.test.ts`
- Modify: `bivium-solver/src/lifecycle.ts`
- Modify: `bivium-solver/src/lifecycle.test.ts`
- Modify: `bivium-solver/src/risk.ts`
- Modify: `bivium-solver/src/index.ts`
- Modify: `bivium-solver/src/index.test.ts`

- [ ] **Step 1: Write dispatcher transaction and replay tests**

Cover mixed GHO/USDC pages, atomic cursor+outbox persistence, invalid-intent quarantine, delivery replay, open GHO circuit with continuing USDC execution, and failed-page persistence holding the cursor.

- [ ] **Step 2: Run focused tests and verify failures**

```bash
npm test -- src/dispatcher.test.ts src/coordinator.test.ts src/lifecycle.test.ts src/index.test.ts
```

- [ ] **Step 3: Implement the dispatcher Durable Object**

Persist `feed_cursor`, `outbox`, and `quarantine` in one SQLite transaction. Delivery is at least once. Mark an outbox row delivered only after the target token coordinator durably acknowledges the idempotency key.

- [ ] **Step 4: Key execution coordinators by chain, Core, and loan token**

Each coordinator owns its own lifecycle store, circuit, `RiskBook`, mint counters, signer, and nonce sequence. An open circuit accepts durable jobs but does not execute them.

- [ ] **Step 5: Rename the generic mint action**

Change `mint_mock_usdc` to `mint_mock_token` across lifecycle types, SQL JSON records, metrics, and tests. Add an explicit one-time migration for unfinished legacy jobs or reject startup when such jobs exist; do not silently reinterpret them.

- [ ] **Step 6: Run full checks and commit**

```bash
npm run check
git add src
git commit -m "feat: isolate solver execution by loan token"
```

### Task 7: Make keeper scheduling manifest- and manager-aware

**Files:**
- Create: `bivium-keeper/src/manifest.ts`
- Create: `bivium-keeper/src/manifest.test.ts`
- Modify: `bivium-keeper/src/index.ts`
- Modify: `bivium-keeper/src/config.test.ts`
- Modify: `bivium-keeper/src/quote.test.ts`
- Modify: `bivium-keeper/wrangler.toml`

- [ ] **Step 1: Write failing multi-manager tests**

Test manifest hash rejection, `managerAbiVersion` dispatch, GHO 18-decimal and USDC 6-decimal strike conversion, pool membership mismatch, and one manager failure with the other still poked.

- [ ] **Step 2: Run tests and verify failures**

```bash
npm test
```

- [ ] **Step 3: Replace global manager/USDC configuration**

Iterate manifest managers and sorted `poolIds`; call `loanToken()` for ABI v2 and `USDC()` only for declared ABI v1. Read on-chain decimals and calculate strike as an exact rational converted to display `number` only at the final pricing-model boundary.

- [ ] **Step 4: Isolate operational failures**

Catch, log, and metric failures by `(manager, loanToken, poolId)`; continue the outer loop. Global failure is reserved for invalid manifest or unavailable canonical chain state.

- [ ] **Step 5: Run dry-run checks and commit**

```bash
npm test
npm run check
git add src wrangler.toml
git commit -m "feat: quote multiple loan-token managers"
```

### Task 8: Build the frontend manifest runtime and multi-manager market model

**Files:**
- Create: `bivium-frontend/lib/deploymentManifest.ts`
- Create: `bivium-frontend/lib/deploymentManifest.test.ts`
- Modify: `bivium-frontend/lib/markets.ts`
- Modify: `bivium-frontend/lib/marketList.ts`
- Modify: `bivium-frontend/lib/poolManager.ts`
- Modify: `bivium-frontend/lib/bivium.ts`
- Modify: `bivium-frontend/lib/marketList.test.ts`
- Modify: `bivium-frontend/lib/poolManager.test.ts`

- [ ] **Step 1: Write failing manifest/catalog tests**

Test exact hash, asset roles, qualified keys, hidden acceptance rows, GHO default, two managers, RFQ-only and manager-pool liquidity sources, and unknown historical-token fail-closed behavior.

- [ ] **Step 2: Run focused tests and verify failures**

```bash
npm test -- lib/deploymentManifest.test.ts lib/marketList.test.ts lib/poolManager.test.ts
```

- [ ] **Step 3: Implement one validated runtime descriptor**

```ts
type MarketRuntime = {
  manifest: ManifestMarket;
  qualifiedKey: string;
  loan: AssetInfo;
  collateral: AssetInfo;
  manager?: Address;
  poolId?: Hex;
};
```

Remove global `NEXT_PUBLIC_USDC` and `NEXT_PUBLIC_DUAL_POOL_MANAGER` as market truth. Build all runtime rows from the bundled candidate manifest and expected hash.

- [ ] **Step 4: Remove floating-point identity calculations**

Replace `strikeFromFloor(number,...)` with canonical decimal-string/BigInt parsing. Use the shared golden literals for raw strike, Core ceiling collateral, and vaultBTC whole-lot rejection.

- [ ] **Step 5: Run tests and commit**

```bash
npm test -- lib/deploymentManifest.test.ts lib/marketList.test.ts lib/poolManager.test.ts
git add lib
git commit -m "feat: load manifest-backed loan markets"
```

### Task 9: Make frontend actions token- and liquidity-source-aware

**Files:**
- Modify: `bivium-frontend/components/ActionDock.tsx`
- Modify: `bivium-frontend/components/Faucet.tsx`
- Modify: `bivium-frontend/components/PortfolioScreen.tsx`
- Modify: `bivium-frontend/components/TradeManager.tsx`
- Modify: `bivium-frontend/components/AppScreen.tsx`
- Create: `bivium-frontend/lib/loanAssetSelection.ts`
- Create: `bivium-frontend/lib/loanAssetSelection.test.ts`

- [ ] **Step 1: Write failing selection/reset tests**

Test GHO default, USDC switch, stale amount/quote/allowance clearing, per-market symbols and decimals, per-manager approvals, RFQ-only actions omitting pool calls, and pool deposits appearing only for `manager-pool` markets.

- [ ] **Step 2: Run tests and verify failures**

```bash
npm test -- lib/loanAssetSelection.test.ts
```

- [ ] **Step 3: Add the loan-asset selector and key state by qualified market**

Changing token must remount/reset the action surface. Never derive copy from `MARKETS[0]`. Select manager, gate, allowance token, faucet token, quote book, and claim metadata from `MarketRuntime`.

- [ ] **Step 4: Generalize the faucet and copy**

Render `Get Mock GHO` and `Get Mock USDC` only for `mock && mintable`. Parse each input with its own decimals. Replace transaction-path USDC strings with `market.loan.symbol`; retain explicit historical examples only in docs.

- [ ] **Step 5: Run full frontend validation and commit**

```bash
npm test
npm run check:boundaries
npm run build
git add components lib
git commit -m "feat: trade GHO and USDC loan markets"
```

### Task 10: Build fork and real-Sepolia acceptance runners

**Files:**
- Create: `vault-contracts-bivium/script/AcceptMultiLoanSepolia.s.sol`
- Create: `vault-contracts-bivium/test/MultiLoanSepoliaFork.t.sol`
- Create: `vault-contracts-bivium/scripts/acceptance/run-multi-loan.ts`
- Create: `vault-contracts-bivium/scripts/acceptance/verify-artifact.ts`
- Create after run: `bivium-docs/acceptance/YYYY-MM-DD-multi-loan-sepolia.md`

- [ ] **Step 1: Write the fork acceptance test**

Exercise GHO RFQ borrow→repay→withdraw→claim and USDC manager deposit→start→anchor→borrow→requestRedeem→settle→claimRedeem. Warp only in the fork test and assert token/market/basket conservation.

- [ ] **Step 2: Run the fork test and fix all failures before broadcast**

```bash
forge test --match-path test/MultiLoanSepoliaFork.t.sol -vvv
```

- [ ] **Step 3: Implement the real-chain state machine**

The runner persists a JSON journal after every transaction, refuses to race maturity with less than 30 minutes remaining, waits for wall-clock Sepolia maturity, verifies canonical receipts, and can resume without repeating finalized actions.

- [ ] **Step 4: Broadcast the complete candidate topology and generate the final candidate manifest**

Use a dedicated candidate branch/build. Record deployment hashes and verify the generated manifest against two independent RPC endpoints.

- [ ] **Step 5: Deploy candidate frontend, solver, and keeper builds**

All services must log the same manifest hash and immutable build IDs before acceptance funds move.

- [ ] **Step 6: Execute Path A, Path B, and product smoke transactions**

Run the exact actor/transaction sequence from the spec. Do not substitute fork evidence for real maturity.

- [ ] **Step 7: Generate and verify the acceptance artifact**

The artifact includes every transaction hash, protocol ID, qualified key, pool ID, maturity/block timestamps, balances, shares, claims, rounding dust, manifest hash, build IDs, and final pass/fail. Verification must fail if any public-promotion artifact differs.

- [ ] **Step 8: Commit acceptance evidence**

```bash
git add acceptance/YYYY-MM-DD-multi-loan-sepolia.md
git commit -m "test: accept multi-loan Sepolia release"
```

### Task 11: Publish the accepted Preview and user documentation

**Files:**
- Modify: `bivium-docs/README.md`
- Modify: `bivium-docs/using-the-app.md`
- Modify: `bivium-docs/faq.md`
- Modify: `bivium-docs/protocol-overview.md`
- Modify: `bivium-docs/SUMMARY.md` only if a new page is added

- [ ] **Step 1: Promote byte-identical accepted artifacts**

Confirm candidate and public manifest hashes/build IDs match before changing traffic. Do not regenerate the manifest during promotion.

- [ ] **Step 2: Update current-product documentation**

State that Mock GHO is the default Sepolia loan token, Mock USDC remains available, each position must be repaid in its original token, and both test assets have no value. Keep P/N interpretation separate from GHO identity.

- [ ] **Step 3: Validate documentation and release state**

```bash
git diff --check
rg -n "Mock GHO|Mock USDC|original loan token" README.md using-the-app.md faq.md protocol-overview.md
```

Expected: current workflows name both tokens and contain no production-GHO claim.

- [ ] **Step 4: Commit docs**

```bash
git add README.md using-the-app.md faq.md protocol-overview.md SUMMARY.md
git commit -m "docs: publish multi-loan Sepolia Preview"
```

## Final verification

- [ ] Every modified repository has a clean worktree and reviewed branch.
- [ ] Core market contract bytecode and six-field `computeId` are unchanged.
- [ ] All Foundry suites pass.
- [ ] Solver `npm run check` passes.
- [ ] Keeper `npm test && npm run check` passes.
- [ ] Frontend tests, boundary checks, and build pass.
- [ ] Two-RPC manifest verification passes with the accepted hash.
- [ ] Real Sepolia GHO repaid road, USDC delivered road, and final product smokes are recorded.
- [ ] Public artifacts are byte-identical to accepted candidate artifacts.
