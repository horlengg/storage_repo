# Building an iOS Control Center Widget with Deep Linking

Control Center widgets, introduced in iOS 18, let users trigger quick actions right from Control Center without opening the app first. In this post, I'll walk through how I built a Control Center widget that opens my app directly into a specific screen — a QR scanner or a KHQR payment view — using `AppIntent`, `WidgetKit`, and custom URL schemes.

## The Goal

I wanted two Control Center buttons:

- **Scan QR** — opens the app straight into the scanner
- **KHQR** — opens the app straight into the KHQR screen

Tapping either button should launch the app and immediately present the right screen, even if the app is already open and showing something else.

## Architecture Overview

The implementation has three pieces working together:

1. **The Widget Extension** — defines the Control Center buttons and their `AppIntent`s
2. **Custom URL Scheme** — used to pass information from the intent back into the running app
3. **The Main App** — listens for the incoming URL and presents the correct sheet

```
Control Center Button → AppIntent.perform() → openURL(customScheme)
       → App launches/resumes → onOpenURL → present matching sheet
```

## 1. Defining the Widget Bundle

Every widget extension needs a bundle entry point. Here, the bundle registers both control widgets:

```swift
import WidgetKit
import SwiftUI

@main
struct AppControlWidgetBundle: WidgetBundle {
    var body: some Widget {
        ScanControlWidget()
        KHQRControlWidget()
    }
}
```

## 2. Creating the Control Widgets

Each `ControlWidget` uses `StaticControlConfiguration` and wraps a `ControlWidgetButton` bound to an `AppIntent`:

```swift
struct ScanControlWidget: ControlWidget {
    static let kind: String = "com.lengdev.ctrwidget.AppControlWidget.scan"

    var body: some ControlWidgetConfiguration {
        StaticControlConfiguration(kind: Self.kind) {
            ControlWidgetButton(action: OpenScanIntent()) {
                Label("Scan QR", systemImage: "qrcode.viewfinder")
            }
        }
        .displayName("Scan QR")
        .description("Open scan KHQR screen")
    }
}

struct KHQRControlWidget: ControlWidget {
    static let kind: String = "com.lengdev.ctrwidget.AppControlWidget.khqr"

    var body: some ControlWidgetConfiguration {
        StaticControlConfiguration(kind: Self.kind) {
            ControlWidgetButton(action: OpenKHQRIntent()) {
                Label("KHQR", systemImage: "qrcode")
            }
        }
        .displayName("KHQR")
        .description("Open KHQR screen")
    }
}
```

Each widget gets a unique `kind` identifier, a display name, and a description shown to the user when they're customizing Control Center.

## 3. Writing the AppIntents

Each button triggers an `AppIntent` that opens a custom URL scheme when run. `openAppWhenRun = true` tells the system to bring the app to the foreground.

```swift
@available(iOS 18.0, *)
struct OpenScanIntent: AppIntent {
    static let title: LocalizedStringResource = "Scan"
    static var openAppWhenRun = true
    static var isDiscoverable = true
    static let scanURL = URL(string: "ctrlwidgetapp://scan")!

    @Environment(\.openURL) private var openURL

    @MainActor
    func perform() async throws -> some IntentResult {
        openURL(Self.scanURL)
        return .result()
    }
}

@available(iOS 18.0, *)
struct OpenKHQRIntent: AppIntent {
    static let title: LocalizedStringResource = "KHQR"
    static var openAppWhenRun = true
    static var isDiscoverable = true
    static let scanURL = URL(string: "ctrlwidgetapp://khqr")!

    @Environment(\.openURL) private var openURL

    @MainActor
    func perform() async throws -> some IntentResult {
        openURL(Self.scanURL)
        return .result()
    }
}
```

> **Gotcha:** It's tempting to write `EnvironmentValues().openURL(...)`, but that constructs a brand-new, default environment instead of using the one your intent is actually running in — the call effectively goes nowhere. Injecting `openURL` with `@Environment(\.openURL)` is what makes it reliably reach the system.

## 4. Handling the URL in the App

Each intent opens a URL like `ctrlwidgetapp://scan` or `ctrlwidgetapp://khqr`. The host component of the URL tells the app which screen to show. An enum keeps the mapping between URL host, sheet title, and theme color in one place:

```swift
enum SheetType: String {
    case idle
    case scan
    case khqr

    var title: String {
        switch self {
        case .scan: return "Scan"
        case .khqr: return "QR Code"
        case .idle: return "Unknown"
        }
    }

    var color: Color {
        switch self {
        case .scan: return .orange
        case .khqr: return .indigo
        case .idle: return .clear
        }
    }
}
```

The app's root scene listens for incoming URLs with `.onOpenURL` and presents a sheet based on the current `SheetType`:

```swift
@main
struct ExploreControlWidgetApp: App {
    @Environment(\.scenePhase) private var scenePhase
    @State private var activeSheet: SheetType = .idle

    var body: some Scene {
        WindowGroup {
            Text("iOS Control Center Widget")
                .onOpenURL { url in
                    handleOpenURL(url)
                }
                .preferredColorScheme(.light)
                .sheet(isPresented: isSheetPresented) {
                    sheetView
                }
        }
    }

    private var isSheetPresented: Binding<Bool> {
        Binding(
            get: { activeSheet != .idle },
            set: { _ in activeSheet = .idle }
        )
    }

    private var sheetView: some View {
        VStack {
            Text(activeSheet.title)
                .foregroundStyle(.white)
        }
        .presentationBackground(activeSheet.color)
        .presentationDetents([.medium, .large])
    }

    private func handleOpenURL(_ url: URL) {
        guard let host = url.host, let sheet = SheetType(rawValue: host) else { return }

        if activeSheet == .idle {
            activeSheet = sheet
        } else {
            activeSheet = .idle
            DispatchQueue.main.asyncAfter(deadline: .now() + 0.3) {
                activeSheet = sheet
            }
        }
    }
}
```

### Why the 0.3-second delay?

If a sheet is already presented and the user taps a *different* Control Center button, SwiftUI won't reliably swap the sheet content if you just change `activeSheet` directly — the old sheet is still animating or bound. Dismissing first (`activeSheet = .idle`) and then re-presenting the new one after a short delay avoids that glitch and gives a clean transition between sheets.

## Wiring It All Together

Putting the flow together end-to-end:

1. User taps **Scan QR** in Control Center.
2. `OpenScanIntent.perform()` runs and calls `openURL(ctrlwidgetapp://scan)`.
3. iOS launches (or foregrounds) the app because `openAppWhenRun` is `true`.
4. The app's `.onOpenURL` handler receives the URL, extracts the host (`"scan"`), and maps it to `SheetType.scan`.
5. `activeSheet` updates, which drives the `sheet(isPresented:)` binding, and the scan sheet appears with an orange background.

## Setup Guide: Widget Extension + Deep Link

Before any of the code above works, two pieces of Xcode project setup need to be in place: the widget extension target itself, and the custom URL scheme that lets the extension talk back to the main app.

### Step 1 — Add a Widget Extension target

1. In Xcode, select **File → New → Target…**
2. Choose **Widget Extension** from the template list, then click **Next**.
3. Name it (e.g. `AppControlWidget`), make sure **Include Configuration App Intent** is checked if you want a configurable widget, and select your main app as the **Embed in Application** target.
4. Click **Finish**, and when prompted, **activate** the new scheme.

This creates a new target and folder with a `WidgetBundle`, a sample `Widget`, and (if selected) a `ConfigurationAppIntent` — the same boilerplate seen in `AppIntent.swift` above.

### Step 2 — Set the minimum deployment target

Control Center widgets (`ControlWidget`, `StaticControlConfiguration`) require **iOS 18.0+**. Select the widget extension target in the project navigator, go to **General → Minimum Deployments**, and set it to iOS 18.0. Guard any iOS-18-only intents with `@available(iOS 18.0, *)` as shown earlier if your main app supports older versions.

### Step 3 — Replace the template code

Delete the placeholder `Widget` struct Xcode generates and replace it with your `ControlWidget`s, `AppIntent`s, and `WidgetBundle`, following the structure from the sections above:

- `AppControlWidgetBundle.swift` — the `@main` `WidgetBundle`
- `AppControlWidgetControl.swift` — the `ControlWidget`s and their `AppIntent`s
- `AppIntent.swift` — any configuration intent (optional, delete if unused)

### Step 4 — Enable App Groups (if the widget needs to share data)

If your widget and app ever need to share state (not required for this URL-based approach, but common for other widgets), add the **App Groups** capability to both targets under **Signing & Capabilities**, and use the same group identifier in both.

### Step 5 — Register a custom URL scheme

This is what lets `openURL(ctrlwidgetapp://scan)` actually reach your app.

1. Select the **main app target** (not the widget extension) in the project navigator.
2. Go to **Info** tab → **URL Types** → click **+**.
3. Set:
   - **Identifier**: your bundle ID (e.g. `com.lengdev.ctrwidget`)
   - **URL Schemes**: `ctrlwidgetapp` (just the scheme, no `://`)
   - **Role**: `Editor`

Alternatively, add this directly to the app target's `Info.plist`:

```xml
<key>CFBundleURLTypes</key>
<array>
    <dict>
        <key>CFBundleURLName</key>
        <string>com.lengdev.ctrwidget</string>
        <key>CFBundleURLSchemes</key>
        <array>
            <string>ctrlwidgetapp</string>
        </array>
    </dict>
</array>
```

### Step 6 — Handle the URL in the app

Add `.onOpenURL` to your root scene (as shown in the `ExploreControlWidgetApp` example) and parse the `host` of the incoming URL to decide what to present. Keep the scheme-to-screen mapping in one place — the `SheetType` enum above is a good pattern — so adding a new deep link later just means adding one enum case, one intent, and one `ControlWidget`.

### Step 7 — Test the deep link before touching Control Center

You can test the URL scheme independently of the widget:

- **Simulator**: run `xcrun simctl openurl booted ctrlwidgetapp://scan`
- **Safari on device/simulator**: type `ctrlwidgetapp://scan` into the address bar
- **Notes app**: paste the link as text and tap it

If the app opens and presents the right sheet, the scheme is wired correctly and any remaining issue is isolated to the widget/intent side.

### Step 8 — Add the widget to Control Center

On a device or simulator running iOS 18+:

1. Open **Settings → Control Center**.
2. Tap **Add a Control**.
3. Find your app's controls by the `displayName` set on each `ControlWidget` (e.g. "Scan QR", "KHQR").
4. Add them, then open Control Center to test — tapping the button should launch the app and trigger the deep link.

### Common pitfalls

- **Forgetting `@available(iOS 18.0, *)`** on `AppIntent`s that use `ControlWidget`-only APIs, causing build errors on projects with a lower deployment target elsewhere.
- **Using `EnvironmentValues()` instead of `@Environment(\.openURL)`** inside the intent — covered above, but worth repeating since it's the most common silent failure.
- **Mismatched URL scheme casing or typos** between the `Info.plist` entry and the hardcoded `URL(string:)` in the intents — these must match exactly.
- **Testing only in Control Center** — since intents can be slow to reflect code changes there, always verify the URL scheme independently first (Step 7) to isolate whether the bug is in the deep link or in the widget/intent wiring.

## Wrap-up

Control Center widgets are a great case for `AppIntent` + custom URL schemes: the intent doesn't need to know anything about your app's UI, it just needs to open a URL, and the app itself decides what that URL means. Keeping the routing logic in one `SheetType` enum makes it trivial to add a third or fourth widget later — just add a new case, a new intent, and a new `ControlWidget`.

If you're building something similar, the main pitfall to watch for is using `@Environment(\.openURL)` rather than a fresh `EnvironmentValues()` instance inside your `AppIntent` — it's an easy mistake that silently breaks the whole flow.