# Complex example — multiple Priemkas + Otgruzkas + correction causing deficit, then deficit is covered by another Party

This example matches your requirements:

- You do multiple Priemkas and Otgruzkas.
- Later you discover Priemka was wrong and apply a correction.
- **Parties do not go negative** (party layers stay at **0** minimum).
- Oversell is allowed → the system can show a **negative yacheyka (deficit)**.
- If later there is available stock for the same product from another Party, we **move the deficit** onto that stock (i.e., we cover the deficit).
- If there is not enough available stock, remaining deficit stays negative.

This describes **business behavior using your tables**, not code.

---

## 0) Tables we use (conceptually)

### Ledger (truth)

- `warehouse_transactions`
- `warehouse_transaction_items`

### Derived stock + allocations (persisted for speed, rebuildable by engine)

- `warehouse_stock_items` (yacheyka layers; each layer belongs to a Party via `batches_id`)
- `warehouse_transaction_stock_items` (edges: shipment item → which layers were consumed)
- `warehouse_stocks` (aggregates)

### One additional concept: deficit layer

To allow oversell while keeping parties non-negative, we represent oversell as a special layer:

- **Deficit yacheyka**: a `warehouse_stock_items` row with:
  - `batches_id = NULL` (or a dedicated “DEFICIT” batch)
  - `available_weight` can be **negative**

This is the “нет партии / deficit” bucket.

---

## 1) Setup (single warehouse, single product)

Warehouse: **W1**  
Product: **P1 (Product1)**  

We have two procurement parties (batches):

- **Party A** = `batches_id = A`
- **Party B** = `batches_id = B`

Both parties arrive in **two trucks** each (so you can see “same party, several trucks”).

---

## 2) Timeline of events (ledger)

### Event 1 — Priemka A-1 (Party A, Truck 1)

- Receiving item: **+300 kg** (Party A)

Stock layers created:

- Layer A1: (Party A) `available=300` (linked to receiving item A-1)

### Event 2 — Priemka A-2 (Party A, Truck 2)

- Receiving item: **+400 kg** (Party A)

Stock layers created:

- Layer A2: (Party A) `available=400` (linked to receiving item A-2)

✅ After Event 2, Party A total available = **700**

### Event 3 — Otgruzka S1 (shipment)

- Shipment item: **-200 kg**

FIFO consumption (oldest first):

- Consume 200 from Layer A1: A1 becomes `300 → 100`

Edges created:

- S1 → A1 consumed 200

### Event 4 — Otgruzka S2

- Shipment item: **-250 kg**

FIFO consumption:

- Consume 100 from A1: A1 `100 → 0`
- Consume 150 from A2: A2 `400 → 250`

Edges:

- S2 → A1 consumed 100
- S2 → A2 consumed 150

### Event 5 — Otgruzka S3

- Shipment item: **-180 kg**

FIFO consumption:

- Consume 180 from A2: A2 `250 → 70`

Edges:

- S3 → A2 consumed 180

✅ After Event 5 (before any correction):

- Party A remaining = A1 0 + A2 70 = **70**
- Total shipped so far = 200 + 250 + 180 = **630**
- Total received for Party A (as entered) = **700**

---

### Event 6 — Priemka B-1 (Party B, Truck 3)

- Receiving item: **+150 kg** (Party B)

Stock layer:

- Layer B1: (Party B) `available=150`

### Event 7 — Priemka B-2 (Party B, Truck 4)

- Receiving item: **+200 kg** (Party B)

Stock layer:

- Layer B2: (Party B) `available=200`

✅ After Event 7, Party B total available = **350**

---

### Event 8 — Otgruzka S4

- Shipment item: **-300 kg**

FIFO rule across parties: **Party ordering + received_at ordering**

At this moment we still have Party A remaining 70, then Party B.

Consumption:

- Consume 70 from A2: A2 `70 → 0`
- Consume 150 from B1: B1 `150 → 0`
- Consume 80 from B2: B2 `200 → 120`

Edges:

- S4 → A2 consumed 70
- S4 → B1 consumed 150
- S4 → B2 consumed 80

✅ After Event 8:

- Party A remaining = **0**
- Party B remaining = **120**

So far everything is consistent.

---

## 3) Now the critical part: correction of Party A priemka

### Business reality discovered later

You realize Party A was wrong:

- Real Party A received should have been **500 kg**, not 700 kg.

So correction delta is:

- \(\Delta_A = 500 - 700 = -200\)

### What must happen to keep parties correct

We apply this correction to the **history**, and the engine recomputes.

Key constraints you requested:

- Party layers must not go negative.
- If we don’t have enough party stock after correction, we allow deficit.
- If another party has available stock, the deficit should move to it.

### Replay result (high level)

Before correction, Party A total shipped consumption across S1..S4 was:

- S1: 200 (A)
- S2: 250 (A)
- S3: 180 (A)
- S4: 70  (A)
- Total A consumption = **700**

But after correction Party A received total becomes **500**.

So we have **200 kg too much** previously “covered” by Party A.

That 200 kg must be handled in this order:

1) Party A layers stay at 0 (cannot go negative).
2) Create a deficit of 200 (negative yacheyka).
3) If there is other party stock available (Party B has 120 remaining), cover as much deficit as possible by taking from that stock.

### Step 3.1 — Create deficit layer

Create or update the deficit layer for (W1, Product1):

- Deficit layer D: `available = -200`

### Step 3.2 — Cover deficit from Party B available (deficit “moves”)

Party B has remaining 120 (B2).

Cover deficit by consuming from B2 (conceptually moving the deficit onto Party B stock):

- B2 `available 120 → 0`
- Deficit D `-200 → -80`

How this is represented in derived tables:

- The engine rewrites allocations (`warehouse_transaction_stock_items`) for historical shipments so that
  the last 120 kg that previously belonged to Party A is now allocated to Party B layers instead.
  (This is the “movement of deficit” you described.)

✅ Final result after correction replay:

- Party A remaining = **0**
- Party B remaining = **0**
- Deficit layer remaining = **-80**

Meaning: you are oversold by 80 kg, and there is no other party stock to cover it.

---

## 4) The “another party exists later” case (deficit becomes 0)

If after the correction you later receive another party (Party C) with 500 kg:

- Party C available = 500
- Deficit D = -80

Then replay will net it:

- Party C becomes 420
- Deficit becomes 0

Parties still never go negative.

---

## 5) Why this matches your requirement “party should be 0”

When you correct a receiving downwards, you **do not** want to make Party A negative.

Instead:

- Party A is fully exhausted → **0**
- The system acknowledges that shipments happened anyway → deficit (negative yacheyka)
- If other parties have stock, deficit is covered by them (and deficit shrinks)
- Remaining deficit stays if nothing can cover it

This is the safest and most explainable model for “oversell allowed” while keeping party logic stable.

