
![humnail.png](https://github.com/horlengg/storage_repo/blob/dev/implement-transaction-voice-alert-ios.webp?raw=true)


<br>

# Implementing Voice Alerts for New Transactions

Many people in shops and markets never look at their phone when a payment arrives. They listen for it. Popular QR payment apps in Southeast Asia made this a habit: the phone announces "received 5 dollars" out loud, so a seller can keep serving customers.

iOS has no built-in text-to-speech for push notifications. A notification can only play a single sound file that ships with the app, or that the app places in a location the system can read. In this post I'll show how I built a **spoken amount notification** in a native iOS banking app, using:

- a **Notification Service Extension** that intercepts the push before it is shown,
- an **App Group** to share settings and files between the app and the extension,
- pre-recorded **voice clips** (one per word or number),
- a small **audio composer** that stitches those clips into one smooth sound.

By the end you'll have the full setup in Xcode and the code to build it.

---

<br>
<br>

## Demo

<br>

<video controls>
  <source src="https://github.com/horlengg/storage_repo/raw/refs/heads/dev/VoiceAlertDemo.mp4" type="video/mp4">
</video>

<br>
<br>

## 1. How it works

The server sends a normal push notification with two extra fields, the amount and the currency. On the device, the flow is:

```
APNs push (mutable-content: 1)
        │
        ▼
NotificationService (extension process)
        │  1. read settings from App Group (enabled? voice? language?)
        │  2. SpeechSequence: "12.50 USD" → ["received","10","2","dollars","and","50","cents"]
        │  3. load each clip from the asset catalog
        │  4. AudioComposer: trim, match loudness, crossfade, normalize → merged.caf
        │  5. copy merged.caf → <App Group>/Library/Sounds/
        ▼
UNNotificationSound(named: "merged-<uuid>.caf")
        │
        ▼
System shows the notification and plays the custom sound
```

The important trick is step 5. When a notification is delivered, the system looks for custom sounds in the app's **Library/Sounds** folder. For extensions, it also looks in the **App Group container's** **Library/Sounds** folder. That is what lets a sound generated at runtime, inside the extension, be played as the notification sound.

### The push payload

The extension only runs if the payload has **"mutable-content": 1**. The amount and currency travel as custom keys:

```json
{
  "aps": {
    "alert": {
      "title": "Payment received",
      "body": "You received a payment"
    },
    "mutable-content": 1,
    "sound": "default"
  },
  "trxAmount": "12.50",
  "trxCurrency": "USD"
}
```

If anything fails inside the extension, the user still gets this normal notification with the default sound. This fallback matters for a banking app: a failed audio step must never mean a missing notification.

---

<br>

## 2. Xcode setup

### 2.1 Create the App Group

The main app and the extension run in **separate processes with separate sandboxes**. They can't read each other's files or **UserDefaults**. An App Group gives both a shared container.

**In the Apple Developer portal**

1. Go to **Certificates, Identifiers & Profiles → Identifiers**.
2. Choose **App Groups** from the dropdown and click **+**.
3. Register a group ID, for example **group.com.yourcompany.yourapp**.
4. Open your app's App ID, enable **App Groups**, and select the group you just created.

**In Xcode (main app target)**

1. Select your project, then the **app target**.
2. Open **Signing & Capabilities**, click **+ Capability**, and add **App Groups**.
3. Tick **group.com.yourcompany.yourapp**. (Xcode can also create it for you if automatic signing is on.)

You'll repeat the last two steps for the extension target after you create it.

<br>

### 2.2 Create the Notification Service Extension

1. In Xcode, choose **File → New → Target…**
2. Select **Notification Service Extension** (iOS section) and click **Next**.
3. Give it a name such as **NotificationService**. The bundle ID will be **com.yourcompany.yourapp.NotificationService**.
4. Click **Finish**. When Xcode asks *"Activate scheme?"*, choose **Cancel** or **Activate**. Either is fine, but activating switches your scheme to the extension.
5. Set the extension's **Minimum Deployment** to match your app (or lower). A higher deployment target on the extension can stop it from loading.
6. Select the **extension target → Signing & Capabilities**, add **App Groups**, and tick the same group as the main app.

Xcode generates a **NotificationService.swift** with a **UNNotificationServiceExtension** subclass. You'll replace its body in the next sections.

<br>

### 2.3 Share code and assets with the extension

The extension is its own target, so it only sees files you explicitly give it.

- **Swift files** used by both the app and the extension (for example your app group helper and the speech classes) need **Target Membership** ticked for both targets. Select the file, open the **File Inspector**, and tick both targets.
- **Voice clips** should live in an **Asset Catalog** as **Data Sets** (right-click in the catalog → **New Data Set**, then drag in the .mp3). Make sure the catalog has target membership for the extension. Assets are then loaded with NSDataAsset(name:bundle:).

Organize clip names by language and voice, like this:

```
Sounds.xcassets/
├── en/
│   └── female/
│       ├── received      (data set: received.mp3)
│       ├── 1 … 20
│       ├── 30, 40 … 90
│       ├── hundred
│       ├── thousand
│       ├── million
│       ├── and
│       ├── dollars, dollar
│       └── cents, cent
└── km/
    └── female/
        └── …
```

Data Sets can live in folders with **"Provides Namespace"** enabled, which lets you refer to them as **en/female/received**.

### 2.4 Enable push and test it

- Add the **Push Notifications** capability to the main app target.
- To test without a server, create a .apns file and drag it onto the simulator:

```json
{
  "Simulator Target Bundle": "com.yourcompany.yourapp",
  "aps": {
    "alert": { "title": "Payment received", "body": "..." },
    "mutable-content": 1
  },
  "trxAmount": "12.50",
  "trxCurrency": "USD"
}
```

- To debug the extension, run the **extension scheme** and pick your app as the host when Xcode asks. Breakpoints in **didReceive** will hit when the push arrives. Alternatively, use **Debug → Attach to Process by PID or Name…** and attach to **NotificationService** before sending the push.

---

<br>

## 3. Sharing settings through the App Group

The user chooses the voice, the language, and whether the feature is on inside the main app. The extension needs to read those choices. **UserDefaults(suiteName:)** with the group ID does exactly this.

```swift
import Foundation

final class AppGroupManager {
    static let shared = AppGroupManager()
    static let groupID = "group.com.yourcompany.yourapp"

    private let defaults = UserDefaults(suiteName: AppGroupManager.groupID)

    static var containerURL: URL? {
        FileManager.default.containerURL(forSecurityApplicationGroupIdentifier: groupID)
    }

    func getEnablePaysound() -> Bool { defaults?.bool(forKey: "enablePaysound") ?? false }
    func setEnablePaysound(_ value: Bool) { defaults?.set(value, forKey: "enablePaysound") }

    func getSpeechVoice() -> String { defaults?.string(forKey: "speechVoice") ?? "Sreymom" }
    func getSpeechLanguage() -> String { defaults?.string(forKey: "speechLanguage") ?? "km-KH" }
}
```

The extension calls **getEnablePaysound()** first. If the feature is off, it hands the original content back and returns immediately, so users who don't want spoken alerts pay no cost.

---

<br>

## 4. Turning an amount into a list of clips

Recording a clip for every possible amount is impossible, so we record a small vocabulary and build every amount from it. **SpeechSequence** converts a string like **"1,250.50"** into an ordered list of clip names.

<br>

### English

English needs a small vocabulary: the numbers up to 20, the tens, and the words **hundred**, **thousand**, **million**, **and**, plus currency words. Here's the core of the spelling logic:

```swift
private func spell(_ n: Int) -> [String] {
    guard n > 0 else { return [] }

    if n <= 20 { return [small[n]] }                       // "7" → ["7"]
    if n < 100 {
        let t = n / 10 * 10, o = n % 10
        return [String(t)] + (o > 0 ? [small[o]] : [])     // 42 → ["40","2"]
    }
    if n < 1_000 {
        let rest = n % 100
        return [small[n / 100], "hundred"]
             + (rest > 0 ? ["and"] + spell(rest) : [])     // 250 → ["2","hundred","and","50"]
    }

    let (unit, name) = n >= 1_000_000 ? (1_000_000, "million") : (1_000, "thousand")
    let rest = n % unit
    var out = spell(n / unit) + [name]
    if rest > 0 {
        out += (rest < 100 ? ["and"] : []) + spell(rest)
    }
    return out
}
```

Money needs care with decimals. Parsing with **Decimal** and rounding to whole cents avoids floating point surprises such as **0.1 + 0.2**:

```swift
var rounded = value * 100
var result = Decimal()
NSDecimalRound(&result, &rounded, 0, .plain)
let total = NSDecimalNumber(decimal: result).intValue
let dollars = total / 100
let cents = total % 100
```

Then the clips are assembled: **12.50 USD** becomes **["10", "2", "dollars", "and", "50", "cents"]**, and **1 USD** uses the singular **dollar**.

<br>

### Khmer

Khmer number words are built by place value, so the implementation decomposes the number into round parts (**100000**, **20000**, **5000**, **300**, **40**, **2**) and each part has its own clip. Some ranges use direct clips instead of a "million" word. This keeps the recorded vocabulary compact, and the result sounds natural because native speakers say numbers the same way.

The currency word (**dollar**, **riel**) and the cent part follow the integer part. Single digit decimals are padded (**.5** means 50 cents), which is an easy bug to miss.

> **Tip:** Keep the vocabulary generator separate from the audio code. **SpeechSequence** returns only strings, so you can unit test it with no audio at all: **XCTAssertEqual(seq("12.50"), ["10","2","dollars","and","50","cents"])**.

---

<br>

## 5. Stitching the clips: **AudioComposer**

If you simply concatenate the clips, the result sounds robotic and clicky. The problems are:

- each clip has leading and trailing silence, so gaps are uneven,
- clips are recorded at slightly different volumes,
- hard cuts between words produce audible pops.

**AudioComposer** fixes these in a pipeline:

1. **Decode to mono float.** Every clip is converted to the same sample rate and format with **AVAudioConverter**, so they can be mixed.
2. **Optional time stretch.** For English, clips after the first are sped up (1.2×) with **AVAudioUnitTimePitch** in *offline manual rendering* mode. This keeps the pitch and shortens the announcement.
3. **Trim silence.** Samples below −40 dB relative to the peak are removed at both ends, with about 10 ms of padding.
4. **Match loudness.** The RMS of the next clip's head is scaled to match the previous clip's tail (the gain is clamped between 0.5 and 2.0 to avoid extreme jumps).
5. **Pitch-synchronous crossfade.** The tail of the current audio and the head of the next clip are blended over about three pitch periods. The pitch period is estimated with autocorrelation (**vDSP_dotpr**), and Hann-windowed periods are overlap-added.
6. **Normalize.** The final peak is scaled to 0.85 so the notification is loud enough.
7. **Write a .caf** as 16-bit linear PCM.

The loudness matching step is short and effective:

```swift
private func matchLoudness(tail: [Float], head: inout [Float], windowLen: Int = 2048) {
    guard tail.count >= windowLen, head.count >= windowLen else { return }
    var tailRMS: Float = 0, headRMS: Float = 0
    vDSP_rmsqv(Array(tail.suffix(windowLen)), 1, &tailRMS, vDSP_Length(windowLen))
    vDSP_rmsqv(Array(head.prefix(windowLen)), 1, &headRMS, vDSP_Length(windowLen))
    guard headRMS > 0.0001 else { return }
    var gain = min(max(tailRMS / headRMS, 0.5), 2.0)
    vDSP_vsmul(head, 1, &gain, &head, 1, vDSP_Length(head.count))
}
```

And the join loop, which is the heart of the composer:

```swift
for clip in clips {
    var samples = trimSilence(clip)
    guard !samples.isEmpty else { continue }

    if allSamples.isEmpty { allSamples = samples; continue }

    matchLoudness(tail: allSamples, head: &samples)

    let period = max(1, estimatePitchPeriod(samples, sampleRate: sampleRate))
    let fadeLen = min(period * 3, min(allSamples.count, samples.count) / 2)
    guard fadeLen > 0 else { allSamples.append(contentsOf: samples); continue }

    let crossfaded = psolaCrossfadeJoin(
        tail: Array(allSamples.suffix(fadeLen)),
        head: Array(samples.prefix(fadeLen)),
        sampleRate: sampleRate
    )
    allSamples.removeLast(fadeLen)
    allSamples.append(contentsOf: crossfaded)
    allSamples.append(contentsOf: samples.suffix(from: fadeLen))
}
```

A note on clamping: a bad pitch estimate can produce a huge or zero crossfade length. Clamping **fadeLen** to half of the shorter signal keeps the output sane even with a noisy clip.

---

<br>

## 6. The notification service extension

Here is the extension, trimmed to the important parts:

```swift
import UserNotifications
import os

final class NotificationService: UNNotificationServiceExtension {

    private enum Config {
        static let soundFilePrefix = "merged-"
        static let soundMaxAge: TimeInterval = 24 * 60 * 60
    }

    private enum SpeechError: Error { case invalidPayload, missingClip(String), noAppGroup }

    private let log = Logger(subsystem: Bundle.main.bundleIdentifier ?? "NotificationService",
                             category: "spoken-amount")
    private var contentHandler: ((UNNotificationContent) -> Void)?
    private var bestAttemptContent: UNMutableNotificationContent?

    override func didReceive(_ request: UNNotificationRequest,
                             withContentHandler contentHandler: @escaping (UNNotificationContent) -> Void) {

        // 1. Feature switch from the App Group
        guard AppGroupManager.shared.getEnablePaysound() else {
            contentHandler(request.content)
            return
        }

        self.contentHandler = contentHandler
        guard let content = request.content.mutableCopy() as? UNMutableNotificationContent else {
            contentHandler(request.content)
            return
        }
        bestAttemptContent = content

        // 2. Best effort: any failure keeps the normal notification
        do {
            try applyPaysoundAudio(to: content, userInfo: request.content.userInfo)
        } catch {
            log.error("Spoken amount skipped: \(String(describing: error), privacy: .public)")
        }
        contentHandler(content)
    }

    // 3. The system is about to kill us: deliver what we have
    override func serviceExtensionTimeWillExpire() {
        if let contentHandler, let bestAttemptContent {
            contentHandler(bestAttemptContent)
        }
    }
}
```

<br>

### Building and installing the sound

```swift
private func applyPaysoundAudio(to content: UNMutableNotificationContent,
                               userInfo: [AnyHashable: Any]) throws {

    guard let amount = userInfo["trxAmount"] as? String, !amount.isEmpty,
          let currencyCode = userInfo["trxCurrency"] as? String,
          let currency = SpeechCurrency(rawValue: currencyCode),
          let language = speechLanguage
    else { throw SpeechError.invalidPayload }

    let tokens = ["received"] + SpeechSequence(amount: amount,
                                               currency: currency,
                                               language: language).serializeClips()

    // Work in a unique temp folder and always clean up
    let fm = FileManager.default
    let workDir = fm.temporaryDirectory.appendingPathComponent(UUID().uuidString, isDirectory: true)
    try fm.createDirectory(at: workDir, withIntermediateDirectories: true)
    defer { try? fm.removeItem(at: workDir) }

    let clips  = try tokens.map { try writeClip(token: $0, to: workDir) }
    let merged = workDir.appendingPathComponent("merged.caf")
    try AudioComposer.compose(urls: clips, outputURL: merged,
                              speed: language == .english ? 1.2 : nil)

    let soundName = "\(Config.soundFilePrefix)\(UUID().uuidString).caf"
    try installSound(from: merged, named: soundName)
    content.sound = UNNotificationSound(named: UNNotificationSoundName(soundName))
}
```

`AVAudioFile` reads from a file URL, so each `NSDataAsset` is first written to the temp folder:

```swift
private func writeClip(token: String, to directory: URL) throws -> URL {
    let assetName = "\(voicePrefixPath)/\(token)"          // e.g. "en/female/10"
    guard let asset = NSDataAsset(name: assetName, bundle: Bundle(for: Self.self)) else {
        throw SpeechError.missingClip(token)
    }
    let url = directory
        .appendingPathComponent(assetName.replacingOccurrences(of: "/", with: "_"))
        .appendingPathExtension("mp3")
    try asset.data.write(to: url, options: .atomic)
    return url
}
```

<br>

### Installing into Library/Sounds and cleaning up

This is the part that makes the custom sound playable:

```swift
private func installSound(from source: URL, named name: String) throws {
    guard let container = AppGroupManager.containerURL else { throw SpeechError.noAppGroup }
    let fm = FileManager.default
    let soundsDir = container.appendingPathComponent("Library/Sounds", isDirectory: true)
    try fm.createDirectory(at: soundsDir, withIntermediateDirectories: true)
    pruneOldSounds(in: soundsDir)
    try fm.copyItem(at: source, to: soundsDir.appendingPathComponent(name))
}
```

Each notification writes a new uniquely named file, so nothing overwrites a sound that is still playing, but nothing removes old files either. **pruneOldSounds** deletes anything with our prefix that is older than 24 hours:

```swift
private func pruneOldSounds(in directory: URL) {
    let fm = FileManager.default
    let cutoff = Date().addingTimeInterval(-Config.soundMaxAge)
    let files = (try? fm.contentsOfDirectory(
        at: directory, includingPropertiesForKeys: [.contentModificationDateKey])) ?? []

    for file in files where file.lastPathComponent.hasPrefix(Config.soundFilePrefix) {
        let modified = try? file.resourceValues(forKeys: [.contentModificationDateKey])
            .contentModificationDate
        if let modified, modified < cutoff { try? fm.removeItem(at: file) }
    }
}
```

The prefix check is a safety net: the cleanup can never delete a file it didn't create.



<br>

## 📦 Full Source Code

You can find the complete source code and example implementation on GitHub:  [Link](https://github.com/horlengg/ImplTransactionVoiceAlert)

<br><br>

<i>Thank you guys for reading this blog!</i>

<br><br><br><br><br>