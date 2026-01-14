# Ledger + Stock Flow (Priemka / Otgruzka / Corrections) — Party stays correct

This is the **clean business flow** for how your warehouse should behave using:

- **Ledger tables (source of truth)**:
  - `warehouse_transactions`
  - `warehouse_transaction_items`
- **Stock tables (derived, but persisted for performance)**:
  - `warehouse_stock_items` (FIFO layers / “yacheyka” rows)
  - `warehouse_stocks` (aggregated balances)
  - `warehouse_transaction_stock_items` (allocation edges for outcomes)

The key improvement vs your current mental model is:

> **Corrections must adjust the exact layers created by the original Priemka items** (linked by `warehouse_transaction_items_id`), not “the current total stock for the party/product”.

This prevents “Party drift” when you already had later Priemkas and/or Otgruzkas.

---

## 1) Core rules

### 1.1 Party (Партия)

- Party = `batches_id`.
- Every receiving item must carry `batches_id` so party ownership is explicit.

### 1.2 Receiving (Приемка) creates stock layers

When you create Priemka:

1) Write ledger:

- `warehouse_transactions` with `type=["receiving"]`
- `warehouse_transaction_items` rows, each has:
  - `products_id`
  - `batches_id` (Party)
  - received quantity fields (weight/count/peredok/zadok)

2) Create stock layers:

- For each `warehouse_transaction_item` you create at least one `warehouse_stock_items` row:
  - `batches_id` = party of the item
  - `warehouse_id` = where it arrived
  - `available_weight` (and counts)
  - **`warehouse_transaction_items_id` = the receiving item GUID**

3) Update aggregate:

- Recompute `warehouse_stocks` (or incrementally update).

**Important:** for correctness with corrections, do **not lose** the link between a stock layer and the receiving item that created it.

    ### 1.3 Shipment (Отгрузка) consumes FIFO layers and creates allocation edges

    When you create Otgruzka:

    1) Write ledger:
    - `warehouse_transactions` with `type=["shipment"]`
    - `warehouse_transaction_items` rows for the shipped products/qty

    2) Consume stock:
    - Decrease `warehouse_stock_items.available_weight` by consuming FIFO.
    - For each consumption, create `warehouse_transaction_stock_items` edges:
        - `warehouse_transaction_items_id` = shipment item
        - `warehouse_stock_items_id` = consumed layer
        - `consumed_weight`

    3) Update aggregate:
    - Recompute/update `warehouse_stocks`.

    ### 1.4 Oversell (negative stock) is allowed

    If shipments exceed available stock, you still create the shipment.

    To represent deficit deterministically **without using `batches_id = NULL` or a fake deficit batch**:

    - Keep `warehouse_stock_items.available_weight` **non-negative** for all real party layers (min 0).
    - Allow `warehouse_stocks.available_weight` to go **negative** (product-level shortage).

    If a correction reduces an old receiving so much that its original yacheyka would go negative:

    1) Clamp that yacheyka to 0.
    2) Compute the deficit weight.
    3) Redistribute the deficit into other existing yacheykas for the same product:
    - first within the same `batches_id` (same party), then
    - into later parties by FIFO party order (`received_at` / party ordering),
    - and if still not enough, keep the remaining deficit only in the aggregate (negative `warehouse_stocks`).

    Conceptually, this is done by rewriting historical shipment allocations (`warehouse_transaction_stock_items`) so consumption moves from earlier batches to later ones when you edit/correct history.

    ---

    ## 2) The correction rule (Корректировка приемки)

    You said you don’t want “corrections as a separate concept”, but you still want the *effect*:
    editing Priemka later must fix the Party stock.

    There are two ways:

    - **Direct edit:** user updates the original Priemka ledger rows (preferred)
    - **Append-only correction:** user creates a correction document that references the original Priemka (audit trail)

    Either way, the stock update rule is the same:

    > Compute **delta** per product for that Priemka item:  
    > \(\Delta = \text{new_received} - \text{old_received}\)
    >
    > Then apply \(\Delta\) to the **stock layers created by that Priemka item** (same `warehouse_transaction_items_id`).

    ### 2.1 If delta is positive (Priemka increased)

    - Add \(\Delta\) to the same receiving-layer group:
    - Increase `available_weight` on existing layers for that receiving item, or
    - Create a new layer with the same `batches_id` + same receiving-item link.

    ### 2.2 If delta is negative (Priemka decreased)

    - You must “remove” \(|\Delta|\) from the layers belonging to that Priemka item.

    Cases:

    - **Case 1: that layer still has enough remaining**
    - Just decrease its `available_weight`.

    - **Case 2: part of that layer was already shipped**
    - You cannot decrease below “what is already consumed” unless you allow negatives.

If oversell is allowed, the clean representation (per your rule) is:

- set that layer’s `available_weight` to 0
- redistribute the missing amount into other yacheykas for the same product (same batch first, then next batches)
- and if still not enough, keep the remaining shortage only at aggregate level (`warehouse_stocks.available_weight < 0`)

    This keeps “history correctness”: the shipment remains, but the reduced receiving creates deficit.

    ---

    ## 3) Your example, corrected

    ### 3.1 Initial Party 1 (ledger)

    Party 1:

    - Product1: 400
    - Product2: 500
    - Product3: 1000

    ### 3.2 Priemka #1 (Fura1) — Party 1

    Ledger receiving items:

    - Product1: +100
    - Product2: +200
    - Product3: +400

    Stock layers (yacheyka) after Priemka #1:

    - Party1 / Product1: 100 (linked to Priemka#1 item for Product1)
    - Party1 / Product2: 200 (linked to Priemka#1 item for Product2)
    - Party1 / Product3: 400 (linked to Priemka#1 item for Product3)

    ### 3.3 Otgruzka — Product1: 50

    Consume FIFO from Party1/Product1 layer:

    - Party1 / Product1: 100 → 50
    - Create allocation edge for 50

    ### 3.4 Priemka #2 (Fura2) — Party 1

    Ledger receiving items:

    - Product1: +300
    - Product2: +200

    Stock layers after Priemka #2:

    - Party1 / Product1: (old layer) 50 + (new layer) 300 = total 350
    - Party1 / Product2: (old layer) 200 + (new layer) 200 = total 400
    - Party1 / Product3: 400

    ### 3.5 Correction of Priemka #1

    You want Priemka#1 to become:

    - Product1: **100 → 50**  (delta = -50)
    - Product2: **200 → 50**  (delta = -150)
    - Product3: **400 → 500** (delta = +100)

    Apply deltas to layers **linked to Priemka#1 items**:

    - Product1: Priemka#1 layer currently has 50 remaining (because 50 already shipped).
    - Apply -50 → layer becomes 0
    - Total Product1 stock becomes **300** (only Priemka#2 remains)

    - Product2: Priemka#1 layer has 200 remaining.
    - Apply -150 → layer becomes **50**
    - Total Product2 stock becomes **50 (from Priemka#1) + 200 (from Priemka#2) = 250**

    - Product3: Priemka#1 layer has 400 remaining.
    - Apply +100 → layer becomes **500**
    - Total Product3 stock becomes **500**

    ✅ Final correct yacheyka totals for Party1 after correction:

    - Product1: **300**
    - Product2: **250**
    - Product3: **500**

    Your draft outcome had Product2=150 — that would only be correct if you had an additional outflow of 100 for Product2 somewhere else.

    ---

    ## 4) Why this is the “better flow”

    - You can freely create later Priemkas and shipments.
    - You can later correct an old Priemka, and only the stock that originated from that Priemka is affected.
    - If the corrected quantity goes below what was already shipped, you don’t break history — you simply create deficit (oversell allowed).

    This is exactly the behavior you want before implementing full Option A replay.

---

## 5) Complex end-to-end example (ledger history → rebuilt stock, ends negative, then new Priemka “absorbs” deficit)

This is the complex workflow you asked for:

- Multiple batches (parties) and multiple yacheykas (layers) per product.
- A correction reduces an old Priemka and triggers redistribution.
- One product ends with **negative aggregate** (shortage) because nothing can cover it.
- Later a **new Priemka** arrives for the same product, and the incoming quantity first covers the deficit (aggregate moves toward 0). Only remaining quantity becomes available stock in a new yacheyka.

### 5.1 Setup

Warehouse: W1  
Products: P1, P2  
Parties: Party A (`batches_id=A`), Party B (`batches_id=B`), Party C (`batches_id=C`)

### 5.2 Ledger timeline (what user does)

#### T1 — Priemka A-1 (Party A, Truck 1)

- P1 +300
- P2 +200

Creates yacheykas (layers):

- A/P1 layer A1: 300
- A/P2 layer A1: 200

#### T2 — Priemka A-2 (Party A, Truck 2)

- P1 +200
- P2 +100

Creates yacheykas:

- A/P1 layer A2: 200
- A/P2 layer A2: 100

At this point:

- Party A P1 total = 500
- Party A P2 total = 300

#### T3 — Priemka B-1 (Party B, Truck 3)

- P1 +400
- P2 +100

Creates yacheykas:

- B/P1 layer B1: 400
- B/P2 layer B1: 100

#### T4 — Otgruzka S1

- P1 -600
- P2 -250

The engine consumes FIFO (older parties first). One valid allocation is:

- P1:
  - consume 300 from A/P1 A1 → 0
  - consume 200 from A/P1 A2 → 0
  - consume 100 from B/P1 B1 → 300 remaining
- P2:
  - consume 200 from A/P2 A1 → 0
  - consume 50 from A/P2 A2 → 50 remaining

Yacheykas after S1:

- A/P1: A1 0, A2 0
- B/P1: B1 300
- A/P2: A1 0, A2 50
- B/P2: B1 100

#### T5 — Otgruzka S2

- P1 -500

Consume FIFO:

- consume 300 from B/P1 B1 → 0
- remaining needed: 200, but there is no more stock for P1 in any yacheyka

So:

- all yacheykas for P1 stay at 0
- aggregate for P1 becomes **-200** (shortage)

This is the key: **no yacheyka goes negative**, only aggregate.

#### T6 — Correction for Priemka A-1 (Party A was overstated)

You discover Priemka A-1 was wrong for P1:

- A-1 P1 should have been 100 (not 300)
- delta = 100 - 300 = **-200**

But A/P1 A1 is already 0 (it was fully consumed in S1).

So the correction creates a deficit pressure of 200 that must be handled as:

1) Clamp A/P1 A1 to 0 (already 0)
2) Try to redistribute the -200 into other yacheykas for P1:
   - same batch A: other A/P1 layers? (A2 is 0) → nothing
   - next batch B: B/P1 layers? (B1 is 0) → nothing
3) Nothing can cover it → aggregate shortage increases:
   - P1 aggregate becomes **-400**

Again: **no yacheyka negative**, only aggregate.

### 5.3 Stock view at this point (what “build from ledger” means)

If you now “rebuild stock from ledger” for P1:

- The engine replays T1..T6 in order
- It creates receiving layers for A, then B
- It consumes them for S1 and S2
- It applies the correction delta by rewriting allocations
- The resulting yacheykas for P1 are all 0, and the aggregate is **-400**

This is your required behavior: history edits change allocations, but yacheykas never go negative.

### 5.4 New Priemka arrives later and absorbs deficit first

#### T7 — Priemka C-1 (Party C, Truck 4)

- P1 +500

Creates a new yacheyka:

- C/P1 layer C1: 500

Now the rule you requested:

> If the product has aggregate deficit (negative `warehouse_stocks.available_weight`), the next Priemka must first cover that deficit.

So, after replay:

- P1 aggregate shortage was -400
- Incoming +500 first covers 400:
  - aggregate becomes 0
  - remaining +100 becomes real available stock

Final after T7:

- C/P1 layer C1 becomes **100 available**
- P1 aggregate becomes **0**

If C-1 had been only +300 instead:

- aggregate would become -100
- yacheykas remain non-negative

This “absorb deficit first” behavior is what you described as:
“new priemka item yacheyka for the same product should get its deficit”.
