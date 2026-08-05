# Bitcoin Credit Markets Article Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Publish an original Bivium research article explaining why Bitcoin collateral needs fixed-term, standardized, market-priced credit claims, and add it to the GitBook navigation.

**Architecture:** Create one self-contained research page that derives Bivium's market structure from its protocol constraints, then connect it to the existing introduction and GitBook summary. Keep product instructions in `using-the-app.md` and risk detail in `security.md`; the new page links to those sources instead of duplicating them.

**Tech Stack:** GitBook, Markdown, Git, ripgrep

---

### Task 1: Draft the Bivium-native research article

**Files:**
- Create: `bitcoin-credit-markets.md`

- [ ] **Step 1: Write the thesis and lending-versus-market opening**

Create the page with the title `Bitcoin Collateral Needs Credit Markets`. Open with the distinction between isolated loan origination and a market in which credit can be priced, transferred, and compared. State that Bitcoin's quality as collateral does not by itself create term liquidity.

- [ ] **Step 2: Derive explicit term structure and standardized DCN**

Explain that maturity establishes a repayment cutoff and comparable term. Define DCN as fungible credit only within one exact eight-field market domain, with no portability across chains, core deployments, tokens, strikes, maturities, repayment policy, or gates.

- [ ] **Step 3: Explain the shared Offer model and price discovery**

Describe Borrow, Lend, Buy, and Sell as movements of the same market claim through ratified offers. Explain that the current Preview uses signed quotes specifically, without implying that every Offer, ratifier, or action is intrinsically signed. Explain that `tick` encodes price, APR is derived for display, and neither the core nor a displayed APR guarantees fair pricing or liquidity.

- [ ] **Step 4: Derive repay-or-deliver settlement**

Explain strict pre-maturity repayment, collateral delivery for unpaid debt, and pro-rata two-token claims from maturity onward. Connect this directly to the absence of price-oracle liquidation and margin calls.

- [ ] **Step 5: State risk and trust boundaries**

Cover physical-delivery exposure, collateral and wrapper quality, liquidity, quote authority, relayer liveness, market-domain validation, token behavior, and smart-contract risk. Link to `security.md` for the complete boundary.

- [ ] **Step 6: Close with the current availability boundary**

Link the Development Preview and state that it is unaudited and non-production. Do not mention or imply Pool, LP, vault, curator, structured-finance, rehypothecation, or future financing functionality.

- [ ] **Step 7: Audit the article before commit**

Run:

```bash
rg -ni '\b(pool|pools|lp|vault|vaults|curator|curators|subscription|deposit|redeem|rehypothecat|securiti[sz])\b' bitcoin-credit-markets.md
rg -n '0x[a-fA-F0-9]{40}' bitcoin-credit-markets.md
git diff --check -- bitcoin-credit-markets.md
```

Expected: no disabled-feature or contract-address matches and no whitespace errors.

- [ ] **Step 8: Commit the article**

```bash
git add bitcoin-credit-markets.md
git commit -m "docs: explain Bivium bitcoin credit markets"
```

### Task 2: Connect the article to GitBook navigation

**Files:**
- Modify: `SUMMARY.md`
- Modify: `README.md`

- [ ] **Step 1: Add the summary entry**

Insert this exact entry immediately after Introduction:

```markdown
* [Bitcoin credit markets](bitcoin-credit-markets.md)
```

- [ ] **Step 2: Add the introduction link**

Under `Where to start`, add a bullet that describes the page as the rationale for fixed terms, standardized DCN, market pricing, and repay-or-deliver settlement.

- [ ] **Step 3: Verify navigation order and commit**

Run:

```bash
sed -n '1,20p' SUMMARY.md
rg -n 'Bitcoin credit markets|bitcoin-credit-markets.md' README.md SUMMARY.md
git diff --check -- README.md SUMMARY.md
```

Expected: the new page is after Introduction and before Using the app, both links point to the same file, and no whitespace errors are reported.

Commit:

```bash
git add README.md SUMMARY.md
git commit -m "docs: add bitcoin credit markets to GitBook"
```

### Task 3: Validate the published documentation set

**Files:**
- Verify: `bitcoin-credit-markets.md`
- Verify: `README.md`
- Verify: `SUMMARY.md`
- Verify: `using-the-app.md`
- Verify: `protocol-overview.md`
- Verify: `security.md`

- [ ] **Step 1: Validate local Markdown links**

Extract every relative `.md` link from the published pages and confirm that its target exists.

Expected: `all local Markdown links exist`.

- [ ] **Step 2: Audit disabled feature claims across published pages**

Run:

```bash
rg -ni '\b(pool|pools|lp|vault|vaults|curator|curators|subscription|deposit|withdraw idle|request redeem|claim redeem|settle pool)\b' README.md bitcoin-credit-markets.md using-the-app.md protocol-overview.md security.md
```

Expected: no user-facing statement presents a disabled feature as available. Core phrases such as `pooled settlement` are acceptable only when they describe accounting rather than a product surface.

- [ ] **Step 3: Audit deployment and economic claims**

Run:

```bash
rg -ni 'guarantee|guaranteed|risk.free|production|dev\.bivium\.pages\.dev|0x[a-fA-F0-9]{40}' bitcoin-credit-markets.md README.md
```

Expected: no guaranteed economic outcome, no production claim, no contract address, and a correct Development Preview link.

- [ ] **Step 4: Validate the article's protocol statements**

Compare its claims against `protocol-overview.md` and confirm it includes fixed term, exact market identity, `tick`, strict pre-maturity repayment, claims from maturity, physical two-token delivery, no oracle, and no liquidation.

- [ ] **Step 5: Review the final diff and working tree**

Run:

```bash
git diff --check origin/main..HEAD
git diff --stat origin/main..HEAD
git status --short --branch
```

Expected: only the design, plan, new article, README, SUMMARY, and intentional `security.md` source-alignment cleanup are present; the worktree is clean after commits.
