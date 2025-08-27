# automatic-carnival
Shopify PreProduct x various customizations

# Pre-Order Automation System

This document describes the technical implementation of our pre-order batching and automation system using **Shopify Flow** + **PreProduct**. It details the metafields in use, all Flows, and required manual actions.

---

## Metafields

### Shop-Level (Global Controls)

* `preorder.current_batch_number` → Integer (active batch ID, e.g. `6`)
* `preorder.current_ship_month` → Single line text (e.g. `February 2026`)
* `preorder.current_ship_message` → Single line text (e.g. `Ships by February 2026`)
* `preorder.current_lead_days` → Integer (e.g. `100`)
* `preorder.next_ship_month` → Single line text (e.g. `March 2026`)
* `preorder.next_ship_message` → Single line text (e.g. `Ships by March 2026`)
* `preorder.pause_rollover` → Boolean (true = stop monthly rollover Flow)

### Product-Level

* `preorder.batch_number` → Integer (copied from shop each rollover)
* `preorder.ship_month` → Single line text
* `preorder.batch_ship_message` → Single line text
* `preorder.lead_days` → Integer
* `flow.auto_pre-order` → Boolean (inclusion toggle; already in use)

### Order-Level (Snapshotted on Creation)

* `preorder.batch_number` → Integer
* `preorder.ship_month` → Single line text

### Variant-Level (To Be Added)

* `preorder.exclude_from_batch` → Boolean (skip this variant in rollover)
* `preorder.lead_days_override` → Integer (per-variant lead time)
* `preorder.batch_ship_message_override` → Single line text
* `preorder.variant_auto_preorder` → Boolean (per-variant opt-in)

---

## Flows

### Flow #1: Monthly Batch Rollover

* **Trigger:** Daily @ 9:00 AM PT
* **Condition 1:** Only run if `{{ now | date: "%d" }} == "01"`
* **Condition 2:** If `shop.metafields.preorder.pause_rollover == true`, post Slack message and stop.

**Else (proceed with rollover):**

1. Increment `current_batch_number` (+1).
2. Copy `next_ship_month` → `current_ship_month` and `next_ship_message` → `current_ship_message`.
3. Slack: STOP notice.
4. Loop: For each product in Auto Pre-Order collection (ID `326055919768`)

   * Update product metafields from shop-level values.
   * Nested: For each variant

     * If `exclude_from_batch == true`, skip.
     * PreProduct → Take this variant off pre-order.
     * PreProduct → Create listing for this variant with:

       * Lead time = `variant.lead_days_override || product.lead_days`
       * Ship message = `variant.batch_ship_message_override || product.batch_ship_message`
   * Add product tag: `PreOrder-B{batch_number}`.
5. Slack: START confirmation.

### Flow #2: Prep Next Batch Info

* **Trigger:** 25th of each month @ 9:00 AM PT.
* **Action:** Posts Slack preview showing current batch, next batch number, next ship month/message, and reminder that setting `pause_rollover = true` will stop the rollover.
* (Optional: auto-fill `next_ship_month/message` if offset is always fixed).

### Flow #3: Order Created Snapshot

* **Trigger:** Order created.
* **Action:** For each line item (variant):

  * Write order metafields:

    * `batch_number` ← product `preorder.batch_number`
    * `ship_month` ← `variant.batch_ship_message_override || product.ship_month`
  * Add order tag: `PreOrder-B{batch_number}`.

### Flow #4: Inventory Change (Rolling Variant Pre-Order)

* **Trigger:** Inventory quantity changed (variant).
* **Conditions:**

  * `inventory_quantity <= 0`
  * AND (`variant.variant_auto_preorder == true` OR `product.flow.auto_pre-order == true`)
  * AND `variant.exclude_from_batch != true`
* **Actions:**

  * PreProduct → Create listing for this variant with:

    * Lead time = `variant.lead_days_override || product.lead_days`
    * Ship message = `variant.batch_ship_message_override || product.batch_ship_message`
  * Slack alert.

### Flow #5: Release Fulfillment (Manual Trigger)

* **Operator step:** When inventory arrives, ops bulk-tags orders `release-now-bN`.
* **Trigger:** Order tags added.
* **Condition:** Tag contains `release-now-b`.
* **Action:** PreProduct → Release fulfillment (lift hold → unfulfilled).
* **Slack:** Release confirmation.

---

## Manual Actions Required

* Ops team: set `next_ship_month/message` around the 25th (unless automated).
* Ops team: bulk-tag orders `release-now-bN` when inventory lands.
* All other batch transitions, order tagging, and variant logic are automated.



