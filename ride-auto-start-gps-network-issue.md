# Ride Starts Automatically Without Driver Clicking "Start Trip"

# Scenario

In a ride-sharing app, some rides start automatically even though the driver never clicks the 'Start Trip' button.
This issue occurs mainly when the GPS signal is weak or when the driver switches between mobile data and Wi-Fi during the ride session.

The expected behavior is that a trip should only begin after the driver manually taps the Start Trip button in the app.


# Assumptions

* The app continuously tracks driver location using GPS/location services.
* Trip status updates depend on driver actions and backend validation.
* The app may use geofencing or proximity detection near pickup location.
* Location tracking runs through background services even when the app is minimized.
* Network changes (Wi-Fi ↔ mobile data) may trigger API retries or state synchronization.
* Weak GPS signals may cause incorrect location coordinates or sudden location jumps.


# Test Ideas

* Attempt to reproduce the ride flow where the driver approaches the pickup point but does not press Start Trip.
* Simulate weak GPS signal using low-accuracy location settings or testing in indoor areas.
* Switch repeatedly between Wi-Fi and mobile data before reaching the pickup location.
* Toggle Airplane mode ON/OFF to simulate network reconnect behavior.
* Keep the app in background mode then reopen it near the pickup location.
* Restart the app during the ride request and observe whether the ride auto-starts.
* Monitor API calls and logs related to:

  * trip status updates
  * driver location updates
  * reconnect or retry requests
  * Compare behavior on:

  * Android vs iOS
  * different device models
  * different app versions
* Simulate delayed or duplicated location updates from the device.
* Verify whether geofence logic mistakenly triggers trip start when driver is near pickup.



# Edge Cases

* GPS signal jumps suddenly from a distant location to the pickup location.
* Driver loses GPS accuracy while the app is in background mode.
* Network reconnect occurs exactly when location updates are being sent.
* Driver switches networks during trip initialization.
* Backend receives duplicate location events after reconnect.
* App restores previous trip session incorrectly after a restart.
* Weak GPS places driver inside pickup geofence incorrectly.
* Background location service triggers trip state change automatically.
* Network delay causes out-of-order trip status updates.
* Driver switches between apps while approaching pickup location.

---

# Risks

* Trips may start without the driver’s consent.
* Fare calculation may begin earlier than intended.
* Customers may be charged incorrectly.
* Driver earnings records may become inaccurate.
* Backend trip state may become inconsistent.
* Customer support complaints may increase due to incorrect trip timing.
* The bug may be difficult to reproduce because it depends on GPS accuracy and network conditions.
* Trust issues between drivers and the platform may arise if trips auto-start unexpectedly.

