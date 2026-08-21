# Bitcoin vault markets (vaultBTC and TBVBTC)

Most Bivium markets take a divisible ERC-20 as collateral: you choose an amount, and the face of the
loan follows from it. A **Bitcoin vault market** inverts that. Its collateral is a whole Bitcoin vault
held by an external trustless-BTC-vault (TBV) registry, and a vault is **indivisible** — you pledge all
of it or none of it. The size of the vault therefore *fixes* the face of the loan rather than the other
way round.

That single property explains everything else on this page: why borrowing selects vaults instead of
typing an amount, why the borrow fills exactly one lender quote, and why two different tokens appear.

> Everything here is part of the **[Development Preview](https://dev.bivium.pages.dev)**. In the
> Preview no native bitcoin ever moves: the registry is a stand-in, and the redemption book is settled
> by a mock keeper. See [Preview boundary](#preview-boundary) before drawing conclusions about
> mainnet behaviour.

## The two tokens

A vault market involves two ERC-20s. They are not two flavours of the same thing; they sit on opposite
sides of a decision the vault owner makes.

| | **vaultBTC** | **TBVBTC** |
|---|---|---|
| What it represents | one specific vault you own, wrapped 1:1 in satoshis | a share of the pool of vaults handed to the keeper side |
| Transferable | **No** — bound to the vault's original depositor | **Yes** — an ordinary ERC-20 |
| Backing | the exact vault your lot points at | every vault in the pool, interchangeably |
| Native-BTC exit | `reclaim` returns the vault to the registry for redemption to the depositor's own BTC key | the redemption book: a keeper front-pays BTC and takes the escrowed TBVBTC |
| Where you see it | inside the app's vault lane only | balances, the TBVBTC market, the redemption book |

**vaultBTC is an internal credential, not an asset to hold.** It cannot be sent to another wallet; it
moves only along the lanes the vault lifecycle needs (wrapping, borrowing, the conversion escrow). If
you want something you can trade, sell, or lend against freely, that is TBVBTC — and getting it means
giving up the direct claim on your particular vault.

## The borrower's lifecycle

### Getting a vault

On mainnet a vault appears because it was pegged in on Bitcoin and the registry activated it through
the app. In the Preview, **Get Mock Vault** in the faucet bar stands in for that step: it wraps a
pretend peg-in of the size you enter and opens a lot for you.

The default size is deliberately small. A vault's face is `vault sats × strike ÷ 1e36`, and (see below)
one borrow consumes exactly one lender quote, so an oversized vault simply cannot be borrowed against
on a thin book.

### Borrowing

Borrow in a vault market has no amount field. You **select whole vaults**; the panel derives the face
and shows the single best quote that can cover it.

1. Select one or more idle vaults. Their sats are summed; the face follows from the market's strike.
2. Approve the vaults as collateral, and grant the vault app the fill capability once per account.
3. Submit. The app escrows the whole group and fills **one** lender bid with you as the taker.

Consequences worth internalising:

- **One quote, not a sweep.** An ordinary market walks the book best-first. A vault borrow does not: if
  no single resting bid covers the face, the panel says so instead of partially filling.
- **All-or-nothing groups.** A group is bound to one loan; it is released or delivered together. A live
  group is also capped in size so that its lifecycle transitions always fit in one transaction.
- **Origination lives in Borrow.** The Pro ticket cannot issue new credit in a vault market, because
  new collateral can only enter through the vault app. Pro can still sell credit you already hold.
- **Rate orders use the vault lane.** A resting "borrow at ≤ this rate" request in a vault market is a
  signed intent answered by lender quotes in the ordinary book; it is not the Permit2 settlement route
  used by other markets.

### Repaying and what comes next

Repayment rules are the ordinary ones: strictly before maturity, in full, in the market's loan token.
What differs is the tail.

After repaying, the loan row's **Release vault** performs two steps in order — it withdraws the
collateral out of the core, then clears the group's binding. Once released, the vault is idle again and
you face three roads:

- **Borrow again** — the same vault against a new quote, in this market or a later maturity. This is how
  a vault position rolls without ever leaving the system.
- **Reclaim** — hand the vault back to the registry so it can be redeemed to the depositor's own BTC
  key. This ends the vault's life inside Bivium; the redemption itself happens on the registry side.
- **Convert** — turn it into TBVBTC (next section).

Nothing forces the choice, and an idle vault can sit indefinitely.

### If the loan is not repaid

From maturity onward with debt outstanding, anyone may mark the group **delivered**. The vault then
belongs to the keeper side, and the collateral leg of settlement pays **TBVBTC** to the credit holders
who claim it. The original depositor keeps the loan proceeds and no longer has the direct claim on the
vault — but see `unconvert` below, which remains open until a keeper settles it.

## Convert and unconvert: the door between the two tokens

`convert` takes an **idle** vault of yours, locks its vaultBTC in an escrow, and mints you an equal
amount of TBVBTC. `unconvert` is the reverse: burn an equal amount of TBVBTC and the same vault comes
back to you as an idle lot.

Three things make this more than a wrapper:

1. **It is not a burn.** vaultBTC is locked, not destroyed. The escrow publishes a one-read check that
   the locked amount always equals the TBVBTC supply.
2. **It works on defaulted vaults too.** A vault delivered by default can be bought back by its original
   depositor with an equal amount of TBVBTC — any TBVBTC, because the pool is interchangeable — for as
   long as no keeper has settled it. Credit holders are unaffected: they hold a pool claim, never a
   claim on one particular vault, and the pool's backing ratio does not move (supply and locked
   collateral fall together).
3. **What you give up is specificity.** Converting exchanges "my vault, redeemable to my own BTC key"
   for "a pool share, exitable via a keeper". Coming back is possible while the vault is unconsumed, but
   it is not guaranteed forever.

The practical reason to convert: TBVBTC is an ordinary ERC-20. It borrows like any other collateral in
its own market — any amount, sweeping several quotes, with the standard rate-order lane — and it can be
sold or moved. A vault cannot do any of that.

## The TBVBTC market

`TBVBTC / USDC` is an ordinary market. Nothing about it is special-cased: TBVBTC is a plain ERC-20
collateral, so borrowing against it behaves exactly like borrowing against any other represented BTC
asset in these docs. Everything in [Using the app](using-the-app.md) applies unchanged.

This is the practical answer to the "my vault is too lumpy" problem: convert once, then use ordinary
market mechanics on any size you like.

## Exiting to native bitcoin

Two exits exist, and they trade off differently.

**Reclaim (vault holders).** The vault returns to the registry, which redeems it to the BTC key bound at
peg-in — the depositor's own. No intermediary chooses the destination. The cost is time: it waits for
the registry's own redemption path.

**The redemption book (TBVBTC holders).** Escrow TBVBTC against an ask denominated in native satoshis
that decays from a starting price toward a floor over a deadline you set. A keeper that finds the price
worthwhile pays your bitcoin destination first and then claims the escrow. You are buying speed and
paying for it in the spread.

Two properties of the book are worth stating plainly:

- **It is escrow, never an IOU.** Until a fill lands your TBVBTC is still fully backed and sitting in
  escrow; past the deadline an unfilled order is cancellable by you.
- **Your guarantee is the claim, not the bitcoin.** The protocol can ensure a TBVBTC holder never loses
  the claim — cancellable escrow, tradable token, buy-back by the original depositor. It cannot itself
  ensure a keeper shows up, and the leg that turns TBVBTC into bitcoin depends on the keeper set and on
  payment verification. See [Security](security.md).

## How a vault market differs, in one table

| | Ordinary market | Bitcoin vault market |
|---|---|---|
| Collateral | divisible ERC-20, any amount | whole vaults; face follows vault size |
| Borrow execution | best-first sweep across quotes | exactly one quote per borrow |
| Origination in Pro | available | Borrow panel only |
| Resting rate order | Permit2 settlement lane | signed vault intent answered in the book |
| Collateral after repay | withdraw | withdraw, then release the group binding |
| Default leg pays | the collateral token | **TBVBTC** |
| Native-BTC exit | out of scope for the core | reclaim, or the keeper redemption book |

## Preview boundary

In the current Development Preview:

- **No native bitcoin moves anywhere.** The registry is a stand-in whose activation is open to anyone —
  that is exactly why it can serve as a faucet — and a reclaimed vault's redemption is recorded, not
  paid.
- **The redemption book is filled by a mock keeper** that does not pay bitcoin, and payment verification
  is not enabled, so a fill is asserted rather than proven. On mainnet this is the single most important
  piece to have in place; the app labels the simulated legs.
- **Addresses are not published here** and change between test releases. If your browser tab predates the
  current release the app will tell you and refuse to transact — reload rather than sign.

None of the Preview's convenience should be read as a claim about a production deployment.

## Continue

- **[Using the app](using-the-app.md)** — the click-by-click vault workflow.
- **[Protocol overview](protocol-overview.md)** — market identity, offers, and repay-or-deliver settlement.
- **[Security](security.md)** — the keeper set, verification, and what each token's guarantee covers.
- **[FAQ](faq.md)** — short answers to the questions this page raises.

> Development Preview and unaudited software. Use test assets only.
