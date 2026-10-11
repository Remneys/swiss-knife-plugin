# SDK registration (React Native)

Read when `swiss-knife:push_device_list` shows no devices for the audience. Installing and configuring the SDK itself is setup-sdk; this is only the push part. Everything below is exported by `@app-swiss-knife/react-native-sdk` as `Notifications`.

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

## Identity and logout

Call `Identity.logIn(userId)` on every launch that restores a session; the device is linked to that external user id, which is what `target.user_ids` targets. The launch registration leaves `user_id` out (the server keeps the link); `Identity.logOut()`, `reset()` and `wipe()` send `user_id: null` (unlink). `Notifications.unregister()` marks the device unregistered (token null).

## Notes

- Images (`image_url`) on iOS need a Notification Service Extension target, a manual Xcode step described in the SDK README.
- Android needs the `google-services.json` of the same Firebase project as the saved service account.
