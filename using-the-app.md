# Using Bivium — a walkthrough

Bivium is **non-recourse, fixed-rate, no-liquidation** lending. You post BTC or ETH as collateral, draw
USDC, and at maturity you either **repay** (and reclaim your collateral) or **walk away** (deliver the
collateral at your chosen floor and keep the USDC). There is **no liquidation engine and no margin calls** —
the contract only ever reads "did you repay, yes or no."

Economically a Bivium loan *is* a put option: the borrower is protected on the downside (long a put), and the
lender earns a premium for standing behind it (short a put). You don't need to think in options to use the app
— but it's why the rates and the "deliver at your floor" mechanic work the way they do.

This guide covers the three tabs — **Borrow**, **Earn**, **Positions** — plus first-time setup. It's a usage
guide; for how the protocol works, see the [protocol overview](protocol-overview.md).

---

## 0. First time — connect and get test tokens

1. **Connect a wallet** (top-right). The app runs on a testnet (Sepolia) — no real funds.
2. **Get gas:** use the external **Sepolia ETH faucet** linked in the faucet bar to fund gas.
3. **Mint demo tokens:** in the **Sepolia faucet** bar, click **Mint USDC** (to lend/repay) and **wrap ETH**
   (to use as collateral). These are mock tokens for the demo only.

If you're on the wrong network, the faucet bar prompts you to switch.

---

## 1. Borrow — get USDC against your BTC/ETH, with no liquidation

The **Borrow** tab lends you USDC against collateral at a **fixed rate**, originated in a single transaction
(you fill the pool's standing bid — there is no router, no gateway, and no capability grant).

1. **Enter how much collateral to lock** (e.g. `1.00` BTC). The big number up top is the current **fixed
   borrow rate**.
2. **Read the QUOTE box:**
   - **You receive (≈)** — the USDC principal paid to you now.
   - **Repay by maturity** — the fixed amount of USDC owed at maturity (the "face").
   - **Pool liquidity** — how much the pool can currently lend (your draw can't exceed it).
3. **Approve** the collateral token to the core (one-time per allowance), then click **Get … now →**.
4. You now hold the USDC. The collateral is escrowed at your **floor** (strike).

**At maturity you choose:**
- **Repay** the face in USDC → reclaim 100% of your collateral; or
- **Walk away** — do nothing, and the collateral is delivered at your floor. You keep the USDC.

No price feed, no liquidation, no margin call can touch the position before maturity. The most you ever owe is
the fixed face; the worst case is that you part with collateral you'd already decided you were willing to sell
at the floor.

---

## 2. Earn — lend USDC and collect the premium

The **Earn** tab is the lender side: deposit USDC into the **curated vault** (a standard ERC-4626 pool) and
earn the curve spread — the **volatility-risk premium** a borrower pays for no-liquidation financing.

1. **Enter an amount** of USDC. The big number is the current **lend rate** (the same curve the Borrow tab
   pays).
2. **Approve** USDC to the vault, then **Deposit** → you receive vault shares.
3. **How your yield accrues:** interest streams into the vault's NAV as locked-profit; if a borrower defaults,
   the position settles **repay-or-deliver** — you receive the collateral (not a fire-sale), and NAV re-marks
   at maturity.
4. **Withdraw** anytime there is idle or repaid liquidity (standard ERC-4626 `redeem`) — the **Withdraw** button
   shows your current redeemable value.

The key idea: you are **short a put**. A high rate means a higher chance you end up holding the collateral — so
only lend against assets (BTC/ETH) you'd be content to own at the floor. (See `SECURITY.md` → adverse selection.)

Below the deposit panel, the **dual-currency / secondary** panels show the same unified order book, where
sophisticated lenders can rest or take individual offers directly.

---

## 3. Positions — manage loans and deposits

The **Positions** tab has two sub-tabs.

**Loans** (you borrowed):
- Each card shows **days to maturity**, the **repay-by** amount, and your locked collateral.
- **Repay** — pay the face in USDC to clear the debt (do this before maturity to reclaim collateral).
- **Withdraw collateral** — once repaid (or the position is otherwise free), pull your BTC/ETH back.

**Deposits** (you lent):
- Each card shows **days to maturity** and your redeemable position.
- **Claim … basket** — at maturity, claim your share of the settlement basket (USDC from repayers +
  collateral from any defaulters, `min(face, collateral)` in kind).
- **Withdraw** — pull idle/repaid liquidity that hasn't been claimed into a position.

---

## Key terms

- **Floor / strike** — the price at which you (borrower) would deliver the collateral, or (lender) would be
  assigned it. Lower floor = safer for the lender, cheaper protection for the borrower.
- **Maturity** — the fixed end date. `repay` and new borrows are only valid *before* it; `claim` only *after*.
- **Rate / APR / tick** — one number, shown as an APR. It's the fixed cost of the loan and the fixed yield to
  the lender; under the hood it's a price on a logistic tick grid with uniform log-odds steps, tuned for
  fine discount/rate resolution near par.
- **Repay-or-deliver** — the only settlement rule: a credit holder receives `min(face, collateral)` in kind. No
  oracle decides this; the borrower's repay/no-repay choice does.
- **No liquidation** — there is no maintenance margin and no liquidator. Nothing happens mid-term regardless of
  price; everything settles once, at maturity.
- **Curated vault** — the passive lender pool (ERC-4626). A curator allocates deposits into vetted markets at
  the curve rate, so depositors don't pick individual markets.

---

## Learn more

- **[Protocol overview](protocol-overview.md)** — the offer model, repay-or-deliver settlement, why
  there's no oracle and no liquidation, and how rates are set.
- **[Security](security.md)** — trust model, the lender-side risk to understand, and token assumptions.

> Testnet, unaudited, demo tokens only. Do not use with real funds.
