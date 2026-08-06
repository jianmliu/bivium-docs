# App Tutorial and FAQ Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Restructure the Bivium GitBook into a linear application tutorial plus a focused FAQ that document only workflows enabled in the Development Preview.

**Architecture:** Keep procedural instructions in `using-the-app.md`, move recurring conceptual and lifecycle questions into a new `faq.md`, and connect both pages through GitBook navigation and the introduction. Treat frontend `origin/dev`, protocol lifecycle rules, and current Preview configuration as the source of truth; use conditional wording for active markets, executable quotes, partial repayment, and post-maturity secondary transfers.

**Tech Stack:** GitBook, Markdown, Git, ripgrep

---

### Task 1: Rewrite the end-to-end application tutorial

**Files:**
- Modify: `using-the-app.md`

- [ ] **Step 1: Establish prerequisites and product terms**

Open with the Development Preview boundary, Sepolia/test-asset requirements, wallet and Header domain checks. Define loan token, represented BTC/ETH collateral, strike, maturity, APR, face, and dual-currency note (DCN) before giving action steps.

- [ ] **Step 2: Add the Borrow sequence**

Document this conditional sequence:

```text
Choose an active configured market → enter amount → review APR, proceeds, face,
collateral, strike, and maturity → approve collateral if requested → Borrow →
verify the Loan row in Portfolio.
```

State that an active market and executable quote are prerequisites and that a configured market does not guarantee liquidity.

- [ ] **Step 3: Add Loan management, Repay, and Reclaim sequences**

Explain debt, locked collateral, lifecycle state, and maturity in Portfolio. State that repayment is allowed strictly before maturity. Document loan-token approval when requested, Repay, and the separate Reclaim action when the UI exposes reclaimable collateral.

- [ ] **Step 4: Add Lend and DCN sequence**

Document choosing an active market with an executable ask, reviewing the terms, using Lend now, and verifying the DCN row. Explain that the result is market-domain-bound credit, not a guaranteed yield product.

- [ ] **Step 5: Add Pro Trade and Order management sequence**

Document exact-market selection, taking available liquidity or placing an enabled order, reading the shared positions tape, and cancelling a resting Order. Qualify post-maturity secondary transfers as possible only when offer timing, available credit, and other execution conditions allow.

- [ ] **Step 6: Add matured DCN Claim sequence**

Document Estimated proceeds, separate loan-token/collateral-token estimates, Claim, and execution-time final state. State that a failed preview is unknown rather than zero.

- [ ] **Step 7: Add unavailable states and focused next links**

Explain matured markets, missing quotes, relayer unavailability, and Portfolio domain-validation failure. Link to `faq.md`, `protocol-overview.md`, and `security.md`.

- [ ] **Step 8: Audit and commit the tutorial**

Run:

```bash
rg -ni '\b(pool|pools|lp|vault|vaults|curator|curators|subscription|auto-repay|penalty interest|margin call|liquidation ltv|earn account)\b' using-the-app.md
rg -n '0x[a-fA-F0-9]{40}' using-the-app.md
git diff --check -- using-the-app.md
```

Expected: no unavailable/Binance-specific feature claims, no contract address, and no whitespace errors.

Commit:

```bash
git add using-the-app.md
git commit -m "docs: expand the Bivium app tutorial"
```

### Task 2: Create the Bivium FAQ

**Files:**
- Create: `faq.md`

- [ ] **Step 1: Add lifecycle and liquidation questions**

Answer whether Bivium uses LTV/margin calls/price-triggered liquidation, when repayment is allowed, and what happens to unpaid debt at maturity. Describe repay-or-deliver without importing overdue interest or post-term liquidation behavior.

- [ ] **Step 2: Add pricing and DCN questions**

Explain that APR is derived from ratified offer price and term, not a protocol service fee or guaranteed return. Define DCN fungibility within the exact market domain and conditionally explain secondary transfer, including post-maturity transfer of existing credit.

- [ ] **Step 3: Add Claim questions**

Explain the two possible claim assets, Estimated proceeds uncertainty, and execution-time state.

- [ ] **Step 4: Add data availability and relayer questions**

Explain why Portfolio unavailable is not an empty account and why relayer failure affects discovery/liveness rather than core custody. Preserve the client's and user's market-domain validation responsibility.

- [ ] **Step 5: Add market configuration and release questions**

Explain that supported assets/markets come from the Preview's configured list, partial repayment is an exact-market parameter rather than a global promise, real funds must not be used, and legacy offers/positions do not migrate.

- [ ] **Step 6: Add focused links and audit the FAQ**

Link to `using-the-app.md`, `protocol-overview.md`, and `security.md`. Run:

```bash
rg -c '^## ' faq.md
rg -ni '\b(pool|pools|lp|vault|vaults|curator|curators|subscription|auto-repay|penalty interest|liquidation ltv|earn account)\b' faq.md
rg -n '0x[a-fA-F0-9]{40}' faq.md
git diff --check -- faq.md
```

Expected: exactly fourteen FAQ question headings, no unavailable/Binance-specific feature claim, no contract address, and no whitespace errors.

Commit:

```bash
git add faq.md
git commit -m "docs: add the Bivium FAQ"
```

### Task 3: Integrate the pages into GitBook navigation

**Files:**
- Modify: `SUMMARY.md`
- Modify: `README.md`

- [ ] **Step 1: Add the FAQ sidebar entry**

Add exactly this entry immediately after Using the app and before Protocol overview:

```markdown
* [FAQ](faq.md)
```

- [ ] **Step 2: Add the introduction link**

Under `Where to start`, add a concise FAQ bullet describing lifecycle, claims, data availability, and release limitations. Preserve the Development Preview warning.

- [ ] **Step 3: Verify navigation and commit**

Run:

```bash
sed -n '1,20p' SUMMARY.md
rg -n 'FAQ|faq.md' README.md SUMMARY.md using-the-app.md
git diff --check -- README.md SUMMARY.md
```

Expected: FAQ follows Using the app in the sidebar, all links use `faq.md`, and no whitespace errors appear.

Commit:

```bash
git add README.md SUMMARY.md
git commit -m "docs: add the FAQ to GitBook"
```

### Task 4: Validate the complete published documentation

**Files:**
- Verify: `README.md`
- Verify: `SUMMARY.md`
- Verify: `bitcoin-credit-markets.md`
- Verify: `using-the-app.md`
- Verify: `faq.md`
- Verify: `protocol-overview.md`
- Verify: `security.md`

- [ ] **Step 1: Validate every local Markdown link**

Extract relative `.md` destinations from all seven published pages and confirm each target exists.

Expected: `all local Markdown links exist`.

- [ ] **Step 2: Audit unavailable and borrowed product behavior**

Run:

```bash
rg -ni '\b(pool|pools|lp|curator|curators|subscription|auto-repay|penalty interest|liquidation ltv|earn account|repay with other)\b' README.md bitcoin-credit-markets.md using-the-app.md faq.md protocol-overview.md security.md
```

Expected: no unavailable feature or Binance-specific behavior is presented as Bivium functionality. Core accounting phrases and external collateral-wrapper risks are acceptable only when clearly distinguished from product availability.

- [ ] **Step 3: Audit lifecycle and economic claims**

Confirm that repayment is strictly pre-maturity; new debt and repayment use applicable lifecycle constraints; existing-credit secondary transfer is conditional; APR is not a service fee or guarantee; Estimated proceeds are not guaranteed; and there is no LTV-triggered liquidation.

- [ ] **Step 4: Audit deployment claims and addresses**

Run:

```bash
rg -ni 'production|dev\.bivium\.pages\.dev|0x[a-fA-F0-9]{40}' README.md using-the-app.md faq.md
```

Expected: Development Preview is consistently non-production/test-only and no contract address is published.

- [ ] **Step 5: Validate final diff and worktree**

Run:

```bash
git diff --check origin/main..HEAD
git diff --stat origin/main..HEAD
git status --short --branch
```

Expected: only the design, plan, tutorial, FAQ, README, and SUMMARY changes are included; the worktree is clean after commits.
