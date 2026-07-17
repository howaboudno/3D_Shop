# 3D Shop App — Specification of Record

**Owner (dev/admin):** Aboud
**Seller (key user):** Petey
**Buyers:** friend group
**Version:** v1 spec — Phase 0 resolved
**Last revised:** 2026-07-17

---

## 1. Purpose

A web app where Petey manages 3D print orders received from friends. Buyers submit a request with an STL and images; Petey quotes it manually (print time, filament weight, materials), the buyer accepts, Petey prints. Orders carry a comment thread and a materials list the buyer may be asked to purchase.

**Explicit framing:** this is not a business. The friend group is collectively funding Petey's hobby. v1 economics are deliberately generous and uneven. Fairness mechanics are a scaling concern, not a v1 concern.

---

## 2. Roles

Two roles, not three.

| Role | Who | Notes |
|---|---|---|
| Seller | Petey | Key user. Owns settings/pricing. No separate admin UI. |
| Buyer | friends | Register + login required to order. |

`is_superuser` flag on Aboud's account for dev access. **Decision: no third admin interface.**

Guests may view the welcome page and portfolio. Everything else requires login.

---

## 3. Core decisions (resolved — do not relitigate without editing this section)

| # | Topic | Decision | Rationale / trigger to revisit |
|---|---|---|---|
| D1 | Print time + filament weight | **Seller enters manually** after slicing locally | Server-side slicing too heavy; volume estimates too inaccurate. Requires a `Quoted` state. |
| D2 | Consumables charge | **Purchase-only (model b).** No per-gram charge for filament drawn from stock. A buyer pays only for materials Petey must newly buy for that job. | Friend group subsidises Petey. See §5.1 for the v2 migration path to per-gram. |
| D3 | Price snapshotting | **Mandatory.** All rates copied onto the order at accept time (T4). Settings changes never re-price historical orders. | Single most likely silent bug otherwise. |
| D4 | Money storage | **Integer cents.** Never float. | — |
| D5 | Privacy | **Half-privacy.** Petey's address is known within the friend group. Materials are shipped to him via wishlist (address hidden by retailer); delivered prints carry his return address. | On scale-up: PO box for return address. |
| D6 | Retailer integration | **None.** Each material has an optional `purchase_url`. Petey builds one wishlist per order and pastes the link. | Amazon cart-add URL schemes are not dependable for third parties. Verify before ever scoping this. |
| D7 | Supplier dimension | **Amazon wishlist as default for everything**, accepting higher prices, because it hides the delivery address. No multi-supplier schema. | — |
| D8 | Stock tracking | **Not in v1.** Petey knows what's on his shelf. | v2/v3, likely as a local interface. |
| D9 | Printer queue | **Not in v1.** Petey accepts or declines based on whether he can hit the deadline. | — |
| D10 | Virus scanning | **Skip.** Extension + magic-byte check, size cap, no execution, served from a non-rendering path. | STLs are inert geometry. |
| D11 | Shipping | **Manual tier selection by seller** during quoting. Cost is inside the quote the buyer accepts. | Flat rate rejected: 200g vs 2kg differ materially in NL. No size×weight matrix. |
| D12 | Failed print / reprint | **Undefined in v1**, acceptable because the buyer bought the materials. | Revisit if it happens twice. |
| D13 | Auth policy | **Open registration** in v1. | Spam prevention + invite codes deferred. |
| D14 | Payments | **Interface prepared, not implemented.** | — |
| D15 | Files | **Object storage, not the app server.** Direct browser upload via presigned URL. The API reads at most the first 84 bytes of an STL, via ranged GET, to validate it. | Cannot be retrofitted cheaply. |
| D16 | Database | **Postgres, not SQLite.** | Concurrent writers, migrations, longer life than the predictions app. |
| D17 | Portfolio "order this" | Creates a **new Draft order prefilled with identical parameters**, editable by the buyer before submitting. Not auto-accepted. | — |
| D18 | Order refs | Human-readable sequential refs (`ORD-0042`) from a Postgres sequence, not UUIDs, for use in wishlist names. | — |
| D19 | Electricity | **Folded into the print fee tier.** No separate line, no `printer_watts` setting, no electricity rate setting. | A 350 W printer bills ~20c on a long print; the tier absorbs it. Consequence: the tier is a **blended** labour + machine + energy number and must be revisited by hand if energy prices move. Nothing in the app will tell you. |
| D20 | STL viewer | **None in v1.** Petey opens the file in his slicer, which is where `print_hours` and `grams_used` come from anyway (D1). Images render inline with a lightbox. | A browser viewer shows Petey less than the tool he is already required to open. Trigger: someone other than Petey needs to inspect geometry. |

---

## 4. Pricing formula

Frozen. The entire app is a UI over this expression.

```
order_total =
    print_fee              # tier lookup on manually entered print_hours (snapshotted)
  + consumables_charge     # v1: always 0 (see D2)
  + shipping               # manual tier; only if delivery, not pickup
  + buyer_purchased_total  # materials the buyer must buy for this job

payout_to_seller = order_total - buyer_purchased_total
```

Rules:
- Every term is stored as its **own line row** on the order, together with the rate it was derived from. Never a single total.
- `buyer_purchased_total` is paid **directly by the buyer via the wishlist**, so it is €0 in Petey's pocket but is inside the number the buyer accepts. Nothing is sprung after acceptance.
- `payout_to_seller` is displayed to the seller only, labelled **"Your payout"**. Not "profit" — the app tracks none of Petey's costs (stock filament, machine wear, his time), so it cannot make that claim.
- Print tiers are **labour + machine + energy**, blended (D19). They do **not** scale with filament price; scaling them with filament price would double-count materials.
- `grams_used` is recorded on every order **even though v1 charges nothing for it** (see §5.1).
- **Rounding rule: round each line to cents, then sum.** Not sum then round. No v1 fixture exercises this (every input is already whole cents), but per-gram charging reintroduces it immediately — see §5.1.

### 4.1 "Dynamic cost" — definition

Dynamic means: the cost of a print is not a fixed number. It is derived from rates Petey maintains (current spool price, current tier table, current shipping tiers) at the moment of quoting. It does **not** mean automatic recalculation of existing orders — see D3.

### 4.2 Tier tables

Boundaries are half-open and leave no gap. A 6 h print is tier P1; a 12 h print is tier P2.

| Tier | Print time | Fee (cents) |
|---|---|---|
| P1 | 0 h – 6 h inclusive | 500 |
| P2 | over 6 h – 12 h inclusive | 1000 |
| P3 | over 12 h | 2500 |

| Tier | Shipping | Cost (cents) |
|---|---|---|
| S0 | Pickup | 0 |
| S1 | Brievenbuspakje (≤2 cm thick, ≤2 kg) | 1000 |
| S2 | Pakket ≤10 kg | 1500 |
| S3 | Pakket >10 kg | TBD |

All values above are **defaults, editable in settings**. Thresholds and amounts both.

### 4.3 Worked examples (test fixtures)

These are the Phase 4 unit tests, not documentation. All values in cents.

| # | Tier | Print time | Grams | print_fee | consumables | shipping | buyer_purchased | **Buyer pays** | **Your payout** |
|---|---|---|---|---|---|---|---|---|---|
| A | P1 | 2 h | 25 | 500 | 0 | 0 (S0 pickup) | 0 (from stock) | **500** | **500** |
| B | P2 | 7 h | 102 | 1000 | 0 | 1000 (S1) | 3500 (spool 2500 + primer 1000) | **5500** | **2000** |
| C | P3 | 16 h | 255 | 2500 | 0 | 1500 (S2) | 6000 (spool 2500 + paint 2500 + brushes 500 + sealing spray 500) | **10000** | **4000** |

Boundary cases to add as tests: exactly 6 h → P1. Exactly 12 h → P2. Zero grams, pickup, no materials → print_fee only.

---

## 5. Deliberate v1 debts

### 5.1 Migration path to per-gram charging (model a)

When the app scales beyond the friend group, D2 must change to: **every order pays `grams_used × price_per_gram`, and a buyer who purchases a spool receives that spool's cost as a credit against their orders.**

To keep that migration cheap, v1 must already:
- record `grams_used` on every order;
- hold `price_per_kg` per material in settings (already needed to quote wishlist purchases);
- keep `consumables_charge` as a real line item in the schema, valued at 0;
- record which order triggered a buyer purchase, and the amount.

The credit concept (a buyer-level balance) is the only genuinely new thing model (a) needs. Do not build it now.

**This is also when the rounding rule stops being theoretical:** 102 g × €25/kg ÷ 1000 = 255 cents exactly, but 103 g = 257.5.

### 5.2 Other deferred items

Payments · stock tracking · supplier dimension · retailer basket integration · invite codes and spam prevention · GDPR deletion endpoint (privacy statement is a static page in Phase 1) · printer queue · reprint/failure policy · footer contact + bug report page · **STL viewer (D20)**.

---

## 6. Status model

### 6.1 States

| Status | Meaning | Terminal |
|---|---|---|
| Draft | buyer is building the request | no |
| Placed | submitted, no response yet | no |
| Quoted | seller has entered print_hours, grams, materials, total | no |
| Accepted | buyer accepted the quote — **prices snapshot here** | no |
| AwaitingMaterials | buyer must purchase from the wishlist | no |
| InProgress | printing / post-processing | no |
| Finished | ready for pickup or shipped | no |
| Completed | buyer (or seller) confirms receipt | **yes** |
| Declined | seller refuses the job | **yes** |
| Cancelled | buyer withdraws, rejects a quote, or seller can no longer deliver | **yes** |

Flag, not a state: `needs_info` — toggled in place on any non-terminal status. Either party raises it, seller clears it.

### 6.2 Transitions (the state machine)

Single source of truth. Anything not in this table returns **409**.

| # | From | To | Who | Trigger |
|---|---|---|---|---|
| T1 | Draft | Placed | buyer | submits the request |
| T2 | Placed | Quoted | seller | enters print_hours, grams, materials, shipping tier |
| T3 | Placed | Declined | seller | won't take the job |
| T4 | Quoted | Accepted | buyer | accepts the number — **snapshot fires** |
| T5 | Quoted | Quoted | seller | re-quotes; wipes and rewrites the quote lines |
| T6 | Accepted | AwaitingMaterials | seller | materials must be bought |
| T7 | Accepted | InProgress | seller | nothing to buy — skips T6 |
| T8 | AwaitingMaterials | InProgress | seller | confirms materials arrived |
| T9 | AwaitingMaterials | Quoted | seller | buyer declined the extra spend; re-quote without it |
| T10 | InProgress | Finished | seller | printed, ready |
| T11 | Finished | Completed | buyer | confirms receipt |
| T12 | Finished | Completed | seller | buyer forgot to confirm |
| T13 | Draft, Placed, Quoted, Accepted, AwaitingMaterials | Cancelled | buyer | withdraws, or rejects the quote |
| T14 | Accepted, AwaitingMaterials | Cancelled | seller | can no longer deliver |

### 6.3 Rules attached to the table

- **T4 is the only edge with an irreversible side effect.** Everything before it is cheap; everything after has frozen money attached (D3).
- **T5 wipes the previous quote lines rather than versioning them.** v1 keeps no quote history — `order_status_history` records that a re-quote happened, not what it said.
- **Cancelled is deliberately blurred.** Buyer-rejects-quote and buyer-changes-mind are the same state. A `cancel_reason` free-text field carries the difference; no extra status.
- **T12 exists because orders stall.** A friend forgets to tick "received" and the order sits in Finished forever.
- **Terminal means terminal.** No edges leave Completed, Declined or Cancelled. A revived order is a new order.

---

## 7. Data model (ERD)

```
users(id, email, name, role, is_superuser, notify_by_email, created_at)

orders(id, ref, buyer_id, title, description, deadline, delivery_method,
       dimensions, notes, status, needs_info, cancel_reason,
       print_hours, grams_used, shipping_tier,
       quoted_at, accepted_at, created_at)

order_lines(id, order_id, kind, label, qty, unit_rate_cents, amount_cents)
    # snapshot, written at T4. kind: print_fee | consumables | shipping | buyer_purchased

order_status_history(id, order_id, from_status, to_status, actor_id, at)

materials(id, name, unit, price_per_unit_cents, price_per_kg_cents,
          purchase_url, active)

order_materials(id, order_id, material_id, qty, buyer_purchased, purchase_url,
                amount_cents, purchased_confirmed_at)

messages(id, order_id, author_id, body, created_at)
order_read_state(order_id, user_id, last_read_at)          # PK (order_id, user_id)

attachments(id, order_id, kind, storage_key, filename, size_bytes, created_at)
    # kind: stl | image

portfolio_projects(id, title, description, published, sort, created_at,
                   prefill_description, prefill_dimensions, prefill_notes,
                   source_stl_key, materials_snapshot, duration_snapshot,
                   cost_cents_snapshot)

portfolio_images(id, project_id, storage_key, sort)

settings(key, value)   # print tiers P1..Pn, shipping tiers S0..Sn, material prices
```

Notes:
- `order_materials.purchase_url` overrides `materials.purchase_url` — materials are a catalog **plus** per-order links, not a pure catalog.
- `materials.active` exists because a material referenced by `order_materials` can never be deleted. Soft-delete or the catalog is append-only forever.
- `prefill_*` are what "order this" (D17) copies into the new Draft. `*_snapshot` are frozen display text on the project page, never joined back to an order. Two different jobs.

---

## 8. Seller interface

### Dashboard (welcome)
Tabs: Open orders · Quoted · Accepted · Awaiting materials. Plus small metrics: order counts, total costs, total print hours. **Built last** — aggregate queries over data that must exist first, read off `order_lines`, never recomputed from settings.

### Orders page
List of all orders, opens into order detail.

### Order detail — wireframe

The screen where every hard thing collides. Every button maps to a transition in §6.2; a button with no transition should not exist.

```
+--------------------------------------------------------------------+
| ORD-0042  ·  Ivan's dragon              [Quoted]  [! needs info]    |
| from Ivan · placed 12 Jul · deadline 2 Aug · DELIVERY               |
+---------------------------------------+----------------------------+
| DESCRIPTION                           | QUOTE                      |
| hollow dragon, painted red,           |   print hours [ 7.0 ]      |
| no supports, ~12cm tall               |   grams       [ 102 ]      |
|                                       |   -> tier P2       10.00   |
| dimensions: ~12cm tall                |                            |
| notes: red + gold drybrush            | MATERIALS         [+ add]  |
|                                       |   spool 1kg  25.00 [x]buyer|
| FILES                                 |     url [ amzn.to/...    ] |
|   [dragon.stl  4.2 MB  download]      |   primer     10.00 [x]buyer|
|   [img] [img] [img]   <- lightbox     |     url [ amzn.to/...    ] |
|                                       |                            |
| ------------------------------------- | SHIPPING                   |
| CHAT                            (2)   |   [ S1 brievenbuspakje v ] |
|   Ivan   can you do gold?             |                     10.00  |
|   Petey  I'd need paint               |                            |
|   [ type a message...      ] [send]   | -------------------------- |
|                                       |   print fee         10.00  |
|                                       |   shipping          10.00  |
|                                       |   buyer buys        35.00  |
|                                       | ========================== |
|                                       |   Buyer pays        55.00  |
|                                       |   Your payout       20.00  |
|                                       |                            |
|                                       | [Re-quote T5] [Decline T3] |
+---------------------------------------+----------------------------+
```

Reading the wireframe:
1. **No STL viewer** (D20) — the file is a download. Petey opens it in his slicer, which is where the numbers in the QUOTE block come from.
2. **Images open in a lightbox**, not inline at size.
3. **"Your payout"** is seller-only and appears nowhere in the buyer's mirror of this screen.
4. **Buttons are transition-labelled.** T2 (Quote) replaces T5 (Re-quote) when the order is still `Placed`. T6/T7/T8/T10 appear as the order advances. A disabled button is a transition the current role may not pull.
5. **Open question:** chat inline makes this screen tall. If chat becomes a tab, it is a second endpoint and a second fetch — decide before Phase 6.

### Settings page (Petey only)
- Material price per unit
- Material price per kg
- Print tiers P1..Pn — thresholds and amounts both editable (§4.2)
- Shipping tiers S0..Sn — labels and amounts editable (§4.2)

No electricity rate. No printer wattage (D19).

**Note:** the old "Materials page" is dissolved. It was doing four jobs. Inventory → dropped (D8). Price catalog → Settings. Cost aggregation → order detail. Metrics → Dashboard.

### Portfolio management
Post completed projects with multiple images + snapshot of materials, duration, cost. Publishing makes it visible to buyers and guests.

---

## 9. Buyer interface

### Welcome page
Header: shop name + login/register. Top: welcome text, rules and restrictions for print requests. Create order button. View orders button. Bottom: carousel of published portfolio projects. Footer: deferred.

### Create order page
Title, description, deadline, delivery-or-pickup, dimensions (free text), post-processing notes. Upload images + STL (presigned direct upload, 50 MB cap on STL — upload time and storage, not rendering). Comment thread with email-notify opt-in (same address as registration). Polling, not websockets.

### View orders page
- List of placed orders with status (§6.1)
- Per order: the same detail screen as §8, inputs read-only, no "Your payout"
- Accept / decline the quote (T4 / T13)
- When materials are required: see the list + cost, accept or decline the additional spend, click through to the wishlist, tick "purchased"
- Mark a Finished order as Completed on receipt (T11)

### Portfolio page
All published projects. Open a project → images, snapshot cost, snapshot materials. "Order this" → new prefilled, editable, unsubmitted Draft (D17).

---

## 10. Stack

| Layer | Choice | Note |
|---|---|---|
| Backend | Python / FastAPI | known quantity |
| DB | **Postgres** | Railway / Neon / Supabase free tier. Local dev via docker compose. |
| Frontend | React / Vite | |
| Files | **S3-compatible object storage** — Cloudflare R2 (free egress) or Backblaze B2 | presigned direct upload; the API reads at most the first 84 bytes, via ranged GET, to validate an STL |
| Email | SMTP provider — Resend / Postmark free tier | needed from Phase 6; domain + SPF/DKIM required |
| Hosting | Vercel (frontend) + Railway/Fly (API), or a €5 Hetzner VPS with Docker Compose | open |

No three.js — the STL viewer is out (D20).

---

## 11. Materials used (current, informational)

1. Filament (Hyper PLA — colour undecided)
2. Superglue / hot glue
3. Sandpaper (80, 120, 240, 320 grit)
4. Wood filler (TCX brand)
5. Primer
6. Paint (Spectrum brand)
7. Brushes
8. Sealing spray

---

## 12. Change log

| Date | Change |
|---|---|
| 2026-07-17 | Initial rewrite from `outline_app.txt`. Resolved: role count (3→2), privacy scope, quoting method, consumables model, supplier dimension, shipping, stock, queue, virus scanning, DB, file storage. Added: pricing formula, state machine, ERD, migration path to per-gram. |
| 2026-07-17 | **Phase 0 resolved.** Electricity added as its own line, then folded into the print tier (D19) — `printer_watts` and electricity rate settings removed. Tier boundaries defined as half-open (§4.2). Three worked examples promoted to fixtures with expected totals in cents (§4.3). Rounding rule stated and noted as unexercised until §5.1. State machine written as a transition table T1–T14 (§6.2) with rules (§6.3); status table demoted to a node list (§6.1). ERD amended: `order_status_history`, `orders.cancel_reason`, `order_read_state`, `users.notify_by_email`, `materials.active`, portfolio `published`/`sort` and `prefill_*` split from `*_snapshot` (§7). STL viewer dropped (D20), three.js removed from stack, 50 MB cap re-justified. `payout_to_seller` defined and labelled "Your payout" (§4, §8). Seller order detail wireframed (§8). |