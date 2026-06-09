---
name: building-ios-skill
description: Use when building, testing, or deploying a native iOS app on Juan's Mac mini 2018 (Intel, macOS 15.7.7) targeting his iPhone 14 (iOS 26.5). Covers Xcode setup, project scaffold, simulator workflow, device deployment, Swift 6 gotchas, iOS 26 Shortcuts/App Intents, URL schemes, and common errors.
---

# Building iOS Apps on Juan's Mac mini (Intel) + iPhone 14

## 1. Hardware Reality — Read This First

| Machine | Spec |
|---|---|
| Mac | Mac mini 2018, Intel i7-8700B, macOS 15.7.7 Sequoia |
| iPhone | iPhone 14 (iPhone14,7), iOS 26.5, UDID `00008110-000A3C600211401E` |

**Critical constraint: Xcode 26.x is ARM64-only.** All versions — even ones claiming "Sequoia support" — are Apple Silicon binaries and will crash immediately with `Bad CPU type in executable` on this Intel Mac. The ceiling is **Xcode 16.4**.

**`xcodes` tool pitfall:** `xcodes install` always selects the Apple Silicon build by default. To force x86_64:
1. Find the cached list at `~/.xcodes/available-xcodes.json` (or wherever xcodes caches it).
2. Remove the Apple Silicon entry for the target version (the one with `arm64` in its URL/filename).
3. Re-run `xcodes install 16.4`.

---

## 2. Xcode Installations

| Version | Path | Use For |
|---|---|---|
| 16.2 | `~/Downloads/Xcode.app` | Simulator builds (faster, already initialized) |
| 16.4 | `~/Downloads/Xcode-16.4.0.app` | Device builds (required for iOS 26.5 via CoreDevice) |

### First-Time Initialization (REQUIRED after install, once per Xcode)

Without this step you get `DVTDownloads Symbol not found` — a framework mismatch from stale system references.

```bash
DEVELOPER_DIR=/Users/juan/Downloads/Xcode-16.4.0.app/Contents/Developer \
  sudo xcodebuild -runFirstLaunch
```

Then **open the Xcode GUI at least once** and sign in with `juandiego085@gmail.com` (Settings → Accounts). The CLI cannot authenticate on its own — it reads the keychain entry Xcode creates on first GUI sign-in.

### Switching Active Xcode

```bash
# For device work
sudo xcode-select -s /Users/juan/Downloads/Xcode-16.4.0.app/Contents/Developer

# For simulator work
sudo xcode-select -s /Users/juan/Downloads/Xcode.app/Contents/Developer
```

Better practice: always use the `DEVELOPER_DIR=` prefix on every `xcodebuild` call instead of switching globally. That way parallel simulator/device workflows don't stomp each other.

### iOS 26.5 Device Support

iOS 26.5 support is handled automatically by CoreDevice in Xcode 16.4. Device support files auto-created at:
```
~/Library/Developer/Xcode/iOS DeviceSupport/iPhone14,7 26.5 (23F77)
```
`xcodebuild -downloadPlatform iOS` downloads simulator runtimes — **not** needed for device deploy.

---

## 3. Project Scaffold with xcodegen

```bash
brew install xcodegen
```

Place `project.yml` in the project root (the folder that will contain the `.xcodeproj`), not inside the app source folder.

### Canonical project.yml

```yaml
name: AppName
options:
  bundleIdPrefix: com.juan
  deploymentTarget:
    iOS: "17.0"
  xcodeVersion: "16"
  groupSortPosition: top
  createIntermediateGroups: true

settings:
  base:
    SWIFT_VERSION: 6.0
    IPHONEOS_DEPLOYMENT_TARGET: 17.0
    DEVELOPMENT_TEAM: ""          # leave blank — Xcode automatic signing fills this
    CODE_SIGN_STYLE: Automatic
    INFOPLIST_FILE: AppName/Resources/Info.plist
    PRODUCT_BUNDLE_IDENTIFIER: com.juan.appname
    ASSETCATALOG_COMPILER_APPICON_NAME: AppIcon
    ASSETCATALOG_COMPILER_GLOBAL_ACCENT_COLOR_NAME: AccentColor
    TARGETED_DEVICE_FAMILY: "1"  # iPhone only

targets:
  AppName:
    type: application
    platform: iOS
    sources:
      - path: AppName
        excludes:
          - Resources/Info.plist  # exclude plist from source scan; add as resource
    resources:
      - path: AppName/Resources
    dependencies: []
    settings:
      base:
        INFOPLIST_FILE: AppName/Resources/Info.plist
```

After any change to `project.yml` or after adding/removing `.swift` files:
```bash
cd <project_root> && xcodegen generate
```

xcodegen auto-picks up all `.swift` files in the `sources` path. Never hand-edit `.pbxproj`.

---

## 4. Info.plist — Required Keys

Missing any of these causes install failure on simulator or device.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>CFBundleExecutable</key>
    <string>$(EXECUTABLE_NAME)</string>       <!-- REQUIRED: missing = "Missing bundle ID" on simulator -->
    <key>CFBundlePackageType</key>
    <string>APPL</string>
    <key>CFBundleInfoDictionaryVersion</key>
    <string>6.0</string>
    <key>CFBundleIdentifier</key>
    <string>$(PRODUCT_BUNDLE_IDENTIFIER)</string>
    <key>CFBundleVersion</key>
    <string>1</string>                        <!-- REQUIRED: missing = "invalid CFBundleVersion" -->
    <key>CFBundleShortVersionString</key>
    <string>1.0</string>
    <key>CFBundleName</key>
    <string>AppName</string>
    <key>CFBundleDisplayName</key>
    <string>AppName</string>
    <key>UILaunchScreen</key>
    <dict/>
    <key>UISupportedInterfaceOrientations</key>
    <array>
        <string>UIInterfaceOrientationPortrait</string>
    </array>
</dict>
</plist>
```

Add keys for URL schemes, location, background modes as needed (see §10).

---

## 5. Build Commands

Always prefix `DEVELOPER_DIR=` — never rely on `xcode-select` global state.

### Simulator (Xcode 16.2)

```bash
DEVELOPER_DIR=/Users/juan/Downloads/Xcode.app/Contents/Developer \
  xcrun xcodebuild \
    -scheme AppName \
    -destination 'platform=iOS Simulator,name=iPhone 16' \
    -quiet clean build
```

### Device (Xcode 16.4)

```bash
DEVELOPER_DIR=/Users/juan/Downloads/Xcode-16.4.0.app/Contents/Developer \
  xcrun xcodebuild \
    -scheme AppName \
    -destination 'platform=iOS,id=00008110-000A3C600211401E' \
    CODE_SIGN_STYLE=Automatic \
    DEVELOPMENT_TEAM=96WN46F42L \
    -allowProvisioningUpdates \
    -allowProvisioningDeviceRegistration \
    clean build
```

**Prerequisite:** Xcode 16.4 must have been opened at least once with `juandiego085@gmail.com` signed in. CLI cannot auth Apple ID on its own.

---

## 6. Simulator Workflow

```bash
# Boot
DEVELOPER_DIR=/Users/juan/Downloads/Xcode.app/Contents/Developer \
  xcrun simctl boot "iPhone 16"
open /Users/juan/Downloads/Xcode.app/Contents/Developer/Applications/Simulator.app

# Install
APP=$(find ~/Library/Developer/Xcode/DerivedData -name 'AppName.app' \
  -path '*Debug-iphonesimulator*' | head -1)
xcrun simctl install booted "$APP"

# Launch
xcrun simctl launch booted com.juan.appname

# Fire URL scheme (for testing URL-triggered flows)
xcrun simctl openurl booted 'finance://tx?merchant=Whole+Foods&amount=6.22&card=Chase'

# Screenshot
xcrun simctl io booted screenshot /tmp/screen.png

# Uninstall (REQUIRED after SwiftData schema changes — no migration in dev)
xcrun simctl uninstall booted com.juan.appname
```

---

## 7. Physical Device Workflow

```bash
# Verify device is seen
xcrun xctrace list devices | grep iPhone
xcrun devicectl list devices

# Launch app (Xcode 16.4 required)
DEVELOPER_DIR=/Users/juan/Downloads/Xcode-16.4.0.app/Contents/Developer \
  xcrun devicectl device process launch \
    --terminate-existing \
    com.juan.appname

# Stream device logs (requires libimobiledevice)
brew install libimobiledevice
idevicesyslog -u 00008110-000A3C600211401E 2>&1 | grep -i "financeapp"
```

### Testing URL schemes on device

No `simctl openurl` equivalent for physical device. Options:
1. Open Safari on the phone, type the URL directly.
2. Use Universal Clipboard: `echo -n 'finance://tx?...' | pbcopy`, paste in Safari on phone.
3. Create a Note via AppleScript (syncs via iCloud): `osascript -e 'tell application "Notes" to make new note with properties {body:"finance://tx?..."}'`

### Screenshots from device

`idevicescreenshot` from libimobiledevice fails with `Invalid service` on iOS 26.5 + Xcode 16.4 (no matching developer disk image for iOS 26.x). Have the user take a screenshot manually or share via AirDrop.

---

## 8. Swift 6 Gotchas

These all produce compile errors under `SWIFT_VERSION: 6.0`, not just warnings:

- **`AppIntent` static properties** (`title`, `description`) must be `nonisolated(unsafe)` or the concurrency checker rejects them.
- **`CLLocationManager` in SwiftUI views** — declare as `@State`, not `let`. A bare `let` triggers "invalid reuse after initialization" on re-render.
- **SwiftData context access** — any class touching `ModelContext` needs `@MainActor`. Apply to `TransactionHandler` and `LocationResolver` when they call `context.insert/save`.
- **`@Relationship(deleteRule: .cascade)`** — add on any child model property, or deleting the parent leaves orphaned records that accumulate and are never shown.
- **Schema changes** — SwiftData has no migration in dev builds. After changing any `@Model` property name or type: uninstall the app from simulator/device before reinstalling.

---

## 9. Signing

- **Team ID:** `96WN46F42L` (personal Apple Developer account, `juandiego085@gmail.com`)
- **Style:** Automatic (Xcode manages provisioning profiles)
- Leave `DEVELOPMENT_TEAM: ""` in `project.yml` — pass it on the CLI or let Xcode GUI fill it.
- CLI flags: `CODE_SIGN_STYLE=Automatic DEVELOPMENT_TEAM=96WN46F42L -allowProvisioningUpdates -allowProvisioningDeviceRegistration`

---

## 10. URL Scheme Setup

### Info.plist addition

```xml
<key>CFBundleURLTypes</key>
<array>
    <dict>
        <key>CFBundleURLName</key>
        <string>com.juan.appname</string>
        <key>CFBundleURLSchemes</key>
        <array>
            <string>myapp</string>
        </array>
    </dict>
</array>
```

### SwiftUI entry point

```swift
@main
struct MyApp: App {
    @UIApplicationDelegateAdaptor(AppDelegate.self) var appDelegate

    var sharedModelContainer: ModelContainer = { /* SwiftData setup */ }()

    var body: some Scene {
        WindowGroup {
            ContentView()
                .preferredColorScheme(.dark)
                .onOpenURL { url in
                    appDelegate.handler.handle(url: url, context: sharedModelContainer.mainContext)
                }
        }
        .modelContainer(sharedModelContainer)
    }
}
```

Use `AppDelegate` + a handler class for URL logic, not scene delegate — cleaner separation and the `@UIApplicationDelegateAdaptor` pattern is idiomatic in SwiftUI apps.

### URL parameter decoding

Shortcuts encodes spaces as `+` in URL query strings (not `%20`). URLComponents does not decode `+` by default:

```swift
func decoded(_ s: String) -> String {
    s.replacingOccurrences(of: "+", with: " ").removingPercentEncoding ?? s
}
```

---

## 11. iOS 26 Shortcuts — Transaction Trigger

The Transaction Trigger lives under **Wallet → card selection**, not a standalone "Transaction" category. To set it up:
1. Open Shortcuts → Automation → New Automation.
2. Scroll to Wallet, pick "Transaction", select the cards to watch.
3. Set "Run Immediately" (no notification, no confirmation).

**What data the trigger provides:**
- `Shortcut Input → Merchant`: merchant name (e.g. "Whole Foods")
- `Shortcut Input → Amount`: amount string including currency symbol (e.g. "$6.22")
- `Shortcut Input → Card`: card name (e.g. "Chase Visa")
- `Shortcut Input → Date`: ISO 8601 timestamp of the transaction

**Getting GPS:** Add a "Get Current Location" action before the Open URL action. Capture lat/lng from the result and append to the URL.

**AppShortcutsProvider / AppIntents:** If you add `AppShortcutsProvider`, call `FinanceAppShortcuts.updateAppShortcutParameters()` on app launch (via `.task { }` on ContentView). The app must be opened at least once before it appears in the automation app list.

**System does not auto-push** transaction fields to `@Parameter` fields in an `AppIntent`. The user must manually map `Shortcut Input` variables to your intent parameters in the Shortcuts editor.

---

## 12. App Architecture That Works

### SwiftData + SwiftUI

```swift
// Editable detail view
struct TransactionDetail: View {
    @Bindable var transaction: Transaction  // @Bindable for @Model, not @State
    // ...
}
```

### Persistent logging (device debug)

Write to `Documents/app.log` via a background queue; cap at N lines to avoid unbounded growth:

```swift
final class Logger {
    static let shared = Logger()
    private let queue = DispatchQueue(label: "log", qos: .background)
    private let url: URL = {
        let docs = FileManager.default.urls(for: .documentDirectory, in: .userDomainMask)[0]
        return docs.appendingPathComponent("app.log")
    }()
    private let maxLines = 1000

    func log(_ msg: String) {
        queue.async {
            let line = "\(Date()) \(msg)\n"
            var existing = (try? String(contentsOf: self.url)) ?? ""
            var lines = existing.components(separatedBy: "\n")
            lines.append(line)
            if lines.count > self.maxLines { lines = Array(lines.suffix(self.maxLines)) }
            try? lines.joined(separator: "\n").write(to: self.url, atomically: true, encoding: .utf8)
        }
    }
}
```

Read log in-app with a raw text view, or `idevicesyslog` while connected.

---

## 13. Common Errors and Fixes

| Error | Cause | Fix |
|---|---|---|
| `Bad CPU type in executable` | Xcode 26.x is ARM64-only | Use Xcode 16.4 |
| `DVTDownloads: Symbol not found` | Old system framework stub from Xcode 26 install | `DEVELOPER_DIR=.../Xcode-16.4.0.app/... sudo xcodebuild -runFirstLaunch` |
| `iOS 18.5 is not installed` / `iOS 26.5 is not installed` | Xcode 16.4 never initialized | Open Xcode 16.4 GUI once, sign in, let component install complete |
| `Missing bundle ID` | No `CFBundleExecutable` in Info.plist | Add `<key>CFBundleExecutable</key><string>$(EXECUTABLE_NAME)</string>` |
| `invalid CFBundleVersion` | No `CFBundleVersion` in Info.plist | Add `<key>CFBundleVersion</key><string>1</string>` |
| `No Account for Team` | Apple ID not in Xcode 16.4 keychain | Open Xcode 16.4 → Settings → Accounts → add `juandiego085@gmail.com` |
| `xcodes installs ARM64 version` | xcodes picks Apple Silicon build | Patch `available-xcodes.json`, remove Apple Silicon entry for target version |
| SwiftData crash on launch after model change | Schema mismatch, no migration | Uninstall app from device/simulator first (`xcrun simctl uninstall booted <id>`) |
| Merchant shows `Whole+Foods` | URL `+` not decoded | `s.replacingOccurrences(of: "+", with: " ").removingPercentEncoding ?? s` |
| `Invalid service` on `idevicescreenshot` | iOS 26.5 has no matching developer disk image | Take screenshot manually on device |
| `CLLocationManager invalid reuse` | `let` manager in SwiftUI view | Change to `@State var manager = CLLocationManager()` |
| Orphaned SwiftData records accumulate | Missing `@Relationship(deleteRule: .cascade)` | Add cascade rule on child-model properties |

---

## 14. Quick-Start Checklist for a New App

- [ ] `brew install xcodegen`
- [ ] Create `project.yml` (see §3 template) in project root
- [ ] Create `AppName/Resources/Info.plist` with all required keys (see §4)
- [ ] Create `AppName/Resources/Assets.xcassets` with AppIcon + AccentColor colorset stubs
- [ ] `cd <project_root> && xcodegen generate`
- [ ] Open Xcode 16.4 GUI, sign in with `juandiego085@gmail.com` (one-time)
- [ ] `DEVELOPER_DIR=.../Xcode-16.4.0.app/... sudo xcodebuild -runFirstLaunch` (one-time)
- [ ] Simulator build: use `DEVELOPER_DIR=.../Xcode.app/...` prefix
- [ ] Device build: use `DEVELOPER_DIR=.../Xcode-16.4.0.app/...` + signing flags (see §5)
- [ ] After any SwiftData `@Model` change: uninstall app before reinstalling
