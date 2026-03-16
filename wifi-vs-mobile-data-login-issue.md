
# Login Works on Wi-Fi but Fails on Mobile Data

# Scenario
After releasing a new mobile app version, users can log in successfully when connected to Wi-Fi but fail when using mobile data.
The backend team says the authentication service is working fine and sees no errors in normal logs.

# Assumptions
- The same user credentials are being used on both Wi-Fi and mobile data
- Authentication API is reachable from at least one network path
- The issue started after a new mobile app release
- The failure may happen before request reaches backend logs
- App may use HTTPS, token-based auth, API gateway, CDN, or WAF

# Objective
Design tests to isolate whether the problem is caused by:
- network configuration
- timeout handling
- API security rules
- client-side logic

# Test Strategy
I would isolate the issue by changing one variable at a time.

- same user
- same device
- same app build
- switch only network type
- then compare behavior across logs, API traces, and app-side debug output

This helps determine whether the issue is network-path-specific or application-specific.

---

# Step 1: Reproduce in a controlled matrix

# Devices
- Android device
- iPhone device

# App versions
- New version
- Previous stable version

# Networks
- Wi-Fi
- Mobile data 4G/5G
- Different carriers if possible
- Hotspot from another phone

# User types
- Existing valid user
- New user
- Different regions/accounts if relevant

# Goal
Build a comparison table like:
- old app + Wi-Fi = pass
- old app + mobile data = pass/fail
- new app + Wi-Fi = pass
- new app + mobile data = fail

# Interpretation
- If only the new version fails on mobile data, issue is likely in app logic, networking config, or request handling introduced in the release.
- If both old and new fail on mobile data, issue may be server/network environment related.

# Step 2: Validate whether request leaves the client
Use tools such as:
- Charles Proxy
- Proxyman
- Android Studio Network Inspector
- iOS network debugging tools
- app debug logs

# Verify:
- does login request get created?
- is request actually sent on mobile data?
- does DNS resolve?
- does TLS handshake complete?
- is there any client-side exception before API call?

# Possible findings
- request never sent → client-side logic problem
- request sent but no response → network routing / timeout / blocking
- response received but app shows failure → response parsing or UI logic issue

# Step 3: Test for network configuration issues

# What to validate
- API domain reachable over mobile network
- DNS resolution works on mobile data
- IPv4 vs IPv6 compatibility
- SSL certificate chain is accepted on mobile network
- CDN / API gateway / firewall rules differ by network path
- app transport security config allows the endpoint

# Tests
# 1. Endpoint reachability test
From device/browser or test app:
- hit auth base URL on Wi-Fi
- hit same URL on mobile data

Check:
- response time
- DNS resolution
- connection refused / reset / timeout

# 2. Compare IP families
Some mobile carriers prefer IPv6.
Test:
- does app/backend fully support IPv6?
- does endpoint fail only on IPv6 but work on IPv4?

# 3. Carrier-specific testing
Test on:
- SIM operator A
- SIM operator B
- hotspot network

# 4. Proxy/VPN comparison
Try:
- mobile data with VPN
- mobile data without VPN

# Interpretation
- if failure depends on carrier or network route, likely network config / DNS / firewall / IPv6 issue


## Step 4: Test timeout handling

The backend may say auth service is fine because the request may never survive long enough or may fail before normal application logs capture it.

# What to validate
- request timeout values in app
- connection timeout
- read timeout
- retry logic
- behavior in slow or unstable networks

# Tests
# 1. Simulate slow network
Use network throttling:
- high latency
- packet loss
- low bandwidth

Check:
- does login fail only under slower conditions?
- is timeout too aggressive for mobile data?

# 2. Compare timeout behavior
Observe:
- Wi-Fi response = 1 to 2 sec
- mobile data response = 4 to 8 sec
- app timeout may be 3 sec

# 3. Retry behavior test
Validate:
- does app retry failed auth requests?
- does it cancel too early?
- is there exponential backoff?

# 4. App state handling
Check if:
- loading spinner ends too soon
- timeout error shown incorrectly
- request still completes after UI already failed

# Interpretation
- if request works when timeout is increased or on slightly faster mobile networks, timeout handling is likely the issue

---

# Step 5: Test API security rules

Sometimes backend service is healthy, but traffic from mobile data may be blocked earlier by:
- WAF
- API gateway
- CDN
- geo/IP restrictions
- bot protection
- rate limiting
- header validation

# What to validate
- are requests from mobile carrier IP ranges blocked?
- does API gateway reject requests without showing in normal app logs?
- is TLS version/security policy different on mobile path?
- are certain headers/token formats stripped or altered?

# Tests
# 1. Compare raw requests
Capture login request from:
- Wi-Fi
- mobile data

Compare:
- headers
- auth token fields
- user-agent
- content-type
- host
- request body
- timestamp/skew-sensitive fields

# 2. Check gateway/WAF logs
Backend normal logs may be clean because request is blocked before reaching auth service.
Validate:
- API gateway access logs
- WAF deny logs
- CDN edge logs
- load balancer logs

# 3. Rate limit test
Try repeated login attempts on mobile data:
- does it start failing after few attempts?
- does carrier NAT cause many users to appear from same IP?

# 4. SSL/TLS security test
Validate:
- TLS handshake success
- supported cipher suites
- certificate pinning behavior
- SNI issues

# Interpretation
- if requests are absent in auth logs but appear in gateway deny logs, issue is security/routing layer, not auth service itself

---

# Step 6: Test client-side logic

Because the problem started after a new release, client code change is a strong suspect.

# What to validate
- network type detection logic
- conditional request building
- feature flag behavior
- environment switching
- offline/online state handling
- token storage / request signing logic

# Tests
# 1. Network-type based code path
Check whether app does different logic for:
- Wi-Fi
- cellular
- metered connection
- roaming

Possible bug examples:
- app disables login over metered network
- app attaches wrong base URL on mobile data
- app blocks request due to "data saver" logic

# 2. Compare app logs
Add debug logs for:
- selected base URL
- request payload creation
- auth header creation
- timeout values
- detected network type
- error callback

# 3. Build comparison
Run:
- previous version on mobile data
- new version on mobile data

If old version works and new fails:
- inspect changed code in networking layer
- inspect SDK/library upgrade
- inspect certificate pinning changes
- inspect HTTP client config changes

# 4. Data saver / permission scenarios
Validate:
- background/mobile data permission
- restricted data usage settings
- OS-level data saver mode
- app-specific cellular data disabled setting

# 5. Response handling test
Check if on mobile data:
- request succeeds but response parsing fails
- compressed response behaves differently
- app misclassifies network error as auth failure

# Interpretation
- if request reaches backend and succeeds, but app still shows login failure, issue is client-side handling or UI logic

---

# Logs and Tools I Would Check

# Client-side
- app debug logs
- crash logs
- networking library logs
- Android Logcat
- iOS Console logs

# Edge / Network
- DNS logs
- API gateway logs
- CDN logs
- WAF logs
- load balancer logs

# Backend
- auth service access logs
- auth service application logs
- reverse proxy logs

# Test Tools
- Charles Proxy / Proxyman
- Postman
- Curl
- Android Studio Network Profiler
- Xcode Instruments
- packet capture if available
- network throttling tools

---

# Isolation Logic

# Case 1: Request never leaves app on mobile data
Likely cause:
- client-side logic
- app permission / network detection bug
- mobile data restriction in app

# Case 2: Request leaves app but never reaches backend/gateway
Likely cause:
- DNS/routing problem
- carrier-specific network issue
- TLS/transport issue

# Case 3: Request reaches gateway but not auth service
Likely cause:
- API gateway rule
- WAF/security filtering
- header or token rejection

# Case 4: Request reaches auth service but mobile data fails under delay
Likely cause:
- timeout handling
- weak retry logic
- poor low-network resilience

# Case 5: Request succeeds but app still shows failure
Likely cause:
- client-side response handling
- parsing/UI state bug
- async callback issue

---

# High-Value Test Cases

# Test Case 1
- Same device
- Same user
- New app version
- Login on Wi-Fi
- Login on mobile data
Expected: compare request path and behavior

# Test Case 2
- New version on mobile data
- Simulate slow network
Expected: determine if timeout threshold is too low

# Test Case 3
- Previous app version on mobile data
- New app version on mobile data
Expected: isolate regression in client release

# Test Case 4
- Mobile data across multiple carriers
Expected: detect carrier/network-specific blocking

# Test Case 5
- Inspect gateway/WAF logs for blocked cellular-origin IPs
Expected: identify security-layer rejection

# Test Case 6
- Enable verbose client logs and compare request construction by network type
Expected: identify conditional logic bug

# Risks
- false assumption that backend is healthy because normal logs show nothing
- issue may exist before request reaches service logs
- customer impact high because login failure blocks all usage
- bug may affect only certain carriers or regions, making it harder to detect
- release regression may be hidden unless tested on real mobile networks

# Conclusion
I would isolate the issue by testing the same login flow across controlled combinations of:
- app version
- device
- Wi-Fi vs mobile data
- different carriers
- normal vs slow network conditions

Then I would trace whether the request:
- is created correctly,
- leaves the client,
- reaches the gateway,
- passes security layers,
- reaches auth service,
- and is handled correctly by the app.

This approach clearly separates whether the issue is caused by:
- network configuration,
- timeout handling,
- API security rules,
- or client-side logic.
