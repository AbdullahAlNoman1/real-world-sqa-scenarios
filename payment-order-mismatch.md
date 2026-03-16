# Payment Successful but Order Not Created

# Scenario
A customer places an order in a food delivery app and completes payment successfully.  
The app shows Order Placed Successfully, but the restaurant dashboard never receives the order.  
Customer support can see the payment record, but there is no order record in the admin/admin panel.

# Assumptions
- The system has separate services for payment, order, and restaurant dashboard/notification
- Payment gateway sends a callback/webhook** after successful payment
- Order creation happens either:
  - synchronously via API call, or
  - asynchronously via message queue/event bus
- Payment and order records are stored in separate tables/services
- Correlation fields like `transaction_id`, `payment_id`, `cart_id`, `user_id`, or `order_reference` are available

# Objective
As an SQA engineer, the goal is to trace the full transaction lifecycle and identify where the breakdown happened between:

*Customer App → Payment Gateway → Payment Service → Order Service → Database → Restaurant Dashboard*

---

# End-to-End Flow I Would Trace

# 1. Start from the customer action
First, I would collect the exact transaction details:
- customer ID
- payment transaction ID
- timestamp
- cart/order request payload
- app version / platform
- merchant reference ID

This helps build a single trace path across all systems.

---

# 2. Validate payment gateway status
I would confirm from the payment gateway:
- Was the payment actually successful?
- What is the final transaction status? (success, authorized, captured, failed)
- Was the webhook/callback triggered to our backend?
- What response did our backend return to the gateway?
- Were there retries from the gateway?

# Things to verify
- transaction ID
- webhook delivery logs
- callback response code (200/400/500)
- duplicate callbacks
- timeout or signature validation issues

#Possible issue
Payment may be successful at gateway level, but webhook may fail before reaching backend.

---

# 3. Check Payment Service logs
Then I would inspect internal payment service logs using:
- transaction ID
- correlation ID
- request ID
- user ID

# I would validate:
- Did payment service receive the gateway callback?
- Was payment status saved in DB successfully?
- Did payment service call/publish the next step for order creation?
- Was any exception thrown after saving payment but before creating order?

# Important log checks
- inbound webhook request log
- signature/auth validation log
- payment persistence log
- downstream API/event trigger log
- retry/error logs

# Possible issue
Payment record may be saved successfully, but downstream call to order service may fail silently.

---

# 4. Validate order creation trigger
After payment success, I would verify how order creation is supposed to happen.

# If synchronous:
Check whether payment service called something like:
- `POST /orders/create`
- `POST /checkout/confirm`
- `POST /orders/place`

Validate:
- request payload
- response code
- response body
- timeout/retry behavior

# If asynchronous:
Check whether payment service published an event like:
- `payment_success`
- `order_create_requested`
- `checkout_completed`

Validate:
- event published successfully
- topic/queue name
- event payload correctness
- message acknowledgment
- consumer processing status

# Possible issue
Payment success event may have been published incorrectly, published to wrong topic, or never consumed.

---

# Services I Would Validate

# 1. Payment Gateway
- transaction success/failure
- callback/webhook delivery
- retry behavior
- signature/token validation

# 2. Payment Service
- payment confirmation handling
- payment DB write
- trigger to order service
- error handling
- idempotency handling

# 3. Order Service
- order creation request received or not
- schema/business validation
- inventory/menu validation
- DB insert status
- rollback/error logs

# 4. Inventory/Menu Service
- item availability check
- restaurant open/closed state
- menu item existence
- price mismatch validation

# 5. Notification / Restaurant Dashboard Service
- order push to restaurant dashboard
- websocket/event delivery
- dashboard refresh sync
- order visibility filters

# 6. Admin Panel / Support Service
- whether admin reads from same order DB or replicated data source
- replication delay / indexing issue
- partial write inconsistency

---

# Logs I Would Check

# Application Logs
- payment service logs
- order service logs
- checkout service logs
- notification service logs
- restaurant dashboard service logs

# Infrastructure Logs
- API gateway logs
- load balancer logs
- message queue logs
- container/pod logs
- ingress/reverse proxy logs

# Database Logs
- payment table insert logs
- order table insert/update logs
- deadlock/rollback logs
- replication lag logs

# Observability/Tracing Tools
If available, I would check:
- Kibana / ELK
- Grafana / Loki
- Datadog
- New Relic
- Jaeger / Zipkin distributed trace

---

# APIs I Would Validate

# Payment-related APIs
- payment initiation API
- payment callback/webhook endpoint
- payment status verification API

# Checkout-related APIs
- checkout confirm API
- order placement API
- cart-to-order conversion API

# Order-related APIs
- create order API
- get order by payment/reference ID
- order status API

# Restaurant-related APIs
- push order to restaurant dashboard
- fetch active orders for restaurant
- order notification API

---

## Failure Points I Would Investigate

# 1. Webhook failure
- payment successful, but callback never reached backend
- callback rejected due to invalid signature
- callback timed out
- callback returned 500

# 2. Payment saved, order not triggered
- payment record persisted
- service crashed before calling order service
- job/event not fired

# 3. Order API failure
- payment service called order API
- order API returned 4xx/5xx
- payload invalid
- timeout occurred

# 4. Queue/Event failure
- event published but not consumed
- consumer down
- message stuck in DLQ
- serialization/deserialization error

# 5. Order validation failure
- cart data missing
- item out of stock
- restaurant offline/closed
- price mismatch
- address validation failure

# 6. Database failure
- order insert failed
- transaction rolled back
- partial commit
- locking/deadlock issue

# 7. UI false success
- frontend showed “Placed Successfully” based only on payment success
- actual order creation confirmation was never validated

This is a very important product/design issue.

# Key Test Data / Fields to Correlate
To trace the issue properly, I would map these fields across all services:
- payment_transaction_id
- merchant_reference_id
- cart_id
- checkout_id
- user_id
- restaurant_id
- order_id
- correlation_id
- timestamp

Without correlation IDs, root-cause analysis becomes much harder.


## Specific Validation Questions I Would Ask
- Did payment succeed at gateway level?
- Did our system receive the gateway callback?
- Did payment service write the payment record?
- Did payment service send order creation request/event?
- Did order service receive that request/event?
- If yes, why was order not inserted?
- If order was inserted, why is it not visible in admin or restaurant dashboard?
- Is there any delay due to async processing or replication?
- Did frontend show success before backend confirmed order creation?


# Edge Cases
- payment succeeded but webhook arrived late
- duplicate payment callback created conflict
- order service temporarily down
- queue consumer lag
- cart expired before order creation
- restaurant became unavailable after payment
- network timeout caused customer success screen despite backend failure
- payment captured but downstream services failed

---

# Risks
- customer charged without getting service
- missing order causes customer dissatisfaction
- restaurant never prepares food
- reconciliation mismatch between finance and operations
- refund workload increases
- duplicate/manual order creation risk
- trust and brand damage

---

# What I Would Recommend as an SQA Engineer

# Functional recommendation
Frontend should show **"Order Confirmed"** only after:
- payment success is confirmed, and
- order record is successfully created

# Technical recommendation
- enforce distributed trace/correlation ID
- implement idempotent order creation
- retry failed order creation safely
- add alert for payment-without-order cases
- maintain reconciliation job:
  - payments with no orders
  - orders with no payments

# Monitoring recommendation
Create a production alert for:
- payment success recorded
- but no order created within X seconds/minutes

This will help detect such incidents immediately.

---

## Conclusion
I would trace the issue step by step from payment gateway confirmation to internal payment service, then to order creation flow, and finally to restaurant/admin visibility.

The main focus would be to identify whether the failure happened in:
- webhook delivery
- payment service processing
- API/event trigger
- order service validation
- database write
- or dashboard visibility layer

This issue is most likely caused by a partial success flow, where payment is recorded but order creation fails or is never completed.
