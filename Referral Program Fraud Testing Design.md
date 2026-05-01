# Referral Program Fraud Testing Design

## Assumptions Based on the Scenario

* The referral program gives a bonus when a user invites a friend.
* The bonus is granted after the referred user completes a required action (e.g., sign-up, verification, first purchase).
* The system primarily identifies users by email address.
* Minor email variations may be treated as different users.
* Duplicate referral payouts may currently be possible.
* Timing and event processing issues may exist.
* Strong identity validation (phone, device, payment method) may not be fully enforced.

---

## 1. Email Manipulation Tests

**Goal:** Prevent multiple bonuses using email variations.

### Test Cases

* Case variation: `User@x.com` vs `user@x.com`
* Leading/trailing spaces: `user@x.com` vs `user@x.com`
* Plus aliasing: `user+1@gmail.com` vs `user@gmail.com`
* Dot variation (Gmail): `u.ser@gmail.com` vs `user@gmail.com`
* Unicode spoofing: visually similar characters in email

### Expected Result

* Emails are normalized.
* Duplicate identities are detected.
* Only one referral reward is granted.

---

## 2. Same Person with Different Identity Signals

**Goal:** Detect users bypassing email checks.

### Test Cases

* Same phone number, different emails
* Same device fingerprint, different emails
* Same payment method, different emails
* Same address/name similarity

### Expected Result

* Identity clustering detects duplicate users.
* Referral bonus is limited to one per real person.

---

## 3. Idempotency and Duplicate Payout Tests

**Goal:** Prevent double reward issuance.

### Test Cases

* Duplicate API calls for reward payout
* Retry of payment or reward service
* Duplicate webhook events
* Message queue reprocessing (at-least-once delivery)

### Expected Result

* Only one payout record exists.
* Repeated requests return “already processed”.

---

## 4. Timing and Race Condition Tests

**Goal:** Prevent exploitation via event ordering.

### Test Cases

* Referral click before signup vs after signup
* Simultaneous signups using same referral
* Device/browser switching during signup flow

### Expected Result

* Referral attribution is fixed at a defined step.
* No duplicate or conflicting rewards.

---

## 5. Self-Referral Tests

**Goal:** Block users referring themselves.

### Test Cases

* Same device/IP for both accounts
* Same phone number
* Same payment method/KYC identity

### Expected Result

* Self-referrals are blocked or flagged for review.

---

## 6. Abuse and Volume Pattern Tests

**Goal:** Detect automated or large-scale fraud.

### Test Cases

* One referrer creates many accounts in short time
* Multiple signups from same IP/device farm
* Unnatural fast signup-to-purchase behavior

### Expected Result

* Rate limiting or fraud detection triggers.
* Suspicious rewards are blocked or held.

---

## 7. Cross-Referrer Conflict Tests

**Goal:** Ensure one referral reward per user.

### Test Cases

* Referred user clicks multiple referral links
* Referral code applied after signup

### Expected Result

* Only one referrer receives credit (first-touch or last-touch rule).

---

## 8. Account Lifecycle Edge Cases

**Goal:** Prevent abuse via account recreation.

### Test Cases

* Delete account and re-register
* Change email after signup
* Reuse referral after account reset

### Expected Result

* Eligibility is tied to persistent identity, not mutable fields.

---

## 9. Data Integrity and Audit Tests

**Goal:** Ensure traceability and investigation capability.

### Test Cases

* Verify logs capture:

  * referrer_id
  * referred_id
  * normalized email
  * phone hash
  * device ID
  * IP address
  * timestamps

### Expected Result

* Full audit trail exists for fraud detection.

---

## Common Edge Cases

### Email Edge Cases

* Case differences
* Spaces before/after email
* Gmail aliasing (+ trick, dot trick)
* Temporary or disposable emails

### Tracking Edge Cases

* Cross-device signup (mobile → desktop)
* Incognito mode usage
* Multiple referral link clicks (A → B)
* Delayed signup after referral click

### Duplicate Event Edge Cases

* Double signup requests
* Payment retry duplication
* Webhook replay
* Double-click payment button

### Same Person Multiple Accounts

* Same phone, different emails
* Same device, different emails
* Same payment method reuse

### Self-Referral Edge Cases

* User referring themselves via second account
* Shared device/IP abuse

### Reward Rule Issues

* Reward without verification
* Reward after refund/cancellation
* Multiple rewards per purchase instead of first purchase only

### Account Delete & Recreate

* Account deletion followed by re-registration
* Email modification post-signup

### High Volume Fraud

* Referral farming
* Bot-driven signups
* Rapid signup/purchase cycles

---

## Risks in Referral Program

### 1. Financial Risk

* Multiple payouts for same user
* Fraudulent reward generation

### 2. Fraud Risk

* Fake accounts
* Self-referrals
* Bot-based referral farming

### 3. System / Technical Risk

* Duplicate events
* Race conditions
* Missing idempotency

### 4. Validation Loophole Risk

* Weak email-based identity checks
* Lack of multi-factor identity validation

### 5. Reputation Risk

* Loss of user trust
* Perception of unfair reward distribution

### 6. Compliance / Audit Risk

* Insufficient logs
* Difficulty in fraud investigation

### 7. False Positive Risk

* Blocking legitimate users (families, shared devices)
