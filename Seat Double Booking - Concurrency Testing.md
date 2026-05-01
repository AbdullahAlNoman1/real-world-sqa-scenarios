#  Seat Double Booking - Concurrency Testing

##  Assumptions Based on Scenario

- Each seat for a showtime is unique in the database  
- Booking is confirmed only after database commit  
- Two users can click "Book" at the exact same time  
- Database is the single source of truth  
- Unique constraint exists on `(showtime_id + seat_id)`  
- Seat has states: `Available → Held → Booked`  
- System may run on multiple servers  
- Concurrency control relies on database locking  

---

## 🧪 Test Scenarios

### 1. Simultaneous Booking Test
**Purpose:** Check race condition handling  
- Two users book the same seat at the same time  

**Expected Result:**
- Only one booking succeeds  
- Second user gets "Seat not available"  
- Only one DB record exists  

---

### 2. High Traffic Test
**Purpose:** Verify system under load  
- 100+ users try booking same seat  

**Expected Result:**
- Only one success  
- Others fail properly  
- No duplicate records  
- System remains stable  

---

### 3. Database Unique Constraint Test
**Purpose:** Ensure DB prevents duplicates  

**Expected Result:**
- Duplicate insert fails  
- No two bookings for same seat  

---

### 4. Transaction Locking Test
**Purpose:** Validate DB locking  

**Expected Result:**
- Second user waits or fails  
- Only one booking confirmed  

---

### 5. Real-Time Seat Update Test
**Purpose:** Verify UI sync  

**Expected Result:**
- Seat becomes unavailable instantly  
- Other users cannot book  

---

### 6. Seat Hold Expiry Test
**Purpose:** Verify hold logic  

**Expected Result:**
- Seat becomes available after expiry  
- No booking created  

---

### 7. Duplicate Request Test
**Purpose:** Handle double click  

**Expected Result:**
- Only one booking created  
- No duplicate confirmation  

---

### 8. Confirmation Validation Test
**Purpose:** Ensure correct confirmation  

**Expected Result:**
- Confirmation only after DB commit  
- No false confirmation  

---

##  Edge Cases

1. Double click → Only one booking  
2. Network timeout + retry → No duplicate booking  
3. Stale UI → Backend rejects invalid booking  
4. Payment success but seat booked → Refund triggered  
5. Payment failed → Seat released  
6. Hold expires during payment → Booking fails/rechecked  
7. Server crash → Transaction rollback  
8. Deadlock → One fails safely  
9. Wrong notification → Only after commit  
10. Entry validation mismatch → DB validation required  
11. Multiple tabs/devices → Only one booking  
12. Cancel booking → Seat becomes available  

---

## 🚨 Risks

### 1. Double Booking Risk
- Two users get same seat  
- Impact: Revenue loss, trust issues  

### 2. False Confirmation Risk
- Both users receive confirmation  
- Impact: Customer dissatisfaction  

### 3. Data Integrity Risk
- Duplicate records  
- Impact: Reporting issues  

### 4. Real-Time Sync Risk
- Seat appears available incorrectly  
- Impact: Poor UX  

### 5. Payment Risk
- Paid but no booking  
- Impact: Refund issues  

### 6. Retry Risk
- Duplicate requests  
- Impact: Double charge  

### 7. Hold Management Risk
- Seat not released properly  
- Impact: Lost sales  

### 8. Deadlock Risk
- DB locking issues  
- Impact: System slow/failure  

### 9. Crash Risk
- App crash mid-process  
- Impact: Inconsistent data  

### 10. Entry Validation Risk
- Invalid ticket allowed/rejected  
- Impact: Operational issues  

### 11. Scalability Risk
- High load increases failures  
- Impact: Revenue loss  

### 12. Audit Risk
- Payment vs booking mismatch  
- Impact: Hard to investigate  

---

##  Summary

To prevent seat double booking:
- Use **DB-level unique constraints**
- Implement **transaction locking**
- Ensure **idempotent APIs**
- Sync **real-time seat availability**
- Send confirmation **only after commit**
