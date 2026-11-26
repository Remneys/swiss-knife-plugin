# iOS purchases: account token, sync, eligibility, events

Read at sub-step 6 of the native iOS recipe, when the app sells anything or has a Restore Purchases button.

These belong to the purchase engine (`SwissKnife.configure` or `SwissKnife.configurePurchases`). An app that configured only the core gets none of them, and nothing about its own StoreKit code changes. If the compiler does not find one of them, the SDK copy is older than it: take the newest tag of the Swift package (or vendor again).

## Contents

- The account token on purchases
- Restore versus silent sync
- Trial and intro-offer eligibility
- Transaction events
- Test it

## The account token on purchases

Apple repeats a purchase's `appAccountToken`, signed, in every transaction and App Store Server Notification of it, so a renewal or a refund lands on the right user even when the notification arrives before the app does. By default the SDK sends the signed-in app user id when it is a UUID, and nothing else is needed. If the app's user ids are not UUIDs, pass a UUID the app keeps per user:

```swift
let result = try await SwissKnife.purchasePackage(package, appAccountToken: userToken)
// purchaseStoreProduct(_:appAccountToken:) takes it the same way
```

`SwissKnife.configure(..., purchases: .init(automaticAppAccountToken: false))` turns the default off.

## Restore versus silent sync

- `SwissKnife.restorePurchases()` belongs behind the app's Restore Purchases button: it asks the App Store to sync (the Apple ID sheet may appear), posts everything and applies the app's transfer setting, so it can move a purchase from another account of the app to this one.
- `SwissKnife.syncCurrentEntitlements()` posts only what the backend has not accepted for the current user, shows no sheet, sends nothing when nothing is new, and never moves a purchase between users.

The SDK already runs the sync after `configure` and after every `Identity.logIn` (`purchases: .init(syncEntitlementsAutomatically: false)` turns that off). Call it yourself only where the app needs fresh entitlements, such as on returning to the paywall. Never wire it to the Restore button.

## Trial and intro-offer eligibility

Before the paywall shows free-trial or intro-price wording:

```swift
let eligibility = await SwissKnife.checkTrialOrIntroDiscountEligibility(
  productIdentifiers: ["pro_yearly", "pro_monthly"]
)
// per product: .eligible, .ineligible, .noIntroOfferExists or .unknown
```

Show the trial wording only for `.eligible`; `.unknown` (StoreKit could not say) promises nothing. `checkTrialOrIntroDiscountEligibility(subscriptionGroupIdentifier:)` answers a whole subscription group with `.eligible` or `.ineligible`. Neither needs `configure` or shows UI.

## Transaction events

For the app's own reaction to a purchase, renewal, restore, sync or refund (a thank-you screen, its own analytics). Access still comes from entitlements (`SwissKnife.getCustomerInfo()`, `SwissKnife.customerInfoStream`), not from events.

```swift
let observer = SwissKnife.addTransactionObserver { event in
  if event.kind == .sync && event.ownedByAnotherUser {
    // This Apple ID's purchase belongs to another account of the app:
    // offer Restore Purchases to move it here.
    return
  }
  guard event.serverConfirmed else { return } // retried; a later event follows
  // event.kind: .purchase, .renewal, .restore, .sync, .revocation
}
// SwissKnife.removeTransactionObserver(observer) when done,
// or: for await event in SwissKnife.transactionEvents { ... }
```

An event fires once the backend answered for the transaction. `serverConfirmed == false` means it could not be reached or refused it; the SDK retries, so a later event for the same transaction can follow: de-duplicate on `event.transactionIdentifier`. Observers run on a background queue: move to the main actor before touching UI.

## Test it

- To try purchases in the simulator without App Store Connect, `swiss-knife:storekit_config_export` builds an Xcode `.storekit` file from the app's catalogue (the monetization skill, Step 12).
- After a test purchase, `swiss-knife:subscription_get` with the event's `transactionIdentifier` as `transaction_id` shows what the backend recorded, and `swiss-knife:subscription_transactions` its history (`swiss-knife:purchase_get` for a one-time purchase).
