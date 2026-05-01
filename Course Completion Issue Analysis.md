# Course Completion Issue Analysis

## Problem Statement
An online learning platform tracks video progress and unlocks certificates when 100% of the course is completed.

Many users report:
- Videos show fully watched (100%)
- Certificates remain locked even after refresh and re-login

---

## Assumptions Based on the Scenario

- UI progress may not be the final truth  
  Just because the frontend shows 100% does not mean backend recorded it.

- Course completion may require more than videos  
  Quizzes, assignments, or other requirements may exist.

- Video watch data may not be fully sent  
  Network issues or event failures may block final updates.

- Completion event may not be triggered  
  Skipping or closing early may prevent event firing.

- Certificate unlock process may have failed  
  Backend job may not execute even after completion.

---

## Possible Backend Failures

### 1. Event Tracking Issues
- Final "video completed" event not sent
- Events rejected due to validation errors
- Wrong user ID or course ID
- Last few seconds not captured

### 2. Progress Calculation Issues
- UI shows 100%, backend shows 99%
- Rounding differences
- Required content incomplete
- Multi-device progress not merged

### 3. Completion Trigger Issues
- Course not marked as completed
- Certificate job failed
- Queue/background job not executed
- Bug in completion logic

---

## How to Verify

### Step 1: Check Event Tracking
- Review logs for affected user
- Confirm "video completed" event received
- Check validation errors

### Step 2: Check Backend Progress
- Verify stored progress percentage
- Confirm all required items completed
- Manually recalculate progress

### Step 3: Check Completion & Certificate
- Verify completion status in DB
- Check certificate job trigger
- Review job logs for failures

---

## Edge Cases

### 1. Video Watching Behavior
- User skipped to end
- Watched at 2x speed
- Closed tab immediately
- Poor internet connection
- Background mode interruption
- Autoplay skipped completion event

### 2. Frontend Issues
- UI shows local/cached data
- Backend not updated yet
- Multiple tabs/devices mismatch

### 3. Event Tracking Problems
- Missing final event
- Out-of-order events
- Blocked by browser/ad blockers
- Incorrect user/course ID
- Duplicate event handling removed valid event

### 4. Progress Calculation Problems
- Rounded to 100% in UI
- Quiz/assignment incomplete
- New content added later
- Hidden required lesson
- Multi-device sync failure

### 5. User Account Issues
- Enrollment expired
- Different account used
- Trial restrictions
- Organization rules

### 6. Certificate Generation Issues
- Background job failure
- Certificate record not created
- Queue delay
- Stuck in pending/failed state

---

## Risks

### 1. User Trust Risk
- Loss of confidence
- Negative reviews
- Brand damage

### 2. Support & Operational Risk
- Increased support tickets
- Manual workload increase
- SLA breaches

### 3. Revenue Risk
- Refund requests
- Drop in renewals
- Lower conversion rates

### 4. Data Integrity Risk
- Frontend/backend mismatch
- Incorrect analytics
- Reporting issues

### 5. Technical Risk
- Event pipeline failures
- Queue backlog
- Race conditions
- DB replication lag

### 6. Compliance Risk
- Certification issues for professional use
- Legal disputes
- Accreditation concerns

### 7. Scalability Risk
- Delayed processing at scale
- Performance bottlenecks

### 8. Security Risk
- Bypass of completion rules
- Over-strict validation blocking users
- Exploitable logic gaps

---
