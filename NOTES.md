# NOTES

Fill in every field below before you submit. We read this file first: a submission with an empty field is not graded.
Write in your own words. If you did not fix something, set Status to `not fixed` and say what you found and tried; that still counts.
Status is one of: `fixed`, `partly fixed`, `not fixed`. Commits: the short hashes of the commits for this item, or `none`.

## 1. Report

> Finance says that on some days the revenue in the daily report is higher than what the payment provider paid out, and a few orders show more money received than their total.

**Status:** fixed

**What I saw:**
When the payment provider sent duplicate/retry webhook events for the same transaction (`id`), `tillpoint/payments.py` inserted another record into the `payments` table every time without checking whether the `event_id` was already processed. Consequently, `self.paid_cents(order_id)` accumulated multiple times the paid amount, and the daily sales report (`reports.daily_sales`) aggregated duplicate payment amounts into the total revenue.

**Root cause:**
The payment webhook handler lacked idempotency handling for repeat deliveries of the same webhook event ID (`event["id"]`). The schema in `tillpoint/db.py` did not declare `event_id` as `UNIQUE`, and `Payments.handle_event` unconditionally executed an `INSERT INTO payments` on every incoming webhook call.

**Fix and why:**
1. In `tillpoint/db.py`, updated the `payments` table schema to mark `event_id TEXT NOT NULL UNIQUE` and added a unique index `idx_payments_event`.
2. In `tillpoint/payments.py`, updated `handle_event` to check if `event["id"]` already exists in `payments` before inserting. If it exists, it returns the current order status idempotently without duplicate insertion.
This prevents webhook retries from duplicating payments in the database while safely returning a success status to the gateway.

**How I verified it:**
Added an automated test `test_payment_webhook_retry_is_idempotent` in `tests/test_orders.py`. Before the fix, sending duplicate webhook payloads with the same `event_id` produced `2400` cents paid instead of `1200` (`AssertionError: 2400 != 1200`). After the fix, the test passes with `paid_cents == 1200` and daily revenue report showing `1200`.

**Commits:** 9ea0a30

## 2. Report

> Products show as sold out, or nearly, while the shelf is full. Staff say it gets worse on days when many customers abandon checkout.

**Status:** fixed

**What I saw:**
When an order was created in `Orders.create`, items were immediately reserved from catalog stock via `self.catalog.reserve(sku, qty)`, and the order status was set to `pending`. When an unpaid customer abandoned checkout and the pending order was cancelled via `Orders.cancel(order_id)`, the inventory was never restored. The product's available stock remained reduced.

**Root cause:**
In `tillpoint/orders.py`, the `cancel` method contained an improper condition:
`if order["status"] == "paid": for item in order["items"]: self.catalog.release(item["sku"], item["qty"])`.
Because abandoned checkouts were in `"pending"` status (unpaid), this condition evaluated to False. Therefore, stock reserved during order creation was only released if the order had already been paid, permanently locking up stock for all cancelled pending orders.

**Fix and why:**
In `tillpoint/orders.py`, removed the check requiring the order to be in `"paid"` status in `cancel()`. Any non-shipped order being cancelled (`pending` or `paid`) now releases its reserved items back to the catalog via `self.catalog.release(item["sku"], item["qty"])`.

**How I verified it:**
Added an automated test `test_cancel_pending_order_restocks` in `tests/test_orders.py`. Before the fix, creating an order for 2 units of an item with stock 3 and cancelling it left stock at 1 (`AssertionError: 1 != 3`). After the fix, the test passes and stock is restored to 3.

**Commits:** cfda726

## 3. Report

> Customers in taxed regions who used a coupon say the tax on their receipt is too high. Receipts without a coupon are right.

**Status:** fixed

**What I saw:**
For orders placed in taxable regions (such as CA, NY, TX, DE) with a coupon applied, the calculated tax was based on the full pre-discount subtotal rather than the discounted subtotal. For example, for a 1200 cent item with a 10% coupon (120 cents discount) in CA (7.25% tax), the tax was calculated as 87 cents (7.25% of 1200) instead of 78 cents (7.25% of 1080).

**Root cause:**
In `tillpoint/pricing.py`, `price_order` computed `tax = tax_for(sub, region)` before calculating the coupon discount, passing the gross `subtotal` instead of `subtotal - discount`. Receipts without a coupon had `discount = 0`, which masked the bug.

**Fix and why:**
In `tillpoint/pricing.py`, reordered the calculations in `price_order`: first compute `discount = discount_for(coupon, sub)`, then compute `taxable = max(0, sub - discount)`, and calculate tax using `tax = tax_for(taxable, region)`. This ensures sales tax is assessed strictly on the post-discount taxable amount.

**How I verified it:**
Added an automated test `test_order_totals_with_coupon_and_tax` in `tests/test_orders.py`. Before the fix, the test failed with `AssertionError: 87 != 78`. After the fix, the test passes, computing tax as 78 cents and total as 1158 cents.

**Commits:** 52f8da2

## 4. Report

> Our shop in India (UTC+5:30) says sales taken in the morning do not appear in that day's report and turn up under the day before.

**Status:** fixed

**What I saw:**
Transactions recorded in the morning in India (e.g. 9:00 AM IST on 2025-03-10, which corresponds to 03:30 UTC on 2025-03-10) did not show up in the report for `2025-03-10` with `utc_offset_minutes=330`, but instead appeared in the previous day's report (`2025-03-09`).

**Root cause:**
In `tillpoint/reports.py`, `day_bounds_utc` converted the local calendar day start into UTC by adding `timedelta(minutes=utc_offset_minutes)` instead of subtracting it:
`start += timedelta(minutes=utc_offset_minutes)`.
To convert local time to UTC, the UTC offset must be subtracted (`UTC = Local - Offset`). Adding +330 minutes set the start of India's day to 05:30 UTC (11:00 AM IST), so any sales between midnight and 11:00 AM IST were treated as belonging to the previous UTC interval.

**Fix and why:**
In `tillpoint/reports.py`, changed `start += timedelta(minutes=utc_offset_minutes)` to `start -= timedelta(minutes=utc_offset_minutes)`. For UTC+5:30 (+330 min), the local day starts at 18:30 UTC on the previous date and ends at 18:30 UTC on the day, correctly capturing morning sales.

**How I verified it:**
Added an automated test `test_daily_sales_with_positive_utc_offset_india` in `tests/test_orders.py`. A sale recorded at 03:30 UTC (9:00 AM IST) on `2025-03-10` failed before the fix with `{'revenue_cents': 0, 'orders': 0} != {'revenue_cents': 1200, 'orders': 1}`. After the fix, the sale is correctly reported on `2025-03-10` and omitted from `2025-03-09`.

**Commits:** cb2dade

## 5. Report

> Ever since we started running two server workers, we have sold more units of an item than we had, and its stock went negative.

**Status:** fixed

**What I saw:**
With two worker processes running against the same database, inventory stock could be decremented below 0, allowing items to be oversold. When worker 1 cached a product with stock = 2 and worker 2 reserved those 2 units, worker 1's in-process cache still showed stock = 2. Worker 1 then approved another reservation, and the SQL update decremented 0 to -1 or -2.

**Root cause:**
1. `Catalog` in `tillpoint/catalog.py` kept an in-memory dictionary `self._cache` that cached entire `Product` objects, including mutable `stock`. Cache invalidation (`_cache.pop`) only affected the local worker process, leaving other worker processes with stale stock numbers.
2. `Catalog.reserve` checked availability in Python against the cached `product.stock`, and then executed an unconditional subtraction `UPDATE products SET stock = stock - ? WHERE sku = ?`.
3. The database schema in `tillpoint/db.py` lacked a `CHECK (stock >= 0)` constraint on the `products` table.

**Fix and why:**
1. In `tillpoint/db.py`, added a database constraint `CHECK (stock >= 0)` on `products.stock`.
2. In `tillpoint/catalog.py`, modified `Catalog._cache` so that it only caches static catalog metadata (`name`, `price_cents`), while `stock` is always queried live from the database in `get_product`.
3. In `tillpoint/catalog.py`, made `reserve` atomic at the SQL level using a conditional update: `UPDATE products SET stock = stock - ? WHERE sku = ? AND stock >= ?`. If `cur.rowcount == 0`, it verifies whether the product exists and raises `OutOfStock` if stock is insufficient.

**How I verified it:**
Added an automated test `test_two_workers_concurrent_stock_does_not_go_negative` in `tests/test_orders.py` using two separate `App` / `Catalog` connections simulating two worker processes. Before the fix, worker 1 reserved an item already depleted by worker 2, failing with `AssertionError: OutOfStock not raised`. After the fix, worker 1 correctly raises `OutOfStock` and stock in SQLite remains 0.

**Commits:** 52be87c

## 6. Open question

> Open question from support, no single right answer: sometimes a customer really does pay twice for one order (two separate payments, not a retry of the same one). Today the system records both and says nothing. Decide what should happen, build the smallest version of your decision, and explain what it costs in `NOTES.md`.

**My decision:**
When a separate second payment event is received for an order that is already paid, the system must accept and record the payment in `payments` (returning 200 to the webhook provider so gateway retries are avoided and financial audits match bank accounts), but immediately flag the order as overpaid by computing and recording `refund_due_cents = total_paid - total_cents` on the order and returning status `"overpaid"` from `handle_event`.

**Why:**
Payment webhooks report financial events that have already transpired in the real world: the customer's payment method has been charged. Dropping the event or responding with an HTTP error would break accounting reconciliation and cause the gateway to retry repeatedly. At the same time, quietly absorbing duplicate payments leads to unaddressed customer overcharges. Explicitly tracking `refund_due_cents` on the order gives support and automated systems a single source of truth for the excess amount that must be refunded, without blocking order fulfillment (`ship()` continues to work because the required funds were satisfied).

**What it costs:**
1. Operational overhead: Customer support or a background refund processor must periodically query orders where `refund_due_cents > 0` to trigger refunds via the payment gateway API.
2. API consumers must be aware that `handle_event` can return `"overpaid"` for duplicate payments on existing orders.
3. Accounting needs to differentiate between gross captured payments and net receipts after expected refunds.

**What I built:**
1. Added `refund_due_cents INTEGER NOT NULL DEFAULT 0` column to the `orders` table in `tillpoint/db.py`.
2. In `tillpoint/payments.py`, updated `handle_event` after recording the payment: if `total_paid > order["total_cents"]`, calculate `refund_due = total_paid - order["total_cents"]`, persist it via `UPDATE orders SET refund_due_cents = ?`, ensure the order status is marked `"paid"` (if it was pending), and return `"overpaid"`.
3. Preserved fulfillment capability: `Orders.ship()` continues to successfully ship the order since its status is `"paid"`.

**How I verified it:**
Added an automated test `test_duplicate_payment_records_overpaid_and_sets_refund_due` in `tests/test_orders.py`. It simulates a primary payment followed by a distinct second payment for the same order, verifying that `handle_event` returns `"overpaid"`, the order retains status `"paid"`, `refund_due_cents` is set to the exact surplus (1200 cents), and `ship()` executes cleanly.

**Commits:** 710ded9

## Other things I noticed

Problems outside the reports: what, where, and whether you fixed them. Write `none` if there were none.

1. Resource leak in tests: In `tests/test_orders.py`, SQLite connections opened during `setUp()` were never explicitly closed, resulting in `ResourceWarning: unclosed database` warnings during test runs on Python 3.15+. Added `tearDown()` to cleanly close the connection. Fixed in commit 52be87c.
2. In `tillpoint/orders.py`, `Orders.list_for` executes an N+1 query pattern by selecting order IDs and then calling `self.get()` for each ID. Not fixed as it is outside the scope of user reports and does not affect correctness.

## Session logs

Which AI tools you used, and one line per file in `session_logs/`.

- AI tool: Antigravity (Gemini 3.8 Flash)
- `session_logs/session_transcript.jsonl`: Complete raw session transcript containing all prompts, tool calls, and outputs.
- `session_logs/session_transcript.md`: Formatted readable Markdown log of the complete development session.
