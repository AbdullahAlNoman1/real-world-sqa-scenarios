# Banking App Issue: Duplicate Credit Problem

## 🧠 Scenario
A banking app shows **"Transfer Successful" instantly**, but later some users report:
- Receiver got money **twice**
- Sender was charged **only once**

---

## 📌 Assumptions Based on the Scenario

### 1. Debit and Credit Are Done in Different Systems
- Sender balance is reduced in one system  
- Receiver balance is increased in another system  
- Not handled in a single transaction  

---

### 2. Not a Single Atomic Transaction
- Debit and credit happen in **separate steps**  
- No all-or-nothing guarantee  
- Leads to inconsistency on failure  

---

### 3. Credit Happens Asynchronously
- After debit → message sent to queue  
- Background worker processes credit  
- Delay introduces risk of failure/retry issues  

---

### 4. System Has Automatic Retry
- Timeout or network issue triggers retry  
- Original request may have already succeeded  
- Retry causes **duplicate credit**  

---

### 5. At-Least-Once Message Delivery
- Message queues may deliver same message multiple times  
- Without duplicate handling → repeated credit  

---

### 6. No Proper Idempotency Check
Missing protections like:
- Unique `transfer_id`
- Idempotency key
- Database unique constraints  

---

### 7. App Shows Success Too Early
- Success shown after debit  
- Credit still pending in background  

---

### 8. Possible Failures
- Network timeout  
- Service crash  
- Server restart  
- Database failover  

---

## 🧪 Test Strategy

### 1. Duplicate Request Test
Send same transfer request multiple times using same `transfer_id`

---

### 2. Timeout and Retry Test
Simulate timeout after credit success  
➡ Ensure retry does NOT create duplicate credit  

---

### 3. Network Failure Test
Drop response after successful credit  
➡ Verify retry behavior  

---

### 4. Duplicate Message Test
Send same queue message multiple times  

---

### 5. Consumer Crash Test
Crash worker:
- After processing credit  
- Before acknowledgment  

---

### 6. Service Restart Test
Restart system during processing  

---

### 7. Concurrency Test
Send multiple identical requests simultaneously  

---

### 8. Partial Failure Test
- Debit succeeds  
- Credit fails  
➡ Verify safe handling  

---

### 9. Database Constraint Test
Attempt duplicate insert with same `transfer_id`  

---

### 10. Reconciliation Test
Verify:
- One debit per transfer  
- One credit per transfer  
- Correct final balance  

---

## ⚠️ Key Edge Cases

1. Credit success but response lost → retry → duplicate  
2. Duplicate message delivery  
3. Worker crash before acknowledgment  
4. Concurrent processing  
5. Missing idempotency key  
6. Partial failure  
7. Service restart during processing  
8. Multiple retry layers (Client + API + Worker)  

---

## 🚨 Risks

### 1. Financial Loss
Duplicate credits = direct money loss  

---

### 2. Data Integrity Risk
Ledger inconsistency  

---

### 3. Reconciliation Failure
End-of-day settlement issues  

---

### 4. Customer Trust Risk
Incorrect balances reduce confidence  

---

### 5. Regulatory & Compliance Risk
Violates financial regulations  

---

### 6. Fraud Risk
Users may exploit retry behavior  

---

### 7. Operational Risk
Manual correction overhead  

---

### 8. System Stability Risk
Retry storms may overload system  

---

## ✅ Key Fix Recommendations

- Use **idempotency keys**
- Enforce **unique constraints (transfer_id)**
- Implement **exactly-once processing (or safe deduplication)**
- Use **distributed transaction patterns (Saga / 2PC where needed)**
- Delay success message until **critical steps confirmed**
- Add **strong reconciliation jobs**

---

## 🏁 Conclusion
Duplicate credits usually occur due to:
- Retry mechanisms
- Async processing
- Lack of idempotency

Proper design + testing + safeguards are essential for financial systems.
