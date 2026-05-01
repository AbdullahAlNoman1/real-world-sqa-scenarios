# Currency Mismatch Issue in E-commerce Checkout

## Problem Summary
Users can switch currency (USD, EUR, BDT).  
Product listing updates correctly, but checkout total remains in the old currency — leading to incorrect payments.

---

## Assumptions Based on Scenario

### Key Assumptions

- Currency switch only updates **UI display (product listing)**.
- Checkout total is **not recalculated** after currency change.
- There is a mismatch between:
  - **Display currency (frontend)**
  - **Transaction currency (backend/payment)**
- Multiple sources of truth exist:
  - Product listing (frontend or pricing API)
  - Cart/checkout (backend services)
- Payment provider charges based on **backend request**, not UI.
- Currency selection is stored (cookie/session/local storage), but not consistently shared.

---

## Likely Integration Gaps

### 1. Missing Currency in API Requests
- Frontend updates listing using selected currency.
- Cart/checkout APIs do NOT send currency parameter.
- Backend continues using default currency (e.g., USD).

---

### 2. Cart Stores Price Without Currency Context
- Example:
  - `unit_price = 100` (no currency)
- Prices are stored once and not updated.
- Leads to incorrect totals after currency change.

---

### 3. No Repricing Trigger
- Listing uses real-time pricing.
- Checkout uses cached/cart snapshot values.
- Currency change does not trigger recalculation.

---

### 4. UI-Only Conversion
- Frontend converts price visually using FX rate.
- Backend still uses original currency.
- Result: mismatch between shown and charged amount.

---

### 5. Caching Issue
- Cart/checkout totals cached per session.
- Cache key does NOT include currency.
- Old totals reused after currency switch.

---

### 6. Different Calculation Logic
- Listing shows base price only.
- Checkout includes:
  - Tax
  - Shipping
  - Discounts
- Currency applied inconsistently.

---

### 7. Payment Uses Old Currency
- Payment intent created before currency change.
- Not updated/recreated.
- Gateway charges incorrect currency.

---

### 8. Missing Validation Before Payment
System does NOT validate:
- `order.currency == selected_currency`
- `display_total == payable_total`

---

## Test Scenarios

### 1. Currency Propagation Test
Verify currency consistency across:
- Product listing
- Product details
- Cart
- Checkout

---

### 2. Cart Recalculation Test
Steps:
- Add items in USD
- Switch to EUR/BDT

Verify:
- Line item prices update
- Discounts recalculate
- Tax & shipping update
- Grand total correct

---

### 3. Backend Validation Test
Check API:
- Request includes correct currency
- Response includes:
  - Currency code
  - Unit price
  - Total amount

---

### 4. Cache Validation Test
- Switch currency multiple times
- Ensure no stale totals are reused

---

### 5. Payment Integration Test (Critical)
Verify:
- Payment request amount
- Payment request currency
- Gateway charge matches checkout

---

### 6. Edge Case Testing
Test with:
- Coupons (percentage & fixed)
- Shipping options
- Tax calculations
- Rounding differences

---

## Risks

### 1. Financial Loss
- Incorrect charges → revenue loss or refunds

### 2. Customer Trust Risk
- Mismatch reduces trust and causes complaints

### 3. Chargeback Risk
- Users dispute incorrect charges

### 4. Legal/Compliance Risk
- Violates consumer protection laws in some regions

### 5. Data Integrity Risk
- Inconsistent order and accounting data

### 6. Caching Risk
- Stale data leads to unpredictable behavior

### 7. Pricing Engine Risk
- Incomplete recalculation causes incorrect totals

### 8. Operational Risk
- Increased support and finance workload

---

## Conclusion
This issue is mainly caused by **lack of synchronization between frontend display, backend pricing, and payment system**.  
Fix requires:
- Single source of truth for pricing
- Proper currency propagation
- Recalculation on currency change
- Strict validation before payment
