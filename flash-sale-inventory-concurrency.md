# Flash Sale Inventory Concurrency Issue

# Scenario
During a massive flash sale, thousands of users add the same limited-stock product to their carts.
Some users complete payment, but later receive cancellation messages saying Out of stock.
At the same time, some users who paid later successfully receive the product.

# Assumptions
- Product stock is limited
- Multiple users are checking out at nearly the same time
- Inventory is updated through backend services and database transactions
- Cart, payment, and order services may be separate services
- Stock may be reserved either at add-to-cart time or at checkout/payment time

# Objective
Identify possible concurrency issues and design tests to validate:
- inventory locking
- race conditions
- transaction atomicity

# What Concurrency Issues Could Be Happening?

# 1. Overselling due to race condition
Two or more users may read the same available stock before any one transaction updates it.

Example:
- Stock available = 1
- User A reads stock = 1
- User B also reads stock = 1
- Both proceed to payment
- Both think item is available

This creates overselling.

---

# 2. No proper inventory locking
If stock rows/items are not locked during reservation or checkout:
- multiple requests may update the same stock concurrently
- inventory count may go negative
- stock may be reserved for the wrong user sequence

---

# 3. Delayed stock reservation
If the system reserves stock only **after payment success**:
- multiple users can pay for the same last item
- later, the system realizes stock is insufficient
- some paid users get cancellation

This explains why earlier payers may still lose the item.

---

# 4. Inconsistent order of processing
Payment completion time and inventory allocation time may not match.

Example:
- User A pays first
- User B’s inventory reservation job executes first
- User B gets product
- User A gets cancelled later

This can happen in async systems or queued processing.

---

# 5. Non-atomic payment and inventory update
If payment capture and stock deduction happen in separate transactions:
- payment may succeed
- stock deduction may fail
- order may later be cancelled

This is a classic partial success problem.

---

# 6. Lost update problem
Two transactions update the same stock record, and one overwrites the other’s result.

Example:
- both processes read stock = 10
- both subtract 1
- both write back 9

Actual expected stock should be 8, but system stores 9.

---

# 7. Stale cache / delayed stock sync
One service may read stock from cache while another writes to DB:
- checkout sees old stock
- payment succeeds based on stale data
- fulfillment later checks real stock and cancels


# 8. Cart reservation expiry issue
If add-to-cart temporarily reserves stock:
- reservation expiry may not release stock correctly
- expired reservations may still block others
- or stock may be double-used

# 9. Queue/event ordering issue
In an event-driven architecture:
- payment success event
- stock reservation event
- order confirmation event

If these events are processed out of order or retried incorrectly:
- later payer may get stock
- earlier payer may fail

# 10. Idempotency / duplicate processing issue
Retry or duplicate requests may:
- reserve stock twice
- process payment twice
- create conflicting inventory states

---

# How I Would Test Inventory Locking

# Goal
Validate that when many users try to buy the same limited-stock item, inventory is reserved correctly and only allowed quantity succeeds.

# Test 1: Single remaining stock with parallel checkout
- Set stock = 1
- Launch 20–100 parallel checkout/payment attempts
- Same product, different users

# Validate
- only one order should be confirmed
- all others should fail gracefully before payment capture or be refunded automatically
- stock should never go below 0

# Expected result
If locking works properly, exactly one successful allocation should happen.

---

# Test 2: Small stock, very high concurrency
- Set stock = 10
- Run 500 or 1000 simultaneous purchase attempts

# Validate
- successful confirmed orders <= 10
- no negative stock
- no duplicate allocation
- no paid-but-unfulfilled orders beyond controlled/expected failure policy

---

# Test 3: Lock duration behavior
- Simulate user entering checkout but delaying payment
- another user completes payment during that time

# Validate
- whether stock is locked at checkout, payment initiation, or payment confirmation
- whether lock expires correctly
- whether stock is released on timeout


# Test 4: Lock release on failure
- reserve stock
- fail payment intentionally
- verify stock is released correctly

# Validate
- no stuck reservation
- stock becomes available again
- next user can successfully buy

# How I Would Test Race Conditions

# Goal
Create overlapping requests to expose timing-related bugs.

# Test 1: Simultaneous read-update race
Use load/concurrency tools to fire many requests at the exact same moment.

Example tools:
- JMeter
- k6
- Locust
- Gatling
- custom scripts

# Validate
- two users should not both read and consume same last stock
- DB updates should be serialized or safely handled

---

# Test 2: Artificial delay injection
Introduce delay between:
- stock check
- stock reserve
- payment success
- order confirmation

# Purpose
Widen the race window and see if inconsistent outcomes appear.

Example:
- delay order service by 2 seconds
- payment completes instantly
- launch competing users

# Validate
- stock consistency remains correct even under delayed service execution

# Test 3: Cross-service race
If services are separate:
- checkout service
- payment service
- inventory service
- order service

Test multiple users completing the full flow concurrently.

# Validate
- request sequence integrity
- consistent winner selection
- correct stock count after all events processed


# Test 4: Retry race
Trigger retries from:
- payment callback
- inventory update
- order confirmation

# Validate
- duplicate retries do not double-deduct stock
- same order is not processed twice

---

# How I Would Test Transaction Atomicity

# Goal
Ensure payment, inventory deduction, and order confirmation behave as one reliable business transaction, even if implemented across multiple services.

# Test 1: Payment success, inventory failure
Simulate:
- payment captured successfully
- inventory update fails due to DB error/service timeout

# Validate
System should do one of these safely:
- rollback payment / auto-refund
- mark order pending and retry reservation
- never leave silent paid-but-cancelled inconsistency without compensation


# Test 2: Inventory deducted, order creation fails
Simulate:
- stock reduced
- order record creation fails

# Validate
- stock must be restored, or
- operation must be retried safely
- no stock leakage

# Test 3: Order created, payment callback duplicated
Simulate duplicate payment notification.

# Validate
- order should not be duplicated
- stock should not be deducted twice
- transaction must be idempotent

# Test 4: System crash in the middle of flow
Inject failure after:
- payment recorded
- before inventory commit
or
- after inventory commit
- before order confirmation

# Validate
- recovery logic
- compensation transaction
- data reconciliation correctness

# Specific Failure Points I Would Validate

# Inventory Service
- row-level locking
- optimistic locking version checks
- stock decrement logic
- reservation expiry logic

# Payment Service
- duplicate callback handling
- payment capture timing
- refund/compensation flow

# Order Service
- order creation idempotency
- state transition correctness
- pending vs confirmed vs cancelled status

# Database
- transaction isolation level
- deadlocks
- lost updates
- rollback behavior
- commit ordering

# Queue/Event System
- event duplication
- event ordering
- retry behavior
- eventual consistency delays

# Cache Layer
- stale inventory reads
- delayed invalidation
- DB-cache mismatch

---

# APIs and Logs I Would Check

# APIs
- add to cart API
- reserve inventory API
- checkout API
- payment confirmation API
- order creation API
- refund/cancel API

# Logs
- inventory service logs
- payment service logs
- order service logs
- queue/event logs
- DB transaction logs
- cache invalidation logs

# Key fields to trace
- product_id
- stock_before / stock_after
- user_id
- cart_id
- order_id
- payment_id
- reservation_id
- correlation_id
- timestamp

---

# High-Value Test Cases

# Test Case 1: Last item under concurrency
- stock = 1
- 100 concurrent users attempt purchase
Expected:
- only 1 confirmed order
- others fail safely

# Test Case 2: Payment earlier but allocation later
- user A starts first
- user B pays slightly later
- add processing delay to A’s order service
Expected:
- system should have deterministic stock allocation rules
- no unfair out-of-order confirmation without business reason

# Test Case 3: Payment succeeds but stock unavailable
- simulate stock depletion between payment and reservation
Expected:
- proper compensation/refund flow
- clear customer messaging
- no hidden inconsistency

# Test Case 4: Duplicate callback/idempotency
- send same payment success callback 2–3 times
Expected:
- single confirmed order
- single stock deduction

# Test Case 5: Reservation timeout
- hold stock in cart
- let reservation expire
- another user buys product
Expected:
- stock release and reuse should be accurate


# What Results Would Indicate a Bug?

- confirmed orders exceed available stock
- stock count becomes negative
- stock count does not match successful orders
- earlier successful payment loses stock unfairly without defined rule
- later payer receives stock while earlier payer is cancelled due to processing race
- duplicate orders created from same payment
- payment captured but no confirmed order/refund
- reserved stock never released

# Risks
- overselling during peak sale
- customer trust damage
- refund overload
- inventory mismatch
- unfair order fulfillment
- finance and fulfillment reconciliation problems
- support escalation spike

# Recommendations
To avoid these issues, system should ideally implement:
- atomic stock reservation
- proper locking strategy
- idempotent payment/order processing
- compensation flow for partial failure
- clear ordering rule for allocation
- reconciliation job for paid-without-order cases
- monitoring for oversell events

# Conclusion
The likely root cause is a concurrency/control problem between payment, stock reservation, and order confirmation.

I would test this by creating high-concurrency purchase attempts and validating:
- whether inventory locking prevents oversell
- whether race conditions allow inconsistent winner selection
- whether payment, inventory, and order transitions remain atomic and recoverable
