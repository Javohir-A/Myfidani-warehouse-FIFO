# FIFO v2 — Option A (Ledger + Deterministic Replay) with Examples

This doc is a practical walkthrough of **Option A**: to get “Excel behavior” (edit an old event → later results change), you treat stock as a **derived** result of a time-ordered **ledger**, and you **replay** that ledger deterministically.

It is written to match the mental model of `Логика FIFO_v2.xlsx`:

- Parties (Партии) are the capacity layers.
- Operations are rows ordered by date.
- Changing one row changes downstream allocations because the whole timeline is recomputed.

In your domain mapping:

- **Party = `batches_id`**
- FIFO layers are `warehouse_stock_items` ordered by `received_at`
- Allocations are stored in `warehouse_transaction_stock_items`

---

## 0) What “recompute engine” means (in one paragraph)

Instead of permanently trusting the current `warehouse_stock_items.available_weight` as the only truth, you treat:

1) **Events** (`warehouse_transactions` + `warehouse_transaction_items`) as the truth, and
2) **Stock + allocations** (`warehouse_stock_items`, `warehouse_transaction_stock_items`, `warehouse_stocks`) as **derived**.

When an old event changes (edit receiving/shipment/return/inventory), you rebuild the derived state by replaying the ordered events.

---

## 1) The data you need for replay (minimum)

For a chosen scope (start simple: **one product + one warehouse**):

- **Receiving events**: add stock into a party (`batches_id`) at a time (`document_date`)
- **Shipment events**: consume stock by FIFO across parties/layers
- **Return events**: undo shipment consumption (preferably attached to a shipment)
- **Inventory events**: apply delta to the “active” party/layer at that time (FIFO v2 rule)
- **Transfer events**: changes warehouse distribution (can be added later)

---

## 2) Replay rules (the deterministic algorithm)

Below is the simplest deterministic set of rules that behaves like the Excel:

### 2.1 Party ordering

Because **Party = `batches_id`**, you still need a party sequence:

- Party order = earliest receiving date for that `batches_id` (or `batches.created_at` if you trust it).

### 2.2 FIFO layers within a party

Each receiving creates a layer:

- Layer key = (`batches_id`, `received_at`)
- Within a party, consume layers ordered by `received_at ASC`.

### 2.3 Consumption

Shipment consumes in this priority:

1) Party 1 layers (oldest batch)
2) Party 2 layers
3) …

And within each party:

- oldest layer (`received_at ASC`) first

### 2.4 Return (attached)

Return should undo **the exact layers** that the shipment consumed (stable behavior).

This is why `warehouse_transaction_stock_items` is important: it records the layer split.

### 2.5 Inventory (FIFO v2 rule)

Inventory delta is applied to the “active” party/layer at that time:

- Replay up to inventory date.
- Find where the current remaining stock sits (the first layer that still has remaining weight).
- Apply `diff = actual - system` to that layer.

If earlier history changes, the “active layer” at inventory date may change too — and inventory will follow it after replay (Excel cascade).

---

## 3) Example A — Receiving → Shipment (basic remaining/остаток)

### Parties (batches)

- Batch A (`batches_id=A`) receiving: +1000 kg at **2026-01-01**
- Batch B (`batches_id=B`) receiving: +500 kg at **2026-01-02**

### Events (time-ordered)

1) receiving A +1000
2) receiving B +500
3) shipment S1 -1200 at **2026-01-03**

### Replay result (FIFO)

Ship 1200:

- Consume from Batch A: 1000 (A exhausted)
- Consume from Batch B: 200

Remaining (остаток):

- Batch A: 0
- Batch B: 300

### What the engine would persist (derived)

- `warehouse_transaction_stock_items` for shipment S1:
  - link(S1_item → stock_layer(A@2026-01-01), consumed_weight=1000)
  - link(S1_item → stock_layer(B@2026-01-02), consumed_weight=200)
- `warehouse_stock_items.available_weight` updated accordingly
- `warehouse_stocks` totals recomputed from stock items

---

## 4) Example B — Edit old receiving (why later shipments change)

Continue Example A, but you **edit** receiving A:

- Batch A receiving changes from **1000 → 800**

### Events after edit

1) receiving A +800
2) receiving B +500
3) shipment S1 -1200

### Replay again

Ship 1200:

- Consume from A: 800
- Consume from B: 400

Remaining:

- A: 0
- B: 100

### What changed (this is the Excel effect)

Shipment allocation changed:

- Before: A1000 + B200
- After:  A800 + B400

Therefore, `warehouse_transaction_stock_items` links and `warehouse_stock_items.available_weight` must change downstream.

---

## 5) Example C — Edit receiving by increasing it

Start from Example A but edit receiving A:

- Batch A receiving changes **1000 → 1300**

Replay:

- Shipment -1200 consumes entirely from A.

Remaining:

- A: 100
- B: 500

Downstream allocations shift “back” into earlier parties.

---

## 6) Example D — Backdated receiving between shipments (classic cascade)

### Parties

- Batch A receiving +1000 at **Jan-01**
- Batch B receiving +500 at **Jan-05**

### Events

- Shipment S1: -900 at **Jan-03**
- Shipment S2: -400 at **Jan-06**

### Replay (before the change)

- After S1: A remaining 100
- S2 consumes: A100 + B300 → B remaining 200

### Now insert/edit a backdated receiving

- Add receiving for Batch A: +300 at **Jan-04**

### Replay (after the change)

- After S1: A remaining 100
- After Jan-04 receiving: A remaining 400
- S2 consumes: A400 + B0 → B remaining 500

This is exactly the “I changed one thing and later things changed” behavior from Excel.

---

## 7) Example E — Return attached to shipment (stable undo)

Take Example A where shipment S1 consumed:

- A: 1000
- B: 200

Now create Return R1 attached to S1: **+150 kg** at **Jan-04**

### Replay rule

Return reverses the same split the shipment made (oldest consumed first):

- Restore into A first (because shipment consumed A first):
  - A +150

After replay:

- A: 150
- B: 300

**Why this matters**: if you just “apply FIFO” for returns without using the original split, returns could go back to the wrong party after history edits.

---

## 8) Example F — Inventory (active party rule from FIFO v2)

Assume after replay up to **Jan-10**, the current remaining stock sits in Batch B (A exhausted).

Inventory event at **Jan-10** says:

- System says: 300 kg
- Actual says: 250 kg
- Diff = -50 kg

### Replay rule (FIFO v2)

Apply -50 to the **active layer** (where remaining stock sits):

- subtract 50 from Batch B active layer

If you later edit an old shipment and at Jan-10 the remaining stock would have been in Batch A instead, the replay engine will move the inventory delta to Batch A — that is Excel behavior.

---

## 9) Example G — Mixed truck receiving (multiple `batches_id` in one Priemka)

One receiving document can contain items from different batches (mixed truck).

Receiving TX at Jan-01:

- Product X, `batches_id=A`, +600 kg
- Product X, `batches_id=B`, +400 kg

In replay this simply means:

- Party A capacity increases by 600
- Party B capacity increases by 400

Then shipments consume Party A first (if A is earlier in party order), then Party B.

---

## 10) Implementation notes for your current tables (practical)

### 10.1 What you already have

- `warehouse_stock_items` includes `batches_id` and `received_at`
- shipment consumes FIFO from `warehouse_stock_items`
- `warehouse_transaction_stock_items` links exist (good for replay + undo)

### 10.2 What you must be careful about

- Avoid merging layers in a way that destroys party traceability (for FIFO v2, layers must retain `batches_id`).
- If you allow historical edits, you must be able to:
  - delete/rebuild `warehouse_transaction_stock_items` allocations after a date, and
  - set `warehouse_stock_items` state to match replay.

---

## 11) Next step (if you want to implement Option A)

Confirm these two design choices and the algorithm becomes fully specifiable:

1) Party ordering: should party order be based on `batches.created_at` or first warehouse receiving date for that batch?
2) Scope: replay per `(warehouse_id, products_id)` or global per product with transfers affecting warehouse distribution?

---

## 12) Table-level step-by-step (no code, no SQL)

If you want the same explanation but written as **exact table actions** (create/update/delete) without any code/SQL details, read:

- `docs/FIFO_V2_REPLAY_ENGINE_TABLE_ACTIONS.md`
