# DiffViewerFeedbackSDK

Binary SwiftUI feedback SDK for development apps on iOS 17+ and macOS 14+. Capture a bug or note with a screenshot and the available interface context. Reports can stay on the device or upload privately to the paired DiffViewer server account. DiffViewer turns selected reports into agent-request Markdown.

This package contains static XCFrameworks and public API interfaces, without implementation sources.

## 1. Add the package

Add `https://github.com/santiagocento/DiffViewerFeedbackSDK.git` in Xcode and link the **DiffViewerFeedback** product to your app target. Both required binary modules are included. Alternatively, add the current binary package directory as a local package dependency.

Use SDK **0.4.0 or later** for the SDK-generated connection flow below. Version 0.3.0 supports server uploads and screenshots with account-generated pairing codes. Version 0.1.1 supports the older CloudKit text-report integration.

## 2. Enable capture at the app root

In DiffViewer, open **Server account** and copy its server URL. Replace the example HTTPS origin below with that address, replace `AppRootView` with your app's root view, and choose one stable project ID per repository.

```swift
import SwiftUI
#if DEBUG
import DiffViewerFeedback
#endif

@main
struct ExampleApp: App {
    var body: some Scene {
        WindowGroup {
            #if DEBUG
            if let serverURL = URL(string: "https://your-diffviewer-server.example") {
                DiffViewerFeedbackHost(configuration: FeedbackConfiguration(
                    projectID: "github.com/your-account/your-repository",
                    serverURL: serverURL
                ), isEnabled: true) {
                    AppRootView()
                }
            } else {
                AppRootView()
            }
            #else
            AppRootView()
            #endif
        }
    }
}
```

Keep the import and enabled host inside the consuming app's `#if DEBUG` branches. Capture defaults to disabled; `isEnabled: true` opts in. A precompiled framework cannot read its consumer's compilation flags, so the binary contains capture code. The distribution verifier checks that a consumer which references the SDK only in Debug excludes capture symbols from its Release executable.

No per-screen registration is required. The SDK records available controller/root types, navigation titles, existing accessibility identifiers, visible labels and build/device metadata before the composer opens. Text-entry contents are excluded from interface labels; screenshots contain the visible app content. Runtime context supplies search clues, not a guaranteed Swift filename or source line. An optional `revision` can be supplied from your build system.

Use the same server origin in both apps. HTTPS is supported; HTTP is accepted only for loopback hosts (`localhost`, `127.0.0.1`, `::1`). A loopback address on a physical iPhone points to that iPhone, not your Mac; use a reachable HTTPS server for device testing. Server feedback does not require CloudKit entitlements or a shared Apple Developer team.

## 3. Pair the development app

1. In the development app, open **Feedback → Connect with DiffViewer**. The SDK generates an eight-character code for its configured project.
2. Tap **Open DiffViewer to approve**. On another device, sign in to DiffViewer and enter the code under **Settings → Account → Feedback SDK**.
3. Review the app and project, then tap **Approve this device**. Return to the development app; it finishes connecting automatically. Codes expire after ten minutes.
4. Save an observation with **Upload to server** enabled, then refresh **App feedback** in DiffViewer using the same account and server.

This flow requires SDK **0.4.0 or later**, the updated DiffViewer app, and an API deployment with `/v1/feedback/connections`. It does not require workspace sync or a separate sign-in in your development app. The short code authorizes a pending device; only that device receives its credential after account approval. The app-opening link contains the short code and server origin, never the device token. Both apps must use the same server origin.

For older SDKs, expand **Connect with an older SDK** in DiffViewer, enter the SDK's project ID and create a one-time pairing code. In the SDK, expand **Already have a pairing code?** and paste it. Existing pairings keep working.

The SDK stores its device credential in the app's Keychain and persists queued reports before uploading. Failed uploads remain queued for retry. **Save Locally** keeps the report and screenshot on that device; it does not make them appear in DiffViewer on another device or automatically upload those local-only notes after pairing. **Show paired devices** in DiffViewer can revoke a device's access.

Apps that already authenticate with the DiffViewer server can instead use `FeedbackConfiguration(projectID:serverURL:accountID:credentialsProvider:)`. Pass the current account ID and a provider that creates and redeems a device pairing using the authenticated API. Pass a nil account ID when signed out and recreate the host when accounts change. This connects without a manual pairing-code form and caches credentials per account, server and project. External apps can use the SDK-generated connection flow above.

### Connection diagnostics

The composer shows the server, project, pairing progress and actionable failure details, including HTTP status and the API's error code/message. Network failures include the URL error code and hints for device reachability or HTTPS certificates. Pairing alone does not prove the server is reachable; upload results report that separately.

In Xcode or Console, filter for subsystem `com.diffviewer.feedback` and category `Connection`. Request start/completion, HTTP failures and credential failures are logged without pairing codes, device tokens or report bodies. Hostnames and error detail are private in the system log. Upload failures keep reports in the local outbox for retry.

## 4. Capture screenshots

On Mac, press **⌘⌥⇧F** and drag to select an area of the current window or presented sheet. Hold Shift for a square or Escape to cancel. The composer opens above the current modal, which remains open after feedback is dismissed. Call `DiffViewerFeedbackCapture.present(projectID:)` from your own development menu to expose the same action.

On iOS, shake the device or tap **Feedback**. Taking a system screenshot prepares an app-window attachment and freezes its interface reference; shake or tap **Add screenshot feedback** to open the note. The host can capture and present feedback while a sheet or full-screen modal is open.

For the exact original or edited iOS system screenshot, add a development **Share Extension** target whose principal controller subclasses `DiffViewerFeedbackShareViewController` and overrides `feedbackConfiguration`. Configure matching `appGroupIdentifier`, `keychainAccessGroup`, App Group and Keychain Sharing entitlements in the app and extension. The SDK composes the note inside the extension and uses the shared pairing and outbox. A recent screenshot reference supplies the original context; otherwise the editable title defaults to Screenshot. Installing the Swift package alone does not create an extension target.

Images are resized to a maximum 1,600-pixel edge, compressed to JPEG below 512 KB and saved without copying source image metadata. The note and image upload together through `POST /v1/feedback/reports/with-screenshot`; deploy a server with this endpoint before testing uploads.

To capture from Terminal on Mac, register the `diffviewer-feedback` URL scheme in the development app's Info.plist:

```xml
<key>CFBundleURLTypes</key>
<array><dict>
    <key>CFBundleURLName</key><string>Development feedback</string>
    <key>CFBundleURLSchemes</key><array><string>diffviewer-feedback</string></array>
</dict></array>
```

Then run:

```sh
open -a "Your Development App" 'diffviewer-feedback://capture'
```

URL registration and Share Extension targets belong to the consuming app. Keep those integrations in development builds.

## Optional: CloudKit text reports

For a CloudKit-configured build of DiffViewer, use `FeedbackConfiguration(projectID:containerIdentifier:)` instead of the server initializer. This mode supports text reports, without screenshot attachments. The development app and DiffViewer need the same entitled CloudKit container, iCloud account and CloudKit environment; the app's Apple Developer team needs access to that container. The SDK cannot grant these signing capabilities.

Enable iCloud / CloudKit in the app target and select the container shown by that build of DiffViewer. The private database needs a `DVRuntimeFeedback` record type with `payload` (Bytes), `projectID` (String), `createdAt` (Date/Time) and `schemaVersion` (Int64), plus a queryable `recordName` index. Deploy the schema and index before using Production. A Debug app does not automatically use the same environment as a distributed build of DiffViewer. This CloudKit setup is independent of server pairing.

## Structured capture data

The current source implementation adds capture-local element refs, parent relationships, control states, window/image geometry, quality limitations and reproduction environment to `runtimeContext.observation`. It also supports masked OCR fallback and app-defined event history through `DiffViewerFeedbackDiagnostics.record(projectID:category:name:)`. Imported screenshots explicitly distinguish a recent interface reference from a verified image match. These additions require rebuilding the binary SDK, deploying an API that preserves observations and updating the reader's Core module.

Configure `textExclusionIdentifiers` for custom sensitive controls and set `enablesTextRecognition` to false to disable OCR. OCR masks native editable/secure controls and configured exclusions in a separate image; screenshot attachments still contain visible app content. Event history retains at most 64 app-defined events from the last five minutes. Use fixed event names without credentials, entered text or request contents.
