# Push Notification Delay Issue – Test Strategy

## Assumptions Based on the Scenario

### 1. About the Problem
- **Delayed**: Notification arrives more than 5–15 minutes after backend send.
- **Not received**: No notification within hours and no delivery confirmation.
- Issue only affects **push notifications** (not in-app messages).

---

### 2. Tracking and Logs
Each notification has a **unique ID**.

#### Backend Logs:
- Notification created time
- Added to queue
- Sent to push service (FCM/APNs)
- Push service response

#### App Logs:
- Device token creation/refresh
- Notification received time
- App state (foreground/background/terminated)

---

### 3. Device Token
- Token is valid and up-to-date
- Correct device targeted (multi-device case)
- Token refresh works after:
  - Reinstall
  - App update
  - Device change

---

### 4. Push Notification Services (FCM / APNs)
- Correct configuration (keys, topics, bundle ID)
- Valid payload size & format
- Proper priority (high/normal)
- Acknowledge possible provider delay

---

### 5. Device State & Background Restrictions
- Battery saver ON/OFF
- Background app refresh ON/OFF
- App force-closed
- OEM restrictions (Android)
- Silent vs alert notifications
- Notification permissions enabled

---

### 6. Network Conditions
- Slow/unstable internet
- High latency
- VPN/firewall restrictions
- Offline → reconnect scenario

---

### 7. Backend Queue
- Queue backlog
- High traffic
- Retry delays
- TTL (Time to Live)

---

### 8. Error Handling
- Invalid tokens handled
- Temporary vs permanent failures
- No silent failures

---

### 9. App Version & Environment
- Issue may affect specific versions
- Production vs staging differences

---

### 10. User Settings
- Notifications disabled
- Do Not Disturb / Focus mode
- Notification summary (iOS)

---

## Investigation Goal
Identify delay source:
- Backend queue?
- Push provider?
- Device receipt?
- OS display?

---

# Test Strategy

## 1. Device State Testing

### 1.1 App State
- Foreground
- Background
- Terminated
- After restart
- After reinstall

### 1.2 Screen State
- Locked vs unlocked
- Screen ON vs OFF
- Long idle state

---

## 2. Background & Battery Restrictions

### 2.1 Power Settings
- Battery saver ON/OFF
- Low power mode (iOS)
- Background refresh ON/OFF

### 2.2 OEM Devices (Android)
- Xiaomi
- Oppo
- Vivo
- Samsung

### 2.3 Doze Mode
- Long idle
- App unused for days

---

## 3. Network Testing

### 3.1 Network Type
- Strong WiFi
- Weak WiFi
- Mobile data
- Switching networks

### 3.2 Network Quality
- High latency
- Packet loss
- Intermittent connection
- Offline → reconnect

### 3.3 VPN / Firewall
- VPN ON/OFF
- Restricted networks

---

## 4. Push Service Testing

### 4.1 Payload & Priority
- High vs normal priority
- Silent vs alert
- Large vs small payload
- Collapse key usage

### 4.2 Token Testing
- Valid token
- Expired token
- Token after reinstall/update

### 4.3 Multi-Device
- Multiple devices per user
- Correct device targeting

---

## 5. Backend Queue Testing

### 5.1 Normal Load
- Single notification
- Small batch

### 5.2 High Load
- Bulk notifications
- Peak simulation

### 5.3 Retry Logic
- Temporary failure simulation
- Backoff validation

### 5.4 TTL Testing
- Expired notifications not delivered

---

## 6. Logging Validation
Compare:
- Backend send time
- Provider response time
- Device receive time

---

## 7. OS Permission Testing
- Permission ON/OFF
- Notification channel disabled
- Focus/DND mode
- Notification summary

---

## 8. Version Testing
- Old vs new app versions
- OS versions
- After update

---

# Edge Cases
- Low storage
- Incorrect device time
- App force-stopped
- Logout before notification
- Login after notification

---

# Validation Metrics
- Backend → Provider time
- Provider → Device time
- Device → Display time

### Issue Classification:
- Backend delay
- Push provider delay
- Device/OS restriction
- User settings issue

---

# Risks

## 1. Business Risks
- Poor UX
- Missed critical alerts
- Customer churn
- Revenue loss

---

## 2. Technical Risks
- Queue overload
- Push throttling
- Invalid tokens
- Silent failures

---

## 3. Device & OS Risks
- Background restrictions
- Disabled permissions
- OS differences

---

## 4. Network Risks
- Unstable internet
- VPN/firewall blocking

---

## 5. Data Risks
- Late outdated notifications
- Duplicate notifications
- Missing critical alerts

---

## 6. Monitoring Risks
- No end-to-end tracking
- Hard to detect root cause

---

## 7. Compliance Risks
- SLA violations (OTP, alerts)
- Security risks (fraud alerts delay)
