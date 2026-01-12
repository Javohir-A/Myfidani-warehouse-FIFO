# FIFO v2 — Option A (Replay Engine) as Table-Level Actions (No Code / No SQL)

This document explains **Option A (ledger + deterministic replay)** using **only table-level actions** (create/update/delete). It is meant to be an “operational spec” you can validate against your UI and Excel v2 behavior.

**Key goal**: when you edit an old document (receiving/shipment/return/inventory), the system updates **downstream** allocations and remaining stock exactly like `Логика FIFO_v2.xlsx`.

---

## 1) Two layers of truth (important)

### 1.1 Source-of-truth tables (events)

These tables are the **ledger**. They represent “what happened” and are the only inputs the replay engine needs:

- `warehouse_transactions` (header: type, status, document_date, warehouse_id, batch_transport_units_id, …)
- `warehouse_transaction_items` (lines: products_id, batches_id, weight/count, …)

### 1.2 Derived tables (computed results)

These tables are **outputs** of the replay engine. They can be deleted and rebuilt from the ledger:

- `warehouse_transaction_stock_items` (allocation links: which stock layer was consumed/adjusted by which transaction item)
- `warehouse_stock_items` (FIFO layers: remaining `available_weight` etc.)
- `warehouse_stocks` (aggregated totals, computed from `warehouse_stock_items`)

**Rule**: if history changes, derived tables must be rebuilt so the result matches replay (Excel behavior).

---

## 2) Definitions (domain mapping)

- **Party (Партия)** = procurement batch = **`batches_id`**
- **FIFO layer** = a row in `warehouse_stock_items` with:
  - `batches_id` (party)
  - `received_at` (layer ordering)
  - `available_weight` / counts (current remaining)
- **Operation date ordering** = `warehouse_transactions.document_date` (with a stable tie-break like `created_at`)

---

## 3) The replay engine “unit of work” (scope)

Start with a simple scope:

- **Scope = (`warehouse_id`, `products_id`)**

Later, you can extend scope rules (e.g., include `warehouse_cells_id`, or global product scope with transfer distribution).

Whenever you create/update/delete any relevant event in that scope, you run a recompute for that scope.

---

## 4) Step-by-step table actions by operation

### 4.1 Receiving (Приемка) — create

**User intent**: accept goods from a truck into warehouse, with per-item `batches_id` (especially for mixed trucks).

#### Table actions (Receiving)

1. **Create** `warehouse_transactions`
   - `type = ["receiving"]`
   - `warehouse_id`
   - `batch_transport_units_id`
   - `document_date`
   - other header fields (partners/branches/etc.)

2. **Create** `warehouse_transaction_items` (one per product line)
   - `warehouse_transactions_id = <receiving>`
   - `products_id`
   - `batches_id` (**Party id** from procurement content)
   - actual received weight/count fields

3. **Trigger recompute** for scope (`warehouse_id`, `products_id`) for each affected product.
   - Recompute will rebuild stock layers and derived allocations (see section 5).

#### Result expectation (Receiving)

- Stock increases in the correct party (`batches_id`), and later shipments may redistribute across parties if history is edited (Excel behavior).

---

### 4.2 Shipment (Отгрузка) — create

**User intent**: ship goods out (issue). This consumes FIFO layers across parties.

#### Table actions (Shipment)

1. **Create** `warehouse_transactions`
   - `type = ["shipment"]` (or `issue` depending on your naming)
   - `warehouse_id`
   - `document_date`
   - link to order if applicable

2. **Create** `warehouse_transaction_items` (one per shipped product line)
   - `warehouse_transactions_id = <shipment>`
   - `products_id`
   - shipped weight/count fields
   - (optional) selection method if you support “by truck”

3. **Trigger recompute** for each affected product scope (`warehouse_id`, `products_id`)

#### Result expectation (Shipment)

- Shipment consumes from the earliest party (by party order) and earliest layers (`received_at`) within that party.
- Derived allocation links are created in `warehouse_transaction_stock_items`.

---

### 4.3 Return (Возврат) — create (attached to a shipment)

**User intent**: return goods back to warehouse, attached to a specific shipment.

#### Table actions (Return)

1. **Create** `warehouse_transactions`
   - `type = ["return"]`
   - `warehouse_id`
   - `document_date`
   - reference to the original shipment (store shipment id in a header field or comment/relationship field)

2. **Create** `warehouse_transaction_items`
   - `warehouse_transactions_id = <return>`
   - `products_id`
   - returned weight/count

3. **Trigger recompute** for affected scopes.

#### Result expectation (Return)

- Return should restore stock in a stable way.
- Best rule: return “undoes” the same FIFO layers the shipment consumed (so it remains correct even if FIFO changes after edits).

That requires allocations to exist (or be rebuilt) in `warehouse_transaction_stock_items`.

---

### 4.4 Inventory (Инвентаризация) — create

**User intent**: reconcile system vs physical stock at a time.

#### Table actions (Inventory)

1. **Create** `warehouse_transactions`
   - `type = ["adjustment"]` (inventory)
   - `warehouse_id`
   - `document_date`
   - `inventory_number`
   - `employees_id`, `comment`

2. **Create** `warehouse_transaction_items` (one per product row in the UI)
   - `warehouse_transactions_id = <inventory>`
   - `products_id`
   - store “actual” values (fact)
   - store “system” values (snapshot at creation time) and/or store `diff`

3. **Trigger recompute** for each affected scope.

#### Result expectation (Inventory, FIFO v2 rule)

- At the inventory date, after applying all earlier events, the engine finds the **active layer** where the remaining stock currently sits.
- The inventory `diff = actual - system` is applied to that active layer.
- If earlier events are edited later, the “active layer” may change — and the inventory effect will move accordingly after replay (Excel behavior).

---

## 5) What “recompute” does (table actions only)

For a given scope (`warehouse_id`, `products_id`) and a recompute start date `T0` (usually the earliest changed event date):

### 5.1 Freeze the inputs (ledger snapshot)

1. Read all relevant `warehouse_transactions` + `warehouse_transaction_items` for that scope from the beginning (or from a stored snapshot) ordered by date.

No mutations yet—just collect the event list.

### 5.2 Delete derived outputs in the affected window

1. **Delete** (or hard-delete) derived allocation rows for events in the recompute window:
   - remove `warehouse_transaction_stock_items` links for affected transactions/items in the scope from date `T0` onward.

2. Reset derived stock state for the scope:
   - Either:
     - **Rebuild `warehouse_stock_items` completely** from scratch from the beginning, or
     - Use snapshots and only rebuild from `T0` forward.

For a first implementation, rebuilding fully for a single product scope is simplest and matches Excel.

### 5.3 Replay events and re-create derived rows

1. For each event in chronological order:
   - **Receiving**: ensure a FIFO layer exists in `warehouse_stock_items` for that party (`batches_id`) and time (`received_at`), and increase remaining.
   - **Shipment**: consume from FIFO layers and **create** allocation rows in `warehouse_transaction_stock_items`.
   - **Return**: restore based on the shipment’s allocation split (create allocation rows for the return too if you track it).
   - **Inventory**: compute diff and apply to the active layer; record link(s) in `warehouse_transaction_stock_items`.

2. After replay finishes, **Update** `warehouse_stocks` totals by recomputing from `warehouse_stock_items`.

**Outcome**: derived tables now match the ledger history—so changing an old row changes later results, like Excel.

---

## 6) Editing scenarios (what happens in tables)

### 6.1 Edit an old receiving (change weight or batches)

#### Table actions (Edit receiving)

1. **Update** the existing `warehouse_transactions` (if header changes: `document_date`, etc.)
2. **Update** the relevant `warehouse_transaction_items` (weight/count changes, and possibly `batches_id` changes)
3. **Trigger recompute** starting from this receiving’s `document_date`
4. Recompute deletes and rebuilds allocations/stock downstream (section 5)

#### Expected behavior (Edit receiving)

- Some later shipments may shift consumption from Party A to Party B (or vice versa).
- Therefore, later `warehouse_transaction_stock_items` allocations change, and `warehouse_stock_items.available_weight` changes too.

This is exactly the Excel cascade.

---

### 6.2 Edit an old shipment (change shipped weight)

Same structure:

1. **Update** shipment header/item rows in the ledger tables
2. **Trigger recompute** from that shipment date
3. Downstream allocations and остатки change (including which parties are exhausted earlier)

---

### 6.3 Delete/cancel an old event

1. **Delete** (or soft-delete) the ledger row(s) in `warehouse_transactions`/`warehouse_transaction_items`
2. **Trigger recompute** from that date
3. Derived allocations and stock rebuild to match the new history

---

## 7) Minimal “correctness constraints”

To make replay deterministic and match FIFO v2:

- Every receiving line must carry the correct `batches_id` (Party).
- `warehouse_stock_items` layers must retain their `batches_id` (do not merge different batches into one layer).
- Returns should be attached to shipments to undo the correct layers.
- Inventory must apply to the active layer at its date (FIFO v2 rule).

---

## 8) What this gives you (why it matches Excel)

- Excel recalculates all downstream party usage whenever an earlier input changes.
- Replay engine does the same: ledger is the input, derived allocations/stock are recomputed outputs.

So **“I change one thing and other things change too”** becomes an expected, safe feature—not a bug.
