# Dev Preview GitBook Alignment Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Update the public Bivium GitBook to describe only functionality enabled in the current dev Preview and remove Pool/LP workflows that are not enabled.

**Architecture:** Treat the deployed `bivium-frontend` `origin/dev` behavior as the product source of truth. Revise the introduction, user journey, protocol context, and security boundaries together so they consistently describe configured markets, Loan/DCN/Order portfolio records, and claim-proceeds previews without implying that Pool functionality is available.

**Tech Stack:** GitBook, Markdown, Git, ripgrep

---

### Task 1: Align the introduction and release status

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Replace the stale release paragraph**

Add the public dev Preview link, identify it as a test environment, state that only visible and enabled workflows are documented, and retain the unaudited/no-real-funds warning. Do not publish contract addresses or claim production status.

- [ ] **Step 2: Check introduction terminology**

Run: `rg -ni '\b(pool|lp|subscription|redeem|settle)\b' README.md`

Expected: no disabled product workflow is presented as available.

- [ ] **Step 3: Commit the introduction update**

```bash
git add README.md
git commit -m "docs: identify the dev preview environment"
```

### Task 2: Rewrite the app walkthrough around enabled workflows

**Files:**
- Modify: `using-the-app.md`

- [ ] **Step 1: Update connection and environment guidance**

Link `https://dev.bivium.pages.dev`, label it Development Preview, explain the Header domain checks, and avoid fresh-release or legacy wording that conflicts with a live test frontend.

- [ ] **Step 2: Limit Markets and Pro to enabled actions**

Document Borrow, Lend now, configured Rate order, and signed order-book behavior. Remove Deposit, Pool, LP, subscription, guarded pool bid, and pool-manager instructions.

- [ ] **Step 3: Limit Portfolio to enabled record types**

Document Loan, DCN, and Order rows only. Explain that configured listed markets bound what the Preview can safely load. State that an unavailable/invalid market domain is unknown state, not an empty portfolio.

- [ ] **Step 4: Document matured DCN claim preview**

Explain that `Estimated proceeds` shows expected loan-token and collateral-token amounts before Claim, refreshes from chain state, and is not a guaranteed execution result.

- [ ] **Step 5: Check disabled terms**

Run: `rg -ni '\b(pool|pools|lp|subscription|redeem|settle|settlement manager)\b' using-the-app.md`

Expected: any remaining `settlement` usage describes core repay-or-deliver settlement only; no disabled Pool/LP workflow remains.

- [ ] **Step 6: Commit the walkthrough update**

```bash
git add using-the-app.md
git commit -m "docs: align the app guide with dev preview"
```

### Task 3: Align protocol and security boundaries

**Files:**
- Modify: `protocol-overview.md`
- Modify: `security.md`

- [ ] **Step 1: Remove optional Pool pricing from the protocol overview**

Keep signed CLOB and configured RFQ/intents as the documented rate sources. Remove Pool/curve availability wording and replace internal “pool” phrasing with market-level settlement wording where it does not change protocol meaning.

- [ ] **Step 2: Clarify dev versus production status**

State that a dev Preview frontend is available for testing but the release is unaudited and not promoted to production. Do not publish addresses.

- [ ] **Step 3: Remove disabled-product references from Security**

Remove optional Pool/vault product-surface descriptions and Preview promotion requirements that enumerate pools. Preserve core accounting, pooled repay-or-deliver settlement, and pro-rata claims when those words describe protocol mechanics rather than a Pool UI feature.

- [ ] **Step 4: Check security and protocol terminology**

Run: `rg -ni '\b(pool|pools|lp|subscription|redeem|pool manager|settlement manager)\b' protocol-overview.md security.md`

Expected: every remaining match is required to explain core pooled settlement or an assurance property, not an enabled Pool/LP product.

- [ ] **Step 5: Commit protocol and security updates**

```bash
git add protocol-overview.md security.md
git commit -m "docs: remove disabled pool product claims"
```

### Task 4: Validate the complete GitBook

**Files:**
- Verify: `README.md`
- Verify: `SUMMARY.md`
- Verify: `using-the-app.md`
- Verify: `protocol-overview.md`
- Verify: `security.md`

- [ ] **Step 1: Validate Markdown formatting**

Run: `git diff --check HEAD~3..HEAD`

Expected: no output and exit status 0.

- [ ] **Step 2: Validate internal Markdown links**

Run a local Markdown-link check that extracts relative `.md` destinations from published pages and confirms every target exists.

Expected: `all local Markdown links exist`.

- [ ] **Step 3: Audit disabled feature terms**

Run: `rg -ni '\b(pool|pools|lp|subscription|deposit|withdraw idle|request redeem|claim redeem|settle pool)\b' README.md using-the-app.md protocol-overview.md security.md`

Expected: no user-facing claim that Pool or LP functionality is enabled; only unavoidable protocol phrases such as pooled settlement remain.

- [ ] **Step 4: Audit deployment claims and addresses**

Run: `rg -ni 'production|preview|dev\.bivium\.pages\.dev|0x[a-fA-F0-9]{40}' README.md using-the-app.md protocol-overview.md security.md`

Expected: Preview is consistently labeled Development/test/non-production and no contract address is published.

- [ ] **Step 5: Review final diff and status**

Run: `git diff HEAD~3 -- README.md using-the-app.md protocol-overview.md security.md && git status --short --branch`

Expected: only intended documentation and planning files differ from `origin/main`; the working tree is clean after commits.
