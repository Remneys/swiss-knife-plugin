---
name: setup-native
description: The native-stack half of the Swiss-Knife setup wizard — the recipes that vendor the iOS SDK (a local Swift package, SwissKnifeiOS) and the Android SDK (Kotlin source, com.appswissknife) out of the public npm package, plus the fallback for web / unrecognized stacks. Loaded by /swiss-knife:setup-sdk when Step 1 detects a native iOS (Xcode project / Package.swift, no React Native) or native Android (Gradle + Kotlin, no React Native) project, or a stack it does not recognize. Covers the dependency, gitignored secret injection, the configure call and the getOfferings smoke test; identity and verification stay in the main setup skill.
allowed-tools: Read, Edit, Write, Grep, Glob, Bash(cat *), Bash(ls *)
---

# Swiss-Knife setup wizard — native stacks

This is a **continuation of `/swiss-knife:setup-sdk`**, not a standalone wizard. Get
here only from that skill's Step 1 stack detection. Steps 1–4 (workspace, app,
project token, `.mcp.json` / `.envrc`), **Step 6 (identity)**, **Step 8 (the user's
data)** and **Step 9 (verify and hand off)** all live in the main skill and all
still apply — this file replaces **Step 5 (SDK integration)** only.

If the user reached the wizard through the bare MCP prompt
(`/mcp__swiss-knife__setup`) and you cannot load this skill, say so plainly and
point at this file in the public plugin repository:
`skills/setup-native/SKILL.md` in `github.com/Remneys/swiss-knife-plugin`.

---

## Native iOS (Swift / SwiftUI / UIKit)

The iOS SDK is the Swift module **`SwissKnifeiOS`**, **iOS 15.0+**, zero external
dependencies, vendored into the app as a local Swift package (sub-step 1). (The
React Native pod declares 15.1 — the RN/Expo default deployment target — but the
native path is 15.0.) Public API:
`SwissKnife.configure(...)`, `SwissKnife.configureCore(...)`,
`SwissKnife.configurePurchases(...)`, `SwissKnife.isConfigured`,
`SwissKnife.isCoreConfigured`, and `try await SwissKnife.getOfferings()`.

`configure` is exactly `configureCore` followed by `configurePurchases`. Unlike
React Native, native **does** expose the two halves separately, so an iOS app can
bring up core at launch and add the store later:

```swift
// Core only — no StoreKit.Transaction.updates observer, no install registered.
SwissKnife.configureCore(apiKey: SwissKnifeConfig.apiKey, proxyURL: SwissKnifeConfig.proxyURL)
// …later, if and when Swiss-Knife takes over purchases:
SwissKnife.configurePurchases(apiKey: SwissKnifeConfig.apiKey, proxyURL: SwissKnifeConfig.proxyURL)
```

Use `configureCore` if the app already has its own StoreKit code or another IAP
SDK — see the Step 2 hazard. `isConfigured` reports the purchase engine and stays
`false` in a core-only app; `isCoreConfigured` is the one to check there.

Do Steps 1–4 first (workspace, app, token, `.mcp.json` + `.envrc` for the MCP
token — those are stack-agnostic and still useful in an Xcode project). The SDK
API key does **not** go in an `EXPO_PUBLIC_*` var on native — inject it as in
sub-step 2 below. Then, instead of Step 5, do the four sub-steps below — and
afterwards still run **Step 6 (identity)**, **Step 8 (the user's data)** and
**Step 9 (verify)**, which apply to every stack. The Swift identity calls are in
Step 6.

**Capability coverage on native.** The Analytics / CrashReporting / Performance /
Attribution wiring in Step 5 is a **React-Native** surface, and so is **Step 7's
analytics boundary** — skip it here rather than pretending. If the user ticked
those in Step 2, say plainly that this recipe covers `configure`, `getOfferings`
and identity only, and that the rest isn't wired here — do **not** hand-roll
`fetch` calls against the ingest endpoints to fake it.

**1. Add the dependency (vendor it from the public npm package).**

> ⚠️ **The iOS SDK is not published to SPM or CocoaPods** — there is no
> repository with a `Package.swift` to point a URL dependency at and no pod, so
> an SPM URL or a `pod` line will not resolve. (This gap is native-only: the
> React Native package _is_ on npm.) Its Swift sources ship unmodified inside the
> public npm package `@app-swiss-knife/react-native-sdk`, so any machine with npm
> can vendor them with no checkout of anything. Tell the user plainly that the
> SDK is vendored, then do this from the app's root:

```bash
mkdir -p Vendor/SwissKnifeiOS/Sources
tmp="$(mktemp -d)"
(cd "$tmp" && npm pack @app-swiss-knife/react-native-sdk --silent && tar -xzf app-swiss-knife-react-native-sdk-*.tgz)
cp -R "$tmp/package/ios/vendor/SwissKnifeiOS" Vendor/SwissKnifeiOS/Sources/SwissKnifeiOS
rm -rf "$tmp"
```

Then write `Vendor/SwissKnifeiOS/Package.swift` (the copied folder has no
manifest; this is the whole of it):

```swift
// swift-tools-version:5.9
import PackageDescription

let package = Package(
  name: "SwissKnifeiOS",
  platforms: [.iOS(.v15)],
  products: [.library(name: "SwissKnifeiOS", targets: ["SwissKnifeiOS"])],
  targets: [.target(name: "SwissKnifeiOS", path: "Sources/SwissKnifeiOS")]
)
```

The sources import Apple frameworks only (Foundation, UIKit, StoreKit, Security,
Network and the like), so there is nothing else to resolve. Commit `Vendor/SwissKnifeiOS` so the app builds on every
machine.

If the app has a `Package.swift`, add the local package and the product to the
target's `dependencies`:

```swift
.package(path: "Vendor/SwissKnifeiOS"),
```

```swift
.product(name: "SwissKnifeiOS", package: "SwissKnifeiOS"),
```

For an `.xcodeproj` with no `Package.swift`, add it in Xcode → File → Add Package
Dependencies… → Add Local… → `Vendor/SwissKnifeiOS`, and link the
`SwissKnifeiOS` library to the app target.

**2. Config / secret injection (no `EXPO_PUBLIC` on native).** The client `sdkKey`
(`sk_live_…`, embeddable) and the origin must NOT be hardcoded into committed
source. Write a small **gitignored** `SwissKnifeConfig.swift` (or an `.xcconfig`)
and read it in `configure`. **Gitignore FIRST** (same rule as `.env`), then write
it:

`.gitignore` (append if missing):

```gitignore
SwissKnifeConfig.swift
```

`SwissKnifeConfig.swift` (gitignored — holds the real values):

```swift
import Foundation

enum SwissKnifeConfig {
  // Client-embeddable SDK key (sk_live_…): designed to ship in the app binary,
  // kept out of git as hygiene. Recover it with app_reveal_key if lost.
  static let apiKey = "<sdkKey from app_create>"
  // Deployment origin (no /api/mcp). Local-dev default shown.
  static let proxyURL = "http://localhost:3000"
}
```

Also commit a `SwissKnifeConfig.example.swift` with placeholder values so teammates
know the shape.

**3. Init code (verbatim).** Call `configure` once at launch, before any other
`SwissKnife.*`. Give the user whichever matches their app:

SwiftUI (`@main`):

```swift
import SwiftUI
import SwissKnifeiOS

@main
struct MyApp: App {
  init() {
    SwissKnife.configure(
      apiKey: SwissKnifeConfig.apiKey,
      proxyURL: SwissKnifeConfig.proxyURL
    )
  }

  var body: some Scene {
    WindowGroup { ContentView() }
  }
}
```

UIKit (`AppDelegate`):

```swift
import UIKit
import SwissKnifeiOS

@main
class AppDelegate: UIResponder, UIApplicationDelegate {
  func application(
    _ application: UIApplication,
    didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
  ) -> Bool {
    SwissKnife.configure(
      apiKey: SwissKnifeConfig.apiKey,
      proxyURL: SwissKnifeConfig.proxyURL
    )
    return true
  }
}
```

**4. Verify.** `SwissKnife.isConfigured` is `true` after `configure`. Offering
smoke test (async API):

```swift
Task {
  do {
    let offerings = try await SwissKnife.getOfferings()
    print("[swiss-knife] current offering:", offerings.current?.identifier ?? "(none)")
  } catch {
    print("[swiss-knife] getOfferings failed:", error)
  }
}
```

You can't run the app here — tell the user to build + run in Xcode (or
`xcodebuild`) on a simulator/device; the `[swiss-knife]` log on launch is the green
light.

---

## Native Android (Kotlin)

The Android SDK is the Kotlin package **`com.appswissknife`**, **minSdk 24**,
vendored into the app module as source (sub-step 1). Public API:
`SwissKnife.configure(context, apiKey, proxyURL)`,
`SwissKnife.configureCore(context, apiKey, proxyURL)`,
`SwissKnife.configurePurchases(context, apiKey, proxyURL)`,
`SwissKnife.isConfigured`, `SwissKnife.isCoreConfigured`, and
`SwissKnife.getOfferings(onSuccess, onError)`.

`configure` is exactly `configureCore` then `configurePurchases`. Use
`configureCore` alone when another billing library already owns purchases — it
builds no `BillingClient` and registers no `PurchasesUpdatedListener`, so it
cannot race the incumbent (Step 2 hazard). `isConfigured` reports the purchase
engine; `isCoreConfigured` reports the core.

Do Steps 1–4 first (the `.mcp.json` + `.envrc` MCP-token config still applies). The
SDK API key does **not** go in an `EXPO_PUBLIC_*` var — inject it via `BuildConfig`
as in sub-step 2. Then, instead of Step 5, do the four sub-steps below — and
afterwards still run **Step 6 (identity)**, **Step 8 (the user's data)** and
**Step 9 (verify)**, which apply to every stack. The Kotlin identity calls are in
Step 6.

**Capability coverage on native.** The Analytics / CrashReporting / Performance /
Attribution wiring in Step 5 is a **React-Native** surface, and so is **Step 7's
analytics boundary** — skip it here rather than pretending. If the user ticked
those in Step 2, say plainly that this recipe covers `configure`, `getOfferings`
and identity only, and that the rest isn't wired here — do **not** hand-roll
`fetch` calls against the ingest endpoints to fake it.

**1. Add the dependency (vendor it from the public npm package).**

> ⚠️ **The Android SDK is not published to Maven Central or JitPack**, so
> `implementation("com.appswissknife:swiss-knife-android:…")` does not resolve.
> (Native-only gap: the React Native package _is_ on npm.) Its Kotlin sources
> ship unmodified inside the public npm package
> `@app-swiss-knife/react-native-sdk` (`android/vendor/java` and
> `android/vendor/res`), so any machine with npm can vendor them with no checkout
> of anything. Tell the user plainly that the SDK is vendored, then copy them into
> the app module (`app` below; use the real module name):

```bash
tmp="$(mktemp -d)"
(cd "$tmp" && npm pack @app-swiss-knife/react-native-sdk --silent && tar -xzf app-swiss-knife-react-native-sdk-*.tgz)
mkdir -p app/src/main/java app/src/main/res/xml
cp -R "$tmp/package/android/vendor/java/." app/src/main/java/
cp "$tmp/package/android/vendor/res/xml/swissknife_data_extraction_rules.xml" app/src/main/res/xml/
rm -rf "$tmp"
```

The classes keep their `com.appswissknife.*` packages, so `import
com.appswissknife.SwissKnife` works as shown below. The sources need these
dependencies in the module's `build.gradle` (the same list the SDK's own build
uses; all resolve from `google()` and `mavenCentral()`), compileSdk 35 and Java 17
(`compileOptions` and Kotlin `jvmTarget`):

```groovy
dependencies {
  implementation "androidx.annotation:annotation:1.8.2"
  implementation "androidx.security:security-crypto:1.1.0-alpha06"
  implementation "androidx.work:work-runtime-ktx:2.9.0"
  implementation "androidx.lifecycle:lifecycle-process:2.5.1"
  implementation "com.android.billingclient:billing:8.0.0"
  // The SDK ships two optional OkHttp interceptors, so OkHttp must be on the
  // compile classpath. compileOnly: nothing is added to the app at runtime.
  compileOnly "com.squareup.okhttp3:okhttp:4.12.0"
}
```

And in `AndroidManifest.xml`: the Play Billing permission, plus the backup rule
that keeps the SDK's device id out of cloud backup and device transfer.

```xml
<uses-permission android:name="com.android.vending.BILLING" />

<application
    android:dataExtractionRules="@xml/swissknife_data_extraction_rules"
    ... >
```

If the app already sets `android:dataExtractionRules`, do not add a second one:
copy the two `<exclude domain="sharedpref" path="swissknife_device.xml" />` lines
from the copied rules file into the app's own rules (inside both
`<cloud-backup>` and `<device-transfer>`), and delete the copied file.

**2. Config / secret injection (via `BuildConfig`, not committed source).** Put the
`sdkKey` + origin in a **gitignored** `secrets.properties` and expose them through
generated `BuildConfig` fields — never hardcode into committed Kotlin. **Gitignore
FIRST:**

`.gitignore` (append if missing):

```gitignore
secrets.properties
```

`secrets.properties` (gitignored — holds the real values):

```properties
SWISSKNIFE_API_KEY=<sdkKey from app_create>
SWISSKNIFE_API_URL=http://localhost:3000
```

In the module `build.gradle` (Groovy), read them into `BuildConfig`:

```groovy
def secrets = new Properties()
def secretsFile = rootProject.file("secrets.properties")
if (secretsFile.exists()) { secrets.load(secretsFile.newDataInputStream()) }

android {
    buildFeatures { buildConfig true }
    defaultConfig {
        buildConfigField "String", "SWISSKNIFE_API_KEY", "\"${secrets.getProperty("SWISSKNIFE_API_KEY", "")}\""
        buildConfigField "String", "SWISSKNIFE_API_URL", "\"${secrets.getProperty("SWISSKNIFE_API_URL", "http://localhost:3000")}\""
    }
}
```

Commit a `secrets.properties.example` with placeholder values.

**3. Init code (verbatim).** Configure once in a custom `Application.onCreate()`
(the recommended single call site — it runs before any Activity):

```kotlin
import android.app.Application
import com.appswissknife.SwissKnife

class MyApp : Application() {
  override fun onCreate() {
    super.onCreate()
    SwissKnife.configure(
      context = this,
      apiKey = BuildConfig.SWISSKNIFE_API_KEY,
      proxyURL = BuildConfig.SWISSKNIFE_API_URL,
    )
  }
}
```

Register the Application class in `AndroidManifest.xml` on the `<application>` tag:

```xml
<application
    android:name=".MyApp"
    ... >
```

**4. Verify.** `SwissKnife.isConfigured` is `true` after `configure`. Offering
smoke test (callback API):

```kotlin
SwissKnife.getOfferings(
  onSuccess = { offerings ->
    Log.i("swiss-knife", "current offering: ${offerings.current?.identifier ?: "(none)"}")
  },
  onError = { error -> Log.w("swiss-knife", "getOfferings failed: $error") },
)
```

You can't run the app here — tell the user to build + run in Android Studio (or
`./gradlew installDebug`) on an emulator/device; the `swiss-knife` log on launch is
the green light.

---

## Native dependency install — honest framing (applies to BOTH recipes above)

This caveat is **native-only**. The React Native package installs from npm like any
other dependency; the two standalone native SDKs are not published to SPM,
CocoaPods, Maven Central or JitPack. The dependency works today only by
vendoring: the sources are copied out of the public npm package into the app (a
local Swift package, or Kotlin source in the app module), exactly as above, and
they do not update by themselves, so re-run the copy to take a newer SDK.
Everything else the wizard does is unaffected: backend provisioning, the config
files, and the exact `configure` + smoke-test wiring above. Say so once, calmly,
and finish the wiring.

---

## Other stacks (web / unknown)

If it isn't Expo, React Native, native iOS, or native Android (e.g. a web app or an
unrecognized stack), provisioning still works — the control-plane tools are
stack-agnostic. Do Steps 1–4 (workspace, app, token, `.mcp.json`), hand the user
their `sdkKey` + origin, seed a starter offering if they asked, and explain the
two-secrets model. Then point them at the relevant platform SDK and say plainly that
automated client wiring for this stack isn't available yet — you've done the backend
and the config; they finish the integration with the SDK docs.

Keep the same crisp, confident tone throughout. You're setting up an agent-first
product — the setup itself should feel like the best part.
