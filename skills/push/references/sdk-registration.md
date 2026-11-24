# SDK registration (React Native)

Read when `swiss-knife:push_device_list` shows no devices for the audience. Installing and configuring the SDK itself is setup-sdk; this is only the push part. Check the installed package for its `Notifications` export. Versions before 0.5.0 lack the module. With one of those, use the documented device and event HTTP endpoints, or move the app to a version that includes the module.

## Native mode (no push library in the app)

Add the config plugin `@app-swiss-knife/react-native-sdk` to the `app.json` plugins (iOS `mode`: `development` or `production`; Android `defaultChannel`), then:

```ts
import { Notifications } from "@app-swiss-knife/react-native-sdk";

const status = await Notifications.requestPermission({ provisional: false });
if (status === "authorized" || status === "provisional") {
  await Notifications.registerForPushNotifications();
}
const sub = Notifications.addNotificationOpenedListener((open) => {});
```

The device is registered once a token arrives; opens are tracked automatically.

## Manual mode (expo-notifications or Firebase already in the app)

Use the raw device token (`getDevicePushTokenAsync()`, not the Expo push token):

```ts
await Notifications.registerDeviceToken(
  token,
  Platform.OS === "ios" ? "apns" : "fcm"
);
Notifications.trackReceived(notification);
Notifications.trackOpened(response);
```

## Verified recipients

An authenticated backend can enroll an app in server-verified registration for private notifications. Use an admin write token scoped to this app alone. Derive the user ID and session issue time from the authenticated session. Send the device body to `POST /api/sdk/push/devices` with the bearer credential, `X-App-Project-Id`, `X-Push-Registration: server-verified` and `X-Push-Session-At` (UTC ISO time). The installation generates a random v4 UUID for its private `install_id`. Keep this capability out of logs. Persist a positive integer `binding_version` on the installation and increment it before every registration or local revocation. The server refuses delayed writes with older versions. A successful response says `binding: server-verified`.

Enrollment applies to the entire project. If any active SDK-registered device remains, enrollment returns 409 and preserves the original audience. Migrate or revoke those devices before enrolling. All audiences then use verified devices, including previews and sends from the plugin. The device list reports `registration_mode: server_verified`. After local sign-out, the public SDK key can revoke only that installation using the original capability with explicit null user and token. For account logout or deletion, the backend uses the same endpoint and credential with `X-Push-Registration: revoke-user` and `{user_id, cutoff}`. Persist this intent until the server acknowledges `binding: revoked`. An older session cannot register again. A queued delivery skips an installation whose verified recipient changed. Add `require_verified_registration: true` to the REST message body to refuse an unprotected project.

## SDK-managed identity and logout

Call `Identity.logIn(userId)` on every launch that restores a session; the device is linked to that external user id, which is what `target.user_ids` targets. The launch registration leaves `user_id` out (the server keeps the link); `Identity.logOut()`, `reset()` and `wipe()` send `user_id: null` (unlink). `Notifications.unregister()` marks the device unregistered (token null).

## Notes

- Images (`image_url`) on iOS need a Notification Service Extension target, a manual Xcode step described in the SDK README.
- Android needs the `google-services.json` of the same Firebase project as the saved service account.
