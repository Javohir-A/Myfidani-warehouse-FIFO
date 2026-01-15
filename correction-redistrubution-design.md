# Correction Redistribution Design

## Overview

When a priemka (receiving) transaction is corrected and causes stock items to go negative, we need to redistribute shipment transaction items that consumed from those negative stock items to available stock items.

## Business Logic

### Scenario Example

1. **Initial State:**
   - Priemka: 1000 kg → `stock_item_A` created with 1000 kg
   - 5 shipments, each with transaction items consuming 100 kg from `stock_item_A`
   - Total consumed: 500 kg (5 items × 100 kg)
   - `stock_item_A` remaining: 500 kg

2. **After Correction:**
   - User corrects priemka: should be 200 kg (not 1000 kg)
   - Correction transaction created
   - `stock_item_A`: 200 - 500 = **-300 kg deficit**
   - 5 shipment transaction items still linked to `stock_item_A` (each showing 100 kg consumption)

3. **Redistribution Goal:**
   - Make `stock_item_A` go from -300 kg to **0 kg** (not negative)
   - Redistribute the -300 kg deficit to available stock items

4. **Redistribution Process (found `stock_item_B` with 500 kg):**
   - **Items 1-3:** Each had 100 kg from `stock_item_A`
     - Reduce consumption from `stock_item_A` to 0 (adds back 300 kg to `stock_item_A`)
     - `stock_item_A`: -300 + 300 = **0 kg** ✓
     - Create NEW transaction items in shipments 1-3 linked to `stock_item_B` with 100 kg each
     - New items get batch/proforma from `stock_item_B`
   - **Items 4-5:** Each had 100 kg from `stock_item_A`
     - Keep as is (still linked to `stock_item_A`)
     - They already consumed their 100 kg each (ledger records stay the same)
     - `stock_item_A` is now at 0 kg, so no further action needed

5. **Final State:**
   - `stock_item_A`: 0 kg (deficit covered)
   - Items 1-3: Original items at 0 kg, new items linked to `stock_item_B`
   - Items 4-5: Still linked to `stock_item_A` (ledger unchanged)
   - `stock_item_B`: 500 - 300 = 200 kg remaining

## Data Flow

### Input
- `stockItemGUID`: The stock item that went negative
- `deficitAmount`: The amount of deficit (negative value, e.g., -300 kg)
- `productsID`: Product ID for finding alternative stock
- `warehouseID`: Warehouse ID for finding alternative stock

### Process
1. Find all `warehouse_transaction_stock_items` linked to the negative stock item
2. Group by shipment transaction and transaction item
3. Find available stock items (FIFO) for the same product
4. Redistribute consumption to cover the deficit
5. Create new transaction items in shipments
6. Update links and transaction items

### Output
- Redistributed shipment items
- New transaction items created
- Updated links (old ones soft deleted)
- Stock items updated

## Function Design

### Main Function: `redistributeShipmentLinksForNegativeStock`

```go
func redistributeShipmentLinksForNegativeStock(
    request *pkg.FunctionRequest,
    stockItemGUID string,
    deficitWeight float64,  // Negative value (e.g., -300)
    deficitCount float64,
    productsID string,
    warehouseID string,
    packagingID string,
) error
```

**Returns:** Error if redistribution fails

**Process:**
1. Query `warehouse_transaction_stock_items` where `warehouse_stock_items_id = stockItemGUID` and `deleted_at IS NULL`
2. Group links by `warehouse_transaction_items_id` (each shipment item)
3. For each shipment item:
   - Get current consumption amounts (`consumed_weight`, `consumed_quantity`)
   - Get shipment transaction item details
   - Calculate how much to redistribute (up to deficit amount)
4. Find available stock items (FIFO) for same product
5. Redistribute consumption:
   - Soft delete old links (set `deleted_at`)
   - Create new links to available stock items
   - Update transaction items
   - Create new transaction items if needed

## Detailed Algorithm

### Step 1: Find Shipment Links

```sql
SELECT 
    wtsi.guid,
    wtsi.warehouse_transactions_id,
    wtsi.warehouse_transaction_items_id,
    wtsi.warehouse_stock_items_id,
    wtsi.consumed_weight,
    wtsi.consumed_quantity,
    wti.products_id,
    wti.weight_warehouse,
    wti.count_warehouse,
    wt.type,
    wt.warehouse_id
FROM warehouse_transaction_stock_items wtsi
JOIN warehouse_transaction_items wti ON wtsi.warehouse_transaction_items_id = wti.guid
JOIN warehouse_transactions wt ON wtsi.warehouse_transactions_id = wt.guid
WHERE wtsi.warehouse_stock_items_id = '<stock_item_guid>'
  AND wtsi.deleted_at IS NULL
  AND wt.type @> ARRAY['shipment']::text[]
ORDER BY wt.document_date ASC, wti.created_at ASC
```

### Step 2: Group by Transaction Item

Group links by `warehouse_transaction_items_id` to get total consumption per shipment item.

### Step 3: Calculate Redistribution Amount

For each shipment item:
- Current consumption: `consumed_weight` from links
- Redistribution needed: `min(consumed_weight, abs(deficitWeight))`
- Remaining deficit: `deficitWeight + redistributed_amount`

### Step 4: Find Available Stock Items

```sql
SELECT 
    guid,
    available_weight,
    weight,
    count,
    batches_id,
    proformas_id,
    received_at
FROM warehouse_stock_items
WHERE warehouse_stocks_id = (
    SELECT guid FROM warehouse_stocks 
    WHERE warehouse_id = '<warehouse_id>' 
      AND products_id = '<products_id>'
)
  AND products_id = '<products_id>'
  AND is_consumed = false
  AND deleted_at IS NULL
  AND available_weight > 0
  AND guid != '<stock_item_guid>'  -- Exclude the negative one
ORDER BY received_at ASC  -- FIFO
```

### Step 5: Redistribute Consumption

For each shipment item to redistribute:
1. **Soft delete old links:**
   ```go
   updatePayload := map[string]any{
       "guid": linkGUID,
       "deleted_at": time.Now(),
   }
   ```

2. **Update original transaction item:**
   ```go
   updatePayload := map[string]any{
       "guid": transactionItemGUID,
       "weight_warehouse": 0,  // Or remove if 0
       "count_warehouse": 0,
   }
   ```

3. **Create new transaction item in shipment:**
   ```go
   newItemPayload := map[string]any{
       "warehouse_transactions_id": shipmentTransactionGUID,
       "products_id": productsID,
       "warehouse_id": warehouseID,
       "weight_warehouse": redistributedWeight,
       "count_warehouse": redistributedCount,
       "batches_id": newStockItemBatchesID,  // From new stock item
       "proformas_id": newStockItemProformasID,  // From new stock item
       // Copy other fields from original item
   }
   ```

4. **Create new link to available stock item:**
   ```go
   newLinkPayload := map[string]any{
       "warehouse_transactions_id": shipmentTransactionGUID,
       "warehouse_transaction_items_id": newTransactionItemGUID,
       "warehouse_stock_items_id": availableStockItemGUID,
       "consumed_weight": redistributedWeight,
       "consumed_quantity": redistributedCount,
   }
   ```

5. **Update available stock item:**
   ```go
   updatePayload := map[string]any{
       "guid": availableStockItemGUID,
       "weight": currentWeight - redistributedWeight,
       "available_weight": currentAvailableWeight - redistributedWeight,
       "count": currentCount - redistributedCount,
   }
   ```

### Step 6: Update Stock Item (Remove Deficit)

After redistribution, update the original stock item to remove the redistributed amount:
```go
updatePayload := map[string]any{
    "guid": stockItemGUID,
    "weight": currentWeight + redistributedAmount,  // Add back the redistributed amount
    "available_weight": currentAvailableWeight + redistributedAmount,
}
```

## Edge Cases

### 1. Multiple Products in Same Shipment
- Handle each product separately
- Redistribute only the product that went negative
- Other products in shipment remain unchanged

### 2. Insufficient Available Stock
- Redistribute as much as possible
- Remaining deficit stays in original stock item (negative)
- Will be automatically corrected when new stock arrives

### 3. Multiple Stock Items for Same Product
- Use FIFO (oldest first)
- May need to consume from multiple stock items
- Create multiple links if needed

### 4. Shipment Items Already at 0
- Skip redistribution for items already at 0
- Only redistribute items with positive consumption

### 5. Transaction Item Updates
- If `weight_warehouse` becomes 0, consider soft deleting the transaction item
- Or keep it with 0 weight for audit trail

## Integration Points

### In `updateStockForCorrection` Function

After updating stock items (around line 918 in `create_correction_transaction.go`):

```go
// After updating stock item
if newAvailableWeight < 0 {
    // Stock item went negative, redistribute shipment links
    err = redistributeShipmentLinksForNegativeStock(
        request,
        stockItemGUID,
        newAvailableWeight,  // Negative value
        newCount,  // If also negative
        productsID,
        warehouseID,
        packagingID,
    )
    if err != nil {
        request.Logger.Err(err).Msg("Failed to redistribute shipment links")
        // Log but don't fail - stock was already updated
    }
}
```

## Data Structures

### ShipmentLinkInfo
```go
type ShipmentLinkInfo struct {
    LinkGUID                string
    TransactionGUID          string
    TransactionItemGUID      string
    StockItemGUID            string
    ConsumedWeight           float64
    ConsumedQuantity         float64
    TransactionItemData     map[string]any  // Full transaction item data
    TransactionData         map[string]any  // Full transaction data
}
```

### RedistributionResult
```go
type RedistributionResult struct {
    RedistributedWeight      float64
    RedistributedCount       float64
    NewTransactionItems      []string  // GUIDs of new transaction items created
    UpdatedLinks             []string  // GUIDs of links soft deleted
    RemainingDeficit         float64   // If not fully covered
}
```

## Transaction Totals Update

After redistribution, if deficit is fully covered:
- Recalculate shipment transaction totals
- Update `total_weight_warehouse`, `total_count_warehouse` in shipment transactions

## Logging

Log important steps:
- Found shipment links count
- Redistribution amounts
- New transaction items created
- Links soft deleted
- Remaining deficit (if any)

## Testing Scenarios

1. **Single shipment, single item:** One shipment with one item consuming from negative stock
2. **Multiple shipments, multiple items:** 5 shipments, each with items consuming from negative stock
3. **Insufficient available stock:** Not enough stock to cover full deficit
4. **Multiple products:** Shipment has multiple products, only one goes negative
5. **Multiple stock items:** Need to consume from multiple available stock items
6. **Already at 0:** Some shipment items already at 0 consumption

## Future: Automatic Rebalancing

When new stock arrives (priemka/transfer):
- Check if any shipment items are linked to negative stock items
- Automatically redistribute to new stock items
- This will be handled in `ProcessIncome` function
