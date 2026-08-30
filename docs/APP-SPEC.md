# Shree Laxmi Home — Application Specification

**Version:** 1.3 · drawn from `index.html` at service-worker cache `sip-tracker-v93`
**Purpose:** a complete, self-contained description of the app, written so a mobile application can be built from this document alone.

Every rule below is taken from the running code, with line references into `index.html` so any claim can be checked. Where a code comment disagrees with what the code actually does, this document states the behaviour and flags the discrepancy in [§23](#23-known-issues-to-fix-in-the-rebuild).

---

## Table of contents

**Part I — Product**
1. [What this app is](#1-what-this-app-is)
2. [People and the access model](#2-people-and-the-access-model)
3. [Navigation map](#3-navigation-map)

**Part II — Data**

4. [Entity reference](#4-entity-reference)
5. [Identifiers](#5-identifiers)
6. [Seed data](#6-seed-data)
7. [Deletion semantics](#7-deletion-semantics)

**Part III — Rules**

8. [The accounting model](#8-the-accounting-model)
9. [Investment rules](#9-investment-rules)
10. [Shopping rules](#10-shopping-rules)
11. [Recurring entries](#11-recurring-entries)

**Part IV — Architecture**

12. [Sync engine](#12-sync-engine)
13. [Authentication](#13-authentication)
14. [Access control](#14-access-control)
15. [Security model — honest limits](#15-security-model--honest-limits)
16. [Storage keys](#16-storage-keys)

**Part V — Interface**

17. [Design system](#17-design-system)
18. [Interaction patterns](#18-interaction-patterns)
19. [Copy and locale](#19-copy-and-locale)

**Part VI — Rebuild guidance**

20. [What to keep, what to change](#20-what-to-keep-what-to-change)
21. [Recommended architecture](#21-recommended-architecture)
22. [Behaviour catalogue for tests](#22-behaviour-catalogue-for-tests)
23. [Known issues to fix in the rebuild](#23-known-issues-to-fix-in-the-rebuild)

---

# Part I — Product

## 1. What this app is

A private application for one family. Everybody in the family sees the same data, on their own phone, and edits it from wherever they are. There is no server: the shared state is a single JSON file in one member's Google Drive, and every device reads, merges and writes that file.

It has three modules.

| Module | Answers | Gated |
|---|---|---|
| **Monthly Ledger** | Where does the household's money go, and what is actually left? | Yes |
| **Shopping List** | What do we need to buy, what do we get through, and what does it cost? | No |
| **Investment** | Are the monthly SIPs and deposits actually being paid, and what has accumulated? | Yes |

### What it is not

- **Not a bank integration.** Nothing is imported. Every figure is typed in by a family member.
- **Not multi-tenant.** One family, one shared file. There is no notion of separate households.
- **Not an encrypted vault.** The PIN is a screen lock; see [§15](#15-security-model--honest-limits).
- **Not a public product.** The Google OAuth app is in Testing mode with family members as Test users, and must stay that way — see [§13](#13-authentication).

### Design commitments

These came from repeated, explicit user direction and must survive the rebuild.

1. **English only.** No Hindi anywhere in the interface, including item names, hints and toasts.
2. **Home module tiles show no numbers.** Tiles are doors, not dashboards. Figures live inside the modules.
3. **Every dropdown opens blank**, with nothing preselected, so a value is always a deliberate choice.
4. **The app works offline.** All data is local; sync is a background convenience.
5. **Spending and bank outflow are two different questions** and are never conflated ([§8](#8-the-accounting-model)).

---

## 2. People and the access model

A **person** is a named family member. A person may carry the Google email they sign in with, and may be flagged **admin**.

- A Google login is matched to a person **by email only**, never by display name alone (`myPersonId()`, `index.html:5380`).
- The first person to sign in becomes an admin, so the app is never left without one (`claimIdentity()`, `index.html:5399`).
- **Invariant: the app always behaves as though it has an admin.** If no person carries the admin flag, `isAdmin()` returns `true` for everyone (`index.html:5389`). This is a deliberate bootstrap escape hatch — without it, a fresh or half-synced install would lock every member out of the gated modules permanently.
- Non-admins get into the Ledger and Investment only through a **30-minute grant** issued by an admin ([§14](#14-access-control)). The Shopping List needs no approval.

| Capability | Admin | Member with a live grant | Member without a grant |
|---|---|---|---|
| Shopping List | ✅ | ✅ | ✅ |
| Monthly Ledger | ✅ | ✅ | ❌ — request access |
| Investment | ✅ | ✅ | ❌ — request access |
| Approve / deny / revoke access | ✅ | ❌ | ❌ |
| Grant access without being asked | ✅ | ❌ | ❌ |
| Mark another person as admin | ✅ | ❌ | ❌ |
| Download / upload the Drive backup | ✅ | ❌ | ❌ |
| End their own access early | — | ✅ | — |

Backup download and upload are admin-only for a specific reason: the file is the **whole family's** data, so exporting it exports everyone's.

---

## 3. Navigation map

Two levels. **Home** is the shell; each module owns its own bottom navigation bar, which is rebuilt when the module opens and hidden on Home.

```
Home  (#sec-home)
├── Monthly Ledger  (gated)      bottom nav: Insights · Add · Setup · Log
│   ├── Insights      #sec-lgd
│   ├── Add           #sec-lgadd
│   ├── Setup         #sec-lgheads
│   └── Log           #sec-lglog
├── Shopping List  (open)        bottom nav: List · Items · Consumption · History
│   ├── List          #sec-shlist
│   ├── Items         #sec-shitems
│   ├── Consumption   #sec-shcons
│   └── History       #sec-shhist
└── Investment  (gated)          bottom nav: Dashboard · Summary · Tracker · Transfer · Funds
    ├── Dashboard     #sec-dash
    ├── Summary       #sec-summary
    ├── Tracker       #sec-tracker
    ├── Transfer      #sec-transfer
    └── Funds         #sec-funds

Access gate  #sec-noaccess   — shown instead of a gated module
```

Module configuration lives in one object (`MODULES`, `index.html:1307`), including each module's display name, its `gated` flag, and its tab list with icon and label. A rebuild should keep this table-driven shape.

### Home screen contents, in order

1. App name with the app icon to its right; tapping the icon opens it large in a modal.
2. **Module tiles** — one per module, with a lock glyph (🔒/🔓) on gated ones and "Needs admin approval" when locked. **No figures.**
3. **Pending approval requests** (admins only), each with Approve and Decline.
4. **Last signed in** — every known member with a relative timestamp ("just now", "3 hours ago", "yesterday"), deduplicated by email.
5. Sync status and, for admins, the backup tools.

### Gate rules

- Opening a gated module **first pulls from Drive** if a token and file are available, so an approval granted seconds ago on another device is seen before the member is refused (`showModule()`, `index.html:1331`).
- Tab switching re-checks access, so a stale tab can never be used to slip into a gated module after a grant lapses (`showSec()`, `index.html:1539`).
- A sweep every 20 seconds ejects anyone whose grant has lapsed back to Home with "Your 30 minutes are up" (`index.html:5775`).
- Returning Home triggers an immediate sync.

---

# Part II — Data

## 4. Entity reference

All application state is one object, `ST`, persisted as JSON under `localStorage["sip_state_v2"]` and synced verbatim to Drive. It divides into **keyed maps** (`{id: record}`), **row lists** (arrays), and **sync metadata**.

### 4.1 Row lists

#### `ST.ledger[]` — every income and expense entry

Created at `index.html:3632`.

| Field | Type | Notes |
|---|---|---|
| `id` | number | `hashId(srcKey)` for generated rows, otherwise `Date.now() + random(0..999)` |
| `date` | `"YYYY-MM-DD"` | the date the entry is *filed under*, chosen by the user |
| `type` | `"expense" \| "income"` | |
| `head` | head id | |
| `sub` | subhead id or `""` | optional |
| `card` | boolean | a manual one-off "do not count this as spending" mark |
| `pay` | payment-type id or `""` | where the money came from |
| `amt` | number | always > 0 |
| `note` | string | free text |
| `by` | person name (string) | who the entry is *for* |
| `src` | `"sip" \| "rec" \| null` | which generator created it |
| `srcKey` | string \| null | idempotency key for generated rows |
| `createdBy` | person name | who keyed it in |
| `createdAt` | epoch ms | |
| `editedBy` | person name | present only after an edit |
| `editedAt` | epoch ms | present only after an edit |

Note that `by` and `createdBy` are **names, not ids** — a deliberate simplification that keeps history readable if a person record is later removed. A rebuild may move to ids, but must then keep a display name on the row for orphaned references.

#### `ST.shop[]` — the shopping list and its purchase history

One array serves both: an unbought row is on the list, a bought row is history. Created at `index.html:4796`, completed at `:4913`.

| Field | Type | Notes |
|---|---|---|
| `id` | string | `uid("sh_")` |
| `name` | string | denormalised item name at the time of adding |
| `item` | item id | resolves through aliases |
| `qty` | number | `0` until bought |
| `unit` | unit id | `kg`, `g`, `l`, `ml`, `pcs` |
| `done` | boolean | |
| `note` | string | |
| `createdBy` | person name | |
| `createdAt` | epoch ms | |
| `doneAt` | epoch ms | set to **noon** of the chosen date, avoiding timezone drift |
| `doneBy` | person name | |

#### `ST.transfers[]` — money moved between two people

Created at `index.html:1806`.

| Field | Type | Notes |
|---|---|---|
| `id` | number | `Date.now()` |
| `date` | `"YYYY-MM-DD"` | |
| `amt` | number | |
| `mode` | string | how it was sent |
| `from` | person name | |
| `to` | person name | |
| `lgId` | ledger row id | the mirrored income entry |
| `note` | string | |
| `dir` | `"wife" \| "husband"` | **legacy only** — older rows encoded a fixed Husband↔Wife direction |

Every transfer writes a matching **income** row into the ledger under the `i_transfer` head, credited to the recipient, and remembers that row's id so deleting the transfer can remove it too.

Legacy rows are read through `tFrom(t)` / `tTo(t)`, which fall back to `dir` when `from`/`to` are absent. A rebuild should migrate these once at import and drop `dir`.

### 4.2 Keyed maps

| Map | Key | Record | Purpose |
|---|---|---|---|
| `ST.heads` | head id | `{name, type:"expense"\|"income", excl?:bool, bill?:bool}` | expense/income categories |
| `ST.subheads` | subhead id | `{name, head}` | second-level category, scoped to one head |
| `ST.people` | person id | `{name, email?, admin?:bool}` | family members |
| `ST.pays` | payment-type id | `{name}` | UPI/Bank, Cash, Credit Card, and any added |
| `ST.exclPay` | payment-type id | boolean | `true` = does **not** reduce net income |
| `ST.items` | item id | `{name, unit, alias?}` | shopping catalogue; `alias` points at a merge target |
| `ST.recurring` | rule id | `{head, sub, amt, by, day, from, to, note, active}` | monthly auto-posting rules |
| `ST.requests` | request id | `{email, name, at, status:"open"\|"granted"\|"denied"}` | access requests |
| `ST.grants` | lower-cased email | `{until, by, at}` | access windows; `until:0` means revoked |
| `ST.seen` | lower-cased email | epoch ms | last sign-in, for the Home list |
| `ST.shares` | email | `true` | who the Drive file has been shared with |
| `ST.funds` | plan id | string | overrides a plan's fund name |
| `ST.amts` | plan id | number | overrides a plan's monthly amount |
| `ST.cats` | plan id | string | overrides a plan's category |
| `ST.checks` | `"<monthIndex>_<planId>"` | truthy | this plan was paid in this month |
| `ST.locked` | month index | `{at, amts:{planId:number}}` | a finalised month and its frozen amounts |

Two flags on a head are load-bearing:

- **`bill: true`** — this head *settles* a debt rather than creating new spending. The seeded `e_cardbill` head carries it.
- **`excl: true`** — user-set on any head, with the same effect: never counted as spending.

`ST.exclPay.__v2` is a migration marker, not a payment type; it is filtered out of every render because `payModes()` only returns entries with a `name`.

### 4.3 Sync metadata

| Field | Shape | Purpose |
|---|---|---|
| `ST.meta[kind][key]` | epoch ms | when that single field or row last changed, on any device |
| `ST.tomb[kind][id]` | epoch ms | when that row was deleted (`ledger`, `shop`, `transfers` only) |
| `ST.updatedAt` | epoch ms | stamped by every `save()` |

`ST.meta` covers: `checks`, `funds`, `amts`, `cats`, `locked`, `heads`, `subheads`, `people`, `recurring`, `exclPay`, `pays`, `items`, `requests`, `grants`, `seen`, `transfers`, `ledger`, `shop`, `shares`.

Any write must call `touch(kind, key)` (`index.html:1241`) before `save()`, or the change will lose the next merge. **This is the single easiest thing to get wrong in a rebuild** — make it structurally impossible by routing every write through one repository method that stamps automatically.

---

## 5. Identifiers

Two schemes, chosen per record type for a reason.

**Random, for anything a person creates** — `uid(prefix)` (`index.html:2887`) returns `prefix + Date.now().toString(36) + 6 random base-36 chars`. Prefixes in use: `p_` people, `e_`/`i_` heads (by kind), `s_` subheads, `pay_` payment types, `it_` items, `sh_` shop rows, `r_` recurring rules, `rq_` requests.

**Deterministic, for anything generated** — `hashId(srcKey)` (`index.html:3619`) is cyrb53, a stable 53-bit hash. Generated ledger rows derive their id from a `srcKey`, so two devices that generate the same row independently produce the *same id* and the merge collapses them into one instead of duplicating.

`srcKey` formats:

- Recurring: `rec_<ruleId>_<YYYY-MM>`
- SIP tracker tick: derived the same way from the plan and month

> **This was learned the hard way.** Random ids for generated rows produced duplicate SIP and recurring entries on every device. Random ids for access requests (`"rq_" + Date.now()`) silently overwrote one another when two people asked within the same millisecond.

### Ids that must never change

The seeded ids are referenced by existing data and by code:

- Payment types: `bank`, `cash`, `card`
- Heads: `e_groceries`, `e_rent`, `e_utilities`, `e_transport`, `e_education`, `e_medical`, `e_other`, `e_cardbill`, `e_investment`, `i_salary`, `i_business`, `i_rent`, `i_interest`, `i_dividend`, `i_transfer`, `i_other`
- People: `p_husband`, `p_wife`, `p_daughter`
- Plans: `H1`, `H2`, `H3`, `W1`, `W2`, `W3`, `W4`

Three heads are created on demand if a user's data predates them: `ensureTransferHead()` (`i_transfer`), `ensureCardBillHead()` (`e_cardbill`), `ensureInvestHead()` (`e_investment`).

---

## 6. Seed data

A fresh install must ship exactly this, or existing family data will not line up.

### 6.1 Payment types (`DEFAULT_PAYS`, `index.html:2895`)

| id | name | Counts against net income by default |
|---|---|---|
| `bank` | UPI / Bank | ✅ yes |
| `cash` | Cash | ❌ no |
| `card` | Credit Card | ❌ no |

**Any newly added payment type defaults to *not* counting against net income** (`index.html:2934`), because a wallet or meal card is money already set aside. It can be ticked on in Setup.

### 6.2 Heads (`DEFAULT_HEADS`, `index.html:3033`)

Expense: Groceries, Rent / EMI, Utilities, Transport, Education, Medical, Other, **Card Bill Payment** (`bill: true`).
Income: Salary, Business, Rent Received, Interest, Dividend, Transfer, Other Income.

### 6.3 People (`DEFAULT_PEOPLE`, `index.html:3325`)

Husband, Wife, Daughter.

### 6.4 Investment plans (`DEFAULT_PLANS`, `index.html:1170`)

| id | Owner | Label | Fund | Monthly | Category |
|---|---|---|---|---|---|
| H1 | H | Plan H-1 | ICICI Pru Nifty 50 Index Fund (Direct-G) | ₹20,000 | Equity |
| H2 | H | Plan H-2 | Parag Parikh Flexi Cap Fund (Direct-G) | ₹20,000 | Equity |
| H3 | H | Plan H-3 | HDFC Balanced Advantage Fund (Direct-G) | ₹20,000 | Hybrid |
| W1 | W | Plan W-1 | Nippon India Arbitrage Fund (Direct-G) | ₹10,000 | Hybrid |
| W2 | W | Plan W-2 | ICICI Pru Short Duration Fund (Direct-G) | ₹10,000 | Debt |
| W3 | W | Plan W-3 | Bank FD 5-Year (in Wife's name) | ₹7,500 | Fixed Deposit |
| W4 | D | Plan W-4 SSY | Sukanya Samriddhi Yojana (Daughter) | ₹12,500 | Government |

Owner codes: `H` Husband, `W` Wife, `D` Daughter.

Categories and their projection assumptions (used **only** for the projection card, never for actuals):
Equity 12%, Hybrid 9%, Debt 7%, Fixed Deposit 7%, Government 8.2%.

The tracker covers a fixed 60-month window, **Sep 2026 → Aug 2031** (`MONTHS`, `index.html:1168`). A rebuild should make this a computed range rather than a literal array.

### 6.5 Units (`UNITS`, `index.html:4751`)

| id | Display | Base | Factor |
|---|---|---|---|
| `kg` | kg | kg | 1 |
| `g` | gram | kg | 0.001 |
| `l` | litre | l | 1 |
| `ml` | ml | l | 0.001 |
| `pcs` | count | pcs | 1 |

### 6.6 Shopping catalogue (`DEFAULT_ITEMS`, `index.html:4541`)

50 items in seven groups. Ids are fixed so two devices seeding independently produce the same catalogue and the merge collapses it rather than doubling it.

- **Staples** — Atta (kg), Rice (kg), Sugar (kg), Salt (kg), Toor Dal (kg), Moong Dal (kg), Chana Dal (kg), Rajma (kg), Poha (kg), Suji (kg), Besan (kg), Maida (kg)
- **Oils and dairy** — Cooking Oil (l), Ghee (kg), Milk (l), Curd (kg), Paneer (kg), Butter (kg), Eggs (pcs)
- **Vegetables** — Onion (kg), Potato (kg), Tomato (kg), Garlic (kg), Ginger (kg), Green Chilli (g), Coriander Leaves (g), Lemon (pcs), Cauliflower (pcs), Brinjal (kg), Lady Finger (kg), Green Peas (kg)
- **Fruit** — Banana (pcs), Apple (kg), Orange (kg)
- **Spices** — Turmeric Powder (g), Red Chilli Powder (g), Coriander Powder (g), Garam Masala (g), Cumin Seeds (g), Mustard Seeds (g)
- **Beverages and snacks** — Tea (g), Coffee (g), Bread (pcs), Biscuits (pcs), Namkeen (g), Maggi (pcs)
- **Household** — Bath Soap (pcs), Shampoo (pcs), Toothpaste (pcs), Detergent (kg), Dishwash (pcs), Toilet Cleaner (pcs), Floor Cleaner (pcs), Garbage Bags (pcs), Tissue Roll (pcs), Mosquito Repellent (pcs), Match Box (pcs), Gas Cylinder (pcs)

---

## 7. Deletion semantics

Deleting is not removing, because a plain removal cannot be distinguished from "this device has not seen it yet", and the record would come straight back on the next merge.

**Keyed maps — set the value to `null`.** The key survives with a fresh `meta` stamp, so the deletion propagates and wins over an older edit.

```js
ST.heads[id] = null;      // NOT delete ST.heads[id]
touch("heads", id);
```

Applies to heads, subheads, people, payment types, items, and recurring rules. Every reader must therefore filter out null values — that is why `payModes()`, `itemList()`, `headList()` and friends all test for a truthy record with a name.

**Row lists — write a tombstone.**

```js
ST.ledger = ST.ledger.filter(e => e.id !== id);
ST.tomb.ledger[id] = Date.now();
```

The merge drops any row whose tombstone stamp is greater than or equal to the row's own stamp ([§12](#12-sync-engine)), so a delete beats a concurrent edit made earlier, and loses to one made later.

Tombstones exist for `ledger`, `shop` and `transfers`. They are never garbage-collected; at family scale that is fine, but a rebuild may prune tombstones older than, say, a year.

---

# Part III — Rules

## 8. The accounting model

This is the most important chapter. It went through many rounds of correction, and the distinctions here are the whole point of the Ledger.

### 8.1 Two questions, two answers

> **"What did we spend?"** and **"what actually left the bank?"** are different questions, so they get different rules.

**Spent** — everything the household bought in the period, **by any payment method**. Cash, card, wallet: all of it is spending on the day it happens.

**Net** — income minus only the outflow from payment types that are marked as counting. Cash was withdrawn from the bank earlier, and a credit-card bill is settled later; counting either again at the moment of purchase would count the same rupee twice.

### 8.2 `countsAsExpense(e)` — does this land in Spent?

`index.html:3078`

```
if (e.type === "income")            → false
if (e.card)                          → false   // manual one-off exclusion
if (head.excl || head.bill)          → false   // settlement, not new spending
otherwise                            → true
```

A **settlement** is a row that pays down a debt rather than buying something: a credit-card bill payment, or any head the user has flagged `excl`. The purchases the bill pays for were already counted as spending on the days they happened.

### 8.3 `reducesNet(e)` — does this come off Net?

`index.html:3092`

```
if (e.type === "income")            → false
if (e.card)                          → false
otherwise                            → !isPayExcluded(e.pay)
```

`isPayExcluded(pay)` reads `ST.exclPay[pay]`. Ticking a payment type in **Setup › Payment Types** means "not counted against net income".

Note the asymmetry, and that it is intentional: **a card bill paid from the bank does reduce Net even though it is not spending.** That is the moment the money genuinely leaves.

### 8.4 Per-payment-type pots

`sumUp()`, `index.html:3987`. Each payment type keeps its own balance for the period:

- **In** — income received *in* that type.
- **Out** — spending *from* that type, including settlements (a card bill paid from the bank draws down the bank pot).
- **Balance** — In − Out.

So **cash income is drawn down by cash spending**, and never touches the bank. Every payment type behaves the same way; there is nothing special about cash beyond its default `exclPay` setting.

### 8.5 Card outstanding

`cardOutstanding()`, `index.html:3022`. Cumulative across all time, because that is what "outstanding" means:

```
spent = Σ amt where type != income and pay == "card"
paid  = Σ amt where head.bill
due   = spent − paid
```

### 8.6 Worked example

One month. Payment types: **UPI / Bank** counts against Net; **Cash** and **Credit Card** do not.

| # | Date | Head | Amount | Paid by | Notes |
|---|---|---|---|---|---|
| 1 | 02 | Salary (income) | ₹100,000 | UPI / Bank | |
| 2 | 05 | Groceries | ₹4,000 | Cash | |
| 3 | 09 | Transport | ₹2,000 | Credit Card | |
| 4 | 20 | Card Bill Payment | ₹2,000 | UPI / Bank | head has `bill: true` |
| 5 | 25 | Utilities | ₹3,000 | UPI / Bank | |

**Spent** = 4,000 + 2,000 + 3,000 = **₹9,000**
Row 4 is excluded: it settles row 3, which was already counted on the 9th.

**Net** = 100,000 − (2,000 + 3,000) = **₹95,000**
Only the two UPI / Bank expense rows reduce Net. The cash purchase does not (that money left the bank when it was withdrawn). The card purchase does not (it leaves when the bill is paid) — and the bill payment on the 20th *does*.

**Pots**

| Type | In | Out | Balance |
|---|---|---|---|
| UPI / Bank | 100,000 | 5,000 | +95,000 |
| Cash | 0 | 4,000 | −4,000 |
| Credit Card | 0 | 2,000 | −2,000 |

Bank Out is 5,000: the ₹3,000 utilities plus the ₹2,000 card bill — a settlement is still money leaving that pot.

**Card outstanding** = 2,000 spent − 2,000 paid = **₹0**.

The negative cash balance is expected and correct: it means ₹4,000 of previously-withdrawn cash was used, and no cash income was logged this month to offset it.

### 8.7 Dashboard figures

The Ledger Insights screen shows, for the selected month or whole year:

- **Income**, **Spent**, **Net**, and entry count, each with a percentage delta against the previous comparable period (`previousPeriod()`, `index.html:4027`).
- A note naming exactly which payment types are behind Net, so the number can be reconciled against the pot table without guesswork.
- **Three charts** at the foot of the screen, after the breakdowns they summarise: a trend line, then a donut for expenses and one for income.
- Breakdowns by **head** (with subheads nested), by **person**, and by **payment type**.
- The pot table (`renderPotBlock()`, `index.html:4069`).
- Settlement rows kept visible but separated, so a card bill is auditable without polluting Spent.

**Expenses by payment type** (`renderPayBlock()`, `index.html:4096`) is a two-level view:

- **Top level** — one card or list row per payment type actually used in the period, with its total, entry count and share. Every card is tappable.
- **Drill-down** — a header line stating *"N entries · ₹total"*, a head summary (shown only when more than one head is involved), then **one line per entry**: date, bar, amount, and a tag carrying head · subhead · note. Newest first, with a back link.

### 8.8 The two rings

Expenses and income each get a ring, built by the same `donutCard()` (`index.html:3070`) that the Investment dashboard's *Allocation by Category* uses, so the two dashboards read as one app. They sit at the **foot** of the dashboard: a ring answers "roughly how is this split?", which is a closing summary of the itemised blocks above it, not a substitute for them.

`headSlices(heads, kind)` (`index.html:3061`) turns a period's head totals into slices, largest first. `donutCard(title, centre, slices)` then renders an SVG ring plus a legend of amount and share.

| Chart | Slices | Centre |
|---|---|---|
| Where the money went | heads with expense > 0 | total spent |
| Where the money came from | heads with income > 0 | total received |

Three rules keep it honest:

- **Colour is assigned by rank**, not by head, from a fixed seven-colour list (`SLICE_COLORS`). Heads are user-defined and carry no colour of their own, so the biggest head gets the first colour on every device and in every period. A head's colour therefore means "this is the largest", not "this is Groceries".
- **Seven arcs maximum.** Beyond that the ring stops being readable, so the top six are kept and the rest collapse into a single *Everything else* slice carrying their sum.
- **Each ring can be split by head or by subhead**, chosen with a Heads / Subheads switch under the title (`setSliceBy(kind, mode)`). In subhead mode a slice is labelled *Head › Subhead*; entries filed straight on a head keep the plain head name, so the slices still total exactly what the head view totalled. The switch is **hidden when nothing in that kind carries a subhead** — a control that visibly does nothing is worse than no control. Expenses and income remember the choice separately, since income rarely has subheads worth splitting.
- **An empty chart renders nothing at all** — no zero ring, no empty card. A period with no income simply has no income chart.

The centre figure uses `shortMoney()` (`index.html:4787`): ₹1.00 L, ₹12K, ₹2.50 Cr.

> **Invariant:** the entry count printed on a card must equal the number of rows in its drill-down. Grouping the drill-down by head broke this — two entries under one head showed as a single line while the card said "2 entries". The same class of bug appeared in the Grocery breakdown. Any aggregate count must be derived from exactly the rows the detail view will list.

---

### 8.9 The trend line

`trendChart()` (`index.html`, `trendChart`) plots **income against spending** over the selected window, as two SVG polylines. No charting library: two paths, a floor, dots, and three labels are all it needs.

`trendSeries()` builds the points and decides the grain from the period selector, exactly as every other view does:

| Selection | One point per | Labels |
|---|---|---|
| A single month | day of that month | day number |
| A whole year | month of that year | Jan, Feb, … |

Four rules:

- **Empty periods at the end are dropped**, so the line stops where the data does instead of trailing along the floor. Gaps *inside* the range stay as genuine zeros — a month with no spending is information.
- **Spending obeys `countsAsExpense()`**, so settlements never appear as a spike. Income is any `type === "income"` row.
- **Only three points are labelled** — first, last, and the peak. Labelling all 31 days would be unreadable at phone width.
- **Fewer than two points draws nothing.** One point is not a line.

Both series share one y-scale so the two are directly comparable, and the caption names the peak period and its value.

## 9. Investment rules

### 9.1 Plans

`plans()` (`index.html:1252`) overlays user edits on the seed:

```
fund = ST.funds[id] ?? DEFAULT_PLANS[id].fund
cat  = ST.cats[id]  ?? DEFAULT_PLANS[id].cat
amt  = ST.amts[id]  ?? DEFAULT_PLANS[id].amt
```

Fund name, monthly amount and category are all editable from the Funds tab. The plan set itself is fixed.

### 9.2 The tracker

A grid of 60 months × 7 plans. Ticking a cell records `ST.checks["<monthIndex>_<planId>"] = true` and posts an expense into the ledger under the `e_investment` head, dated the **1st** of that month (`sipDate()`, `index.html:3115`), so investment money shows up in the household's spending.

### 9.3 Finalising a month

`toggleFinalise(mi)` (`index.html:1696`) writes:

```js
ST.locked[mi] = { at: Date.now(), amts: { <planId>: <amount>, … } };
```

Finalising freezes that month's ticks **and** snapshots every plan's amount. `amountFor(mi, p)` (`index.html:1263`) then prefers the snapshot over the live value.

The point: **editing a plan's amount re-prices future months only, never recorded history.** Unlocking a month discards the snapshot and returns it to following current values, with a confirmation.

### 9.4 Totals

`investedByPlan()` (`index.html:1269`) walks **every plan across every month** and adds `amountFor(mi, p)` for each ticked cell.

> This must be per-plan-per-month. An earlier version computed `complete_months × monthly_total`, which ignored partly-paid months and under-reported by ₹87,500. A month where five of seven plans were paid still counts for those five.

`byCategory()` (`index.html:1283`) rolls the same figures up by category, giving monthly commitment, invested to date, and the plans in each.

---

## 10. Shopping rules

### 10.1 Adding and buying

- **List** shows what is still to buy, plus the 50-most-consumed items for one-tap adding. The unit auto-fills from the last purchase of that item.
- Tapping an item opens the buy sheet: quantity, unit, date, note. Saving sets `done`, `qty`, `unit`, `doneAt` (noon of the chosen date), `doneBy`.
- **Quick-buy** records a purchase for an item that was never on the list, creating the row already marked done.
- Typing a name that is not in the catalogue **adds it to the catalogue and to the list** in one action, so the list learns.
- **History** rows are editable — the same sheet reopens and updates in place.

### 10.2 Units and consumption

`consumption(year, month)` (`index.html:5124`) groups purchases by `name|base`, converting through the unit table so grams roll into kilograms and millilitres into litres. A month mixing "500 g" and "1 kg" of Atta totals 1.5 kg.

Consumption is available as **cards** or as a **list**, sorted **most-used first**, with the running month preselected.

### 10.3 Editing an item

Names are typed by hand, so they arrive misspelt and inconsistent — "Aalo", "Aata", "Narial pani". `openItemEdit(id)` / `saveItemEdit()` (`index.html:3626`) open a sheet for the item's **name** and **usual unit**, reachable from two places: the ✎ button on each **Buy Again** row, and the ✎ on each chip in the **Items** tab.

**A rename must travel to the purchase history.** `ST.shop` rows carry a denormalised `name`, and `consumption()` groups on `name|base` — so renaming the catalogue entry alone would fork one item's history into two lines. The save therefore re-points every `ST.shop` row whose `item` matches, stamping each one, exactly as `mergeItems()` does. The toast says how many rows moved.

Two refusals, both deliberate:

- **An empty name** is rejected inline.
- **A name another item already owns** (case-insensitive) is rejected with a pointer to Merge items — joining two items is a different, deliberate action with different consequences, and silently absorbing one into the other would lose the distinction.

Editing an alias edits the item it resolves to, so a merged-away name can never be renamed out from under its target.

### 10.4 Item merging

Several catalogue entries can be mapped to one tracked item — "Pyaaz" and "Onions" both resolving to "Onion" so a single name carries the whole history.

`mergeItems(targetId, sourceIds, newName)` (`index.html:4602`):

1. Optionally renames the target.
2. Sets `alias: targetId` on each source; sources stay in the map (so the merge propagates) but are filtered out of every picker.
3. **Re-points existing `ST.shop` rows** at the target and updates their denormalised `name`, so history follows the merged name.

`resolveItemId()` (`index.html:4588`) follows alias chains with a 5-hop guard against cycles.

### 10.5 Grocery spend from the Ledger

The Shopping dashboard surfaces money as well as quantities, by reading the Ledger's `e_groceries` head — quantities live in Shopping, the money for the same shopping lives in the Ledger.

- Two cards: the selected **month** and the whole **year**, each with total, entry count, and a "settled" note when rows were excluded.
- Both cards are tappable. **Month → one line per entry** (date, bar, amount, and a tag of payment type · subhead · note). **Year → one line per month.**
- Only rows that pass `countsAsExpense()` are included, in both the totals **and** the counts, so the card and the list can never disagree.

---

## 11. Recurring entries

A rule posts the same entry every month without anyone remembering.

**Record** (`index.html:4455`): `{head, sub, amt, by, day, from, to, note, active}` — `day` is 1–31, `from`/`to` are `"YYYY-MM"` (`to` empty means open-ended), `active: false` pauses it.

**Generation** — `runRecurring()` (`index.html:4403`) walks from `from` to the current month, and for each month:

1. Builds `srcKey = "rec_" + ruleId + "_" + ym`.
2. Skips if any ledger row already carries that `srcKey`.
3. Otherwise posts a row dated `ymDay(ym, day)`.

Because the row id is `hashId(srcKey)`, two devices generating the same month independently produce identical ids and the merge collapses them.

**`ymDay(ym, day)`** (`index.html:4388`) clamps into the month: day 31 in February lands on the 28th or 29th rather than rolling into March.

A 240-iteration guard bounds the loop. Skipping a rule whose head has been deleted is deliberate and silent. Pausing a rule stops future months; entries already posted stay.

---

# Part IV — Architecture

## 12. Sync engine

### 12.1 The shared file

One file, `shreelaxmi-data.json` (`SYNC_FILE`, `index.html:2398`), in one family member's Drive, shared with the rest. Every device reads, merges, and writes the whole document.

**Resolution order** (`resolveFile()`, `index.html:2518`):

1. Already-known id in memory.
2. Cached id in `localStorage["sip_file_<googleSub>"]`, verified as still reachable.
3. Otherwise search Drive by name, **oldest first** — if someone accidentally created a second copy, the original is the one everybody should converge on.

`checkDupes()` (`index.html:2547`) counts matching files and warns when more than one exists.

### 12.2 The cycle

`syncNow()` (`index.html:2804`) is strictly **read → merge → write**, never a blind overwrite:

```
download remote  →  applyMerge(remote)  →  upload merged  →  saveLocalCopy()
```

Triggers:

| Trigger | Timing |
|---|---|
| Any `save()` | debounced 1.5 s |
| Background poll | every 60 s |
| App returns to foreground | immediately |
| App goes to background / page unload | `flushSync()` |
| Opening a gated module | before deciding access |
| Returning Home | immediately |

Re-entrancy is guarded: a sync requested while one is running sets a pending flag and re-runs afterwards.

### 12.3 The merge

**Keyed maps — `mergeField()` (`index.html:2720`).** For each key present on either side, the side with the newer `meta` stamp wins; the merged stamp is `max(local, remote)`.

```
takeRemote = (key in remote) && (remoteStamp > localStamp || key not in local)
```

Comparison for change detection is by value, using `JSON.stringify` for objects.

> Using `!==` on objects here reported a change on every single sync, because two structurally identical objects are never `===`. That produced a "Family data updated" toast every 60 seconds.

**Row lists — `mergeRows()` (`index.html:2745`).** Union both sides by row id; for each id, the copy with the newer stamp wins. Then merge both tombstone maps (`max` per id) and drop any row whose tombstone stamp is `>=` the row's stamp. The result is sorted by id descending.

**Result.** `applyMerge()` (`index.html:2775`) writes a pre-merge snapshot to `localStorage["sip_backup"]` first, folds every collection, takes `max` of the two `updatedAt` values, and returns whether anything visibly changed — which is what decides between a silent sync and a "Family data updated" notice.

### 12.4 Convergence walkthrough

Two phones edit while offline.

| | Phone A | Phone B |
|---|---|---|
| 10:00 | Edits the Groceries head's name. `meta.heads.e_groceries = 10:00` | |
| 10:05 | | Adds a ledger row `#123`. `meta.ledger.123 = 10:05` |
| 10:07 | | Deletes ledger row `#99`. `tomb.ledger.99 = 10:07` |
| 10:10 | Syncs: uploads its state | |
| 10:12 | | Syncs: downloads A's state, merges, uploads |

After B's sync the file holds: the renamed head (only A touched it), row `#123` (only B has it), and no row `#99` (B's tombstone at 10:07 beats the row's older stamp). A picks all of this up on its next poll. Neither device lost an edit, and neither needed to know what the other was doing.

**The rule that makes it work:** merge per *field*, not per *document*. A device that has been offline for a week only overwrites the specific fields it actually edited.

### 12.5 Local safety net

- `saveLocalCopy()` (`index.html:2600`) writes `{at, data}` to `localStorage["sip_localcopy"]` after every successful sync — a known-good copy in case the Drive file is lost.
- If the file disappears from Drive, the app offers to restore from that copy.
- **Download backup** exports the JSON to a file. **Upload backup** is offered *only* when no Drive file exists, so it can never clobber a live one. Both are admin-only.
- `applyMerge()` additionally keeps the immediately-prior state in `localStorage["sip_backup"]`.

---

## 13. Authentication

### 13.1 What is in place

Google OAuth **implicit flow** (`response_type=token`), opened in a popup, with the token handed back to the opener via `postMessage` (origin-checked).

```
CLIENT_ID = 260664555067-cv9pos03d69rgn6ckn0th2inb2ngdqqi.apps.googleusercontent.com
SCOPES    = https://www.googleapis.com/auth/drive profile email
```

Only the **client id** is needed and it is public. There is no client secret in this app, and none should ever be added — the implicit flow does not use one, and committing one would leak it to everybody who can read the deployed page.

The session (token, expiry, and the profile: name, photo, Google `sub`, email) is persisted to `localStorage["sip_auth"]` so a refresh or a PWA relaunch does not look like a logout.

`fetchProfile()` reads `oauth2/v3/userinfo`, then seeds people, claims the identity, and starts the first sync.

### 13.2 The `drive` scope constraint

The app deliberately uses the **full `drive` scope**, not `drive.file`.

Under `drive.file` an app can only see files *it created for that user*. A family member the file was shared with still could not read it, because **Drive permission is not app authorisation**. The full scope is the only way a shared family file works.

`drive` is a **restricted** scope. The consequence is a hard operating constraint:

> The Google Cloud project must stay in **Testing** mode, with every family member added under **Audience › Test users**. Publishing it would require Google's verification review.

Test-user tokens expire faster than published-app tokens. This is a limit of the current design, not a bug in the code.

### 13.3 Known defects — do not reproduce

1. **Silent renewal never lands.** `silentRefresh()` (`index.html:2124`) injects a hidden iframe with `prompt=none`; the redirect handler posts the new token to `window.parent`. But the only `message` listener in the file is registered *inside* `handleLogin()` and removed on first success — and on a restored session it is never registered at all. The refreshed token is posted into a window with nothing listening. **Every session therefore dies at ~59 minutes regardless of the refresh machinery.**
2. **`silentRefresh()` gives up when the email is unknown.** It returns immediately if `gUserEmail` is unset, and a failed profile fetch leaves it unset forever.
3. **A failed refresh is never retried.** The timer is not re-armed, so one missed refresh (asleep, offline) ends the session permanently.
4. **`sessionExpired()` evicts the user.** On any 401 it nulls the token *and* the entire profile, clears storage, and — because `updateAuthGate()` tests only `!gToken` — blurs the whole app behind the sign-in screen. Since all data is local and the app works offline, that is a far harsher response than the situation warrants.
5. **403 is never handled**, and `checkFile()` returning `false` on an auth error causes the cached Drive file id to be deleted.
6. **Listener leak** — cancelling the sign-in popup leaves a `message` handler attached, one per attempt.

**In the rebuild, none of this should be carried over.** A native app uses the authorization-code flow with PKCE, which yields a **refresh token** and removes the hourly expiry entirely. See [§20](#20-what-to-keep-what-to-change).

---

## 14. Access control

`GRANT_MS = 30 × 60 × 1000` (`index.html:5368`).

### 14.1 Lifecycle

```
Member opens a gated module
   → app syncs first, then re-checks
   → no access: gate screen with "Request access"
   → requests[uid] = {email, name, at, status:"open"}   → flushSync()

Admin sees the request under the Home tiles
   → Approve: grants[email] = {until: now + 30 min, by, at}; request status "granted"
   → Decline: request status "denied"

Member is in. A sweep every 20 s re-checks:
   → grant lapsed → toast "Your 30 minutes are up" → back to Home
```

`hasAccess()` = `isAdmin() || grant.until > Date.now()`. Expiry is evaluated lazily on read, so it works offline and is unrelated to the Google token.

### 14.2 Other transitions

- **Revoke** (admin) and **End Now** (self) both write `{until: 0}` rather than deleting the grant, so the revocation merges to other devices instead of being resurrected. **End Now takes effect immediately, without a confirmation** — an explicit product decision.
- **Grant now** lets an admin hand out 30 minutes without waiting to be asked. It refuses if the person has no sign-in email mapped.
- Approve, revoke, end and request all call `flushSync()` so the decision reaches other devices at once.
- A duplicate open request from the same email is rejected with a message rather than stacking.

### 14.3 Identity claiming

`claimIdentity()` (`index.html:5399`), run after the profile loads:

1. Find the person whose email matches. If none, adopt an existing person who has **no email** and whose display name matches the Google name; otherwise create one.
2. Write the email onto that person.
3. **Retire duplicates** — any other person row claiming the same email is set to `null`, carrying its admin flag across first, so a duplicate can never strand the admin flag on the discarded row.
4. If nobody is an admin, make this person one.
5. Stamp `ST.seen[email] = now`.
6. Re-render Home.

> Step 6 matters. `fetchProfile()` is async and Home draws before it resolves, so without an explicit re-render every member showed as "never signed in". Step 3 fixes the related symptom of one person appearing twice in the Last-signed-in list because two devices each created a row.

`ST.seen` is kept in its own map rather than on the person record on purpose: a sign-in must not bump the row that also carries `admin` and `email`, or a concurrent edit to those could lose to a mere login.

---

## 15. Security model — honest limits

### What the PIN does

`PIN_KEY = "sip_pin"`, PBKDF2-SHA256, **150,000 iterations**, 256-bit key, random salt (`index.html:6031`). The PIN itself is never stored, only the hash. Five wrong attempts trigger a 30-second cooldown.

The lock screen appears on app launch and when the app has been in the background for more than **2 minutes**.

**The PIN is deliberately not synced to Drive** — each device keeps its own.

### What the PIN does not do

- **It does not encrypt anything.** It is a screen overlay. The data sits in `localStorage` in the clear and is readable by anyone with the device unlocked and a browser console.
- **The brute-force cooldown is in memory** and resets on reload.
- **Background timers keep running while locked** — the app continues syncing behind the lock screen.
- Web Crypto requires a secure context, so the PIN only works over HTTPS.

### What a native rebuild should do instead

- Store the shared file's credentials in **Keychain / Keystore**, not in plain app storage.
- Use an **encrypted database** (SQLCipher, or platform file-level encryption) for the state.
- Offer **biometric unlock** backed by the platform's secure enclave, with the PIN as fallback.
- Keep the brute-force counter in persistent storage so a restart does not clear it.
- Treat the backup export as sensitive: it is the whole family's financial history in one file.

---

## 16. Storage keys

| Key | Contents | Synced |
|---|---|---|
| `sip_state_v2` | the entire `ST` object | ✅ (it *is* the synced document) |
| `sip_auth` | `{token, exp, name, photo, sub, email}` | ❌ |
| `sip_pin` | `{salt, hash, len}` | ❌ deliberately per-device |
| `sip_file_<googleSub>` | the Drive file id for that account | ❌ |
| `sip_localcopy` | `{at, data}` — last known-good synced payload | ❌ |
| `sip_backup` | state snapshot taken before each merge | ❌ |

---

# Part V — Interface

## 17. Design system

### Colour tokens (`index.html:21`)

| Token | Value | Role |
|---|---|---|
| `--navy` | `#1B2A4A` | top bar, splash background |
| `--teal` | `#0D7377` | primary accent, active nav, theme colour |
| `--gold` | `#D4920A` | warnings, pending states |
| `--blue` | `#1565C0` | Equity category, Husband |
| `--green` | `#1A7A4A` | income, positive balances, Daughter |
| `--red` | `#C62828` | expense, negative balances |
| `--pink` | `#880E4F` | accent |
| `--purple` | `#4527A0` | Wife |
| `--bg` | `#F4F6FB` | page background |
| `--card` | `#FFFFFF` | surfaces |
| `--border` | `#DDE3EE` | hairlines |
| `--text` | `#1a1a2e` | primary text |
| `--sub` | `#64748B` | secondary text |
| `--muted` | `#94A3B8` | tertiary text, disabled |
| `--safe-top` / `--safe-bot` | `env(safe-area-inset-*)` | notch and home-indicator padding |

Category colours: Equity `#1565C0`, Hybrid `#7B1FA2`, Debt `#00838F`, Fixed Deposit `#EF6C00`, Government `#2E7D32`.

The palette is **light-only**. A rebuild adding dark mode must define a full second token set rather than inverting.

### Type scale

Roughly: 10px tertiary metadata · 11–11.5px labels and hints · 12.5px list rows · 13px section titles (uppercase, letter-spaced) · 15px emphasised values · 17px top-bar title · 22px nav icons. System font stack.

### Components

| Class | Role |
|---|---|
| `card` | 14px radius, 1px border, white surface — the base container |
| `card-title` | uppercase, letter-spaced, muted section heading |
| `stat-card` / `stat-grid` | two-up headline figures |
| `lgd-s` | compact stat tile inside a dashboard strip |
| `cons-card` | grid card: name, big value, sub-line, progress bar; `.tap` adds an action row and makes it a target |
| `cl-row` | list row: rank, name, bar, right-aligned value and share; the list counterpart of `cons-card` |
| `gro-row` | detail row: date · bar · amount · optional full-width tag line |
| `hb-row` | labelled progress bar with a value on the right |
| `pot-row` | payment-type balance row: name, in, out, balance |
| `pay-tabs` | horizontally scrolling filter chips |
| `pay-tot` | "N entries · ₹total" header above a detail list |
| `pay-sum` | head summary strip above a detail list |
| `view-sw` | Cards ↔ List segmented switch |
| `donut-wrap` / `donut-center` | SVG ring with a figure and label in the middle; the SVG is rotated −90° so the first slice starts at twelve o'clock |
| `legend-row` | dot, name, and right-aligned amount · share — the ring's key |
| `tc-wrap` / `tc-line` / `tc-base` / `tc-x` | the trend chart: a scaled SVG, two 2px polylines with `vector-effect: non-scaling-stroke`, a hairline floor, and axis labels |
| `don-sw` | the Heads / Subheads switch, a `view-sw` sitting under a card title |
| `freq` / `freq-ed` / `freq-add` | a Buy Again row: tappable body, edit, and add-to-list |
| `toast` | transient bottom message, 2.5 s default |
| `lock-screen` | full-screen PIN overlay, z-index 500 |
| `auth-gate` | full-screen sign-in overlay, z-index 400 |

Bars and chips carry meaning: teal for neutral magnitude, green for income, red for expense, dashed border for an excluded head.

### Layout

Single column, mobile-first, `max-width` unconstrained. Bottom nav is fixed with safe-area padding; body reserves `70px + safe-bottom`. Wide content scrolls inside its own container rather than the page.

---

## 18. Interaction patterns

**Searchable, creatable selects** (`makeSearchable`). The native select stays as the value holder; a text input filters options. Marked `data-create`, typing an unmatched name **creates the record and selects it** in one action. **Rule: every dropdown opens blank, with nothing preselected.**

**Period selector.** Year and month, present on every dashboard, **defaulting to the running month**. Choosing a blank month switches to whole-year mode, which changes drill-downs from day-level to month-level.

**Cards ↔ List.** A segmented switch on breakdowns. The choice applies to the top level only; a drill-down is always a list, because it shows individual entries.

**Drill-down.** Tapping a card opens a detail view with a header total, the entry list, and a back link. The switch that governs the top level is hidden inside a drill-down.

**Editing in place.** Anything a person typed can be corrected where it is shown, not only where it was created: a ledger entry from the Log, a purchase from History, an item from Buy Again. The rule is that an edit which changes a *displayed name* must also update every denormalised copy of it, or the history splits.

**Confirmations.** Destructive actions confirm and say what will happen ("Remove it anyway? Those entries keep the value but show no label."). **Exceptions that deliberately do not confirm:** End Now (ending your own access), and toggling a tracker cell.

**Feedback.** Every write toasts. Silent background syncs do not; a sync that actually changed something announces "Family data updated".

**Modals.** Centred sheet, backdrop, explicit Cancel and Save, inline error line rather than an alert. Used for entry edit, head edit, buy, PIN setup, and the enlarged app icon.

---

## 19. Copy and locale

- **English only.** This was corrected repeatedly during development; no Hindi anywhere, including seeded item names.
- **Currency** — `fmt(n)` returns `"₹" + n.toLocaleString("en-IN")`, giving Indian digit grouping (₹1,00,000).
- **Dates**
  - `dayLabel("2026-08-03")` → `"03 Aug"`
  - `monthLabel("2026-08")` → `"Aug 2026"`
  - `stampLabel(ms)` → `"3 Aug 2026, 5:48 pm"` — used for created/edited stamps
  - `sinceLabel(ms)` → `"just now"`, `"12 min ago"`, `"3 hours ago"`, `"yesterday"`, `"9 days ago"`
- **Storage format is always `YYYY-MM-DD`**, never a locale string. Purchase timestamps are set to **noon** to avoid timezone rollover.
- Tone: plain, specific, no exclamation marks. Errors say what to do next.

---

# Part VI — Rebuild guidance

## 20. What to keep, what to change

| Area | Verdict | Why |
|---|---|---|
| Data model ([§4](#4-entity-reference)) | **Keep** | Existing family data must import unchanged |
| Seed data ([§6](#6-seed-data)) | **Keep exactly** | Ids are referenced by live rows |
| Accounting rules ([§8](#8-the-accounting-model)) | **Keep** | The most-iterated logic in the app |
| Merge algorithm ([§12](#12-sync-engine)) | **Keep** | Proven convergent; port the semantics literally |
| Deletion via null + tombstones ([§7](#7-deletion-semantics)) | **Keep** | Required for the merge to be correct |
| Access model ([§14](#14-access-control)) | **Keep** | Including the `adminCount()===0` bootstrap |
| Visual language ([§17](#17-design-system)) | **Keep** | The family already knows it |
| **OAuth implicit flow** | **Replace** | Unavailable to native apps. Use authorization-code + PKCE (AppAuth). This yields a **refresh token**, which removes the hourly expiry and every defect in [§13.3](#133-known-defects--do-not-reproduce) |
| `localStorage` | **Replace** | Use SQLite / Room / Core Data — but keep `meta` and `tomb` as columns |
| PIN as a screen overlay | **Strengthen** | Biometrics + encrypted store ([§15](#15-security-model--honest-limits)) |
| Single-file `index.html` | **Replace** | Real modules, real tests |
| Fixed 60-month `MONTHS` array | **Replace** | Compute the range |
| Drive as the only backend | **Reconsider** | Fine at family scale. A real backend becomes worth it if the Testing-mode test-user ceiling or the restricted-scope review ever bites |

### Session expiry, specifically

The user's stated requirement is **"don't expire the session"**. On native this is straightforward: the authorization-code + PKCE flow returns a refresh token, so the app renews access silently and indefinitely without any hidden-iframe trickery, and without third-party-cookie problems. A re-authentication prompt then happens only when the user revokes access, changes their password, or signs out.

Regardless of flow, the app should **never blank itself** because a token lapsed. All data is local; a dead token means only that sync is paused. Degrade to a quiet "Reconnect" affordance and keep every screen usable.

---

## 21. Recommended architecture

Deliberately platform-agnostic; the sections above are the specification, this is one way to satisfy it.

```
┌───────────────────────────────────────────┐
│ UI layer            screens per §3        │
├───────────────────────────────────────────┤
│ View models         one per screen        │
├───────────────────────────────────────────┤
│ Domain services     Accounting  §8        │
│                     Investment  §9        │
│                     Shopping    §10       │
│                     Recurring   §11       │
│                     Access      §14       │
├───────────────────────────────────────────┤
│ Repository          reads, writes,        │
│                     stamps meta/tomb      │
├─────────────────┬─────────────────────────┤
│ Local store     │ Sync service            │
│ SQLite          │ merge §12 · Drive REST  │
└─────────────────┴─────────────────────────┘
```

**Non-negotiables for the data layer**

1. **Every write stamps `meta`.** Make this impossible to forget — no code path outside the repository may mutate state.
2. **Deletes write tombstones**, never hard-delete rows that sync.
3. **The local store is the source of truth for reads.** The UI never waits on the network.
4. **Sync is idempotent and re-entrant-safe**, and always read-merge-write.

**Suggested stack** — Flutter or React Native for one codebase across both phones; SQLite via Drift/WatermelonDB; `flutter_appauth` / `react-native-app-auth` for OAuth; platform background-fetch for periodic sync; the existing Drive REST v3 calls unchanged.

**Schema sketch**

```sql
CREATE TABLE ledger (
  id TEXT PRIMARY KEY, date TEXT, type TEXT, head TEXT, sub TEXT,
  card INTEGER, pay TEXT, amt REAL, note TEXT, by_person TEXT,
  src TEXT, src_key TEXT, created_by TEXT, created_at INTEGER,
  edited_by TEXT, edited_at INTEGER,
  meta_ts INTEGER NOT NULL
);
CREATE TABLE tombstones (kind TEXT, row_id TEXT, deleted_at INTEGER,
                         PRIMARY KEY (kind, row_id));
CREATE TABLE field_meta (kind TEXT, key TEXT, ts INTEGER,
                         PRIMARY KEY (kind, key));
```

The Drive document keeps its current JSON shape, so old and new clients can coexist during migration.

---

## 22. Behaviour catalogue for tests

The current build is pinned by 471 tests across 18 suites. Those harnesses will not survive the rebuild, but the invariants must. Each line below is an acceptance criterion.

**Accounting**

1. An expense counts in Spent regardless of payment method.
2. A row whose head has `bill` or `excl`, or which is flagged `card`, never appears in Spent.
3. Only payment types not marked excluded reduce Net.
4. A card bill paid from the bank reduces Net but is not Spent.
5. Cash income minus cash spending is the cash pot; it never touches the bank pot.
6. Card outstanding = card purchases − bill payments, cumulative across all time.
7. Net reconciles exactly against the counting types' pot outflow.

**Counts and drill-downs**

8. The entry count on a payment-type card equals the number of rows in its drill-down — including when several entries share one head.
9. The entry count on a Grocery card equals the number of rows in its breakdown — including when several entries share one date.
10. Excluded rows are absent from both the totals and the counts, and are reported separately as "settled".
11. A year breakdown groups by month, newest first; a month breakdown lists individual entries, newest first.

**Investment**

12. Invested is computed per plan per month; a partly-paid month counts for the plans that were paid.
13. A finalised month keeps its snapshot amounts; editing a plan afterwards changes only unfinalised months.
14. Unlocking a month discards the snapshot.
15. Ticking a cell posts an expense dated the 1st of that month.

**Shopping**

16. Mixed units aggregate through the base unit (500 g + 1 kg = 1.5 kg).
17. Merging items re-points existing history to the target and its name.
18. Alias chains resolve, and cycles terminate.
19. Consumption is sorted most-used first, with the running month preselected.
20. Renaming an item updates every purchase row filed under it, so its history stays one line.
21. Renaming to a name another item already owns is refused, with a pointer to Merge items.
22. Editing an alias edits the item it resolves to.

**Charts**

23. Every ring's slices sum to 100% and the first starts at twelve o'clock.
24. More than seven heads collapse to six plus *Everything else*, whose amount is the sum of the rest.
25. A period with no income renders no income chart at all, rather than an empty ring.
26. Slice colour follows rank, so the same period renders identically on every device.
27. A head name containing markup is escaped, in both the ring and its legend.
28. Subhead slices total exactly what the head slices totalled; entries with no subhead are kept under the head's own name.
29. The Heads / Subheads switch is hidden when no entry in that kind carries a subhead.
30. Expenses and income remember their head/subhead choice independently.
31. The trend line reads day by day for a month and month by month for a year.
32. Trailing empty periods are dropped; interior empty periods stay as zeros.
33. A settlement never appears as a spike in the spending line.
34. Fewer than two points renders no chart at all.

**Recurring**

35. Running twice never produces a duplicate for the same rule and month.
36. Day 31 in February files on the last day of February.
37. Pausing stops future months; already-posted entries stay.

**Sync**

38. Two devices editing different fields of the same record both keep their edits.
39. A delete beats a concurrent edit stamped earlier, and loses to one stamped later.
40. Rows generated independently on two devices from the same `srcKey` collapse into one.
41. A merge that changes nothing produces no "updated" notice.
42. When several copies of the file exist, the oldest is chosen.

**Access**

43. A member with no grant cannot open a gated module, by tile or by tab.
44. A grant expires exactly 30 minutes after it is issued, and the member is ejected within 20 seconds.
45. Revoke and End Now take effect immediately and propagate.
46. With no admin flagged anywhere, every signed-in user is treated as an admin.
47. Two person rows with the same email are merged, and the admin flag survives.

**Session**

48. A lapsed token never blanks the app or discards the identity.
49. Backup download and upload are refused to non-admins.

---

## 23. Known issues to fix in the rebuild

| Issue | Location | Note |
|---|---|---|
| Silent token renewal never lands; sessions die at ~59 min | `index.html:2124`, `:2226` | The `postMessage` has no listener after first login. See [§13.3](#133-known-defects--do-not-reproduce) |
| A lapsed token evicts the user and blurs the whole app | `sessionExpired()` `:2179`, `updateAuthGate()` `:2296` | Should degrade to local-only |
| A failed refresh is never retried | `scheduleTokenRefresh()` `:2116` | One missed refresh ends the session permanently |
| Sign-in popup leaks a `message` listener per cancelled attempt | `handleLogin()` `:2226` | |
| HTTP 403 is never handled on any Drive call | throughout | Drive returns 403 for rate limits and quota, not only auth |
| An auth failure in `checkFile()` deletes the cached file id | `:2417`, `:2523` | Should distinguish "gone" from "unknown" |
| Comment above `payGroups` contradicts the code | `:4048` | Says excluded rows are included; the function now filters them out |
| `lockNow()` is dead code | `:6212` | The lock button opens the settings modal instead |
| PIN brute-force counter resets on reload | `:6033` | Held in memory only |
| `MONTHS` is a hard-coded 60-entry array ending Aug 2031 | `:1168` | Should be computed |
| Tombstones are never pruned | `:2745` | Fine at family scale; unbounded in principle |

---

## Appendix — file map of the current build

| File | Role |
|---|---|
| `index.html` | The entire application: markup, CSS and one inline `<script>` (~6,236 lines) |
| `sw.js` | Service worker — network-first for navigation so a deploy is picked up immediately, cache-first for assets; cache name is version-bumped on every release |
| `manifest.json` | `name: "Shree Laxmi Home"`, `short_name: "Shree"`, standalone, portrait, `#1B2A4A` background, `#0D7377` theme |
| `icons/favicon-32.png` | Browser tab icon |
| `icons/icon-192.png` | Home-screen icon, `any maskable` |
| `icons/icon-512.png` | Splash icon, `any maskable` |

External dependency: **SheetJS 0.18.5** from cdnjs, used only for the Excel export.

**Deployment.** GitHub Pages at `https://nirajpandey3453.github.io/PWA/`. Because the service worker caches aggressively, `CACHE` in `sw.js` must be bumped on every deploy or installed clients keep serving the old build.
