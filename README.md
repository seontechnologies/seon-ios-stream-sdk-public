# SEON Stream SDK - iOS

The SEON Stream SDK continuously collects behavioural signals from your iOS application - touch patterns, typing dynamics, sensor data, network state, app state, and screen flows - and streams them to the SEON platform for fraud detection and risk assessment. The SDK supports both UIKit and SwiftUI applications.

---

## Requirements

- iOS 15.0 or higher
- Xcode 15 or higher
- Swift 5.9 or higher
- SEON Stream SDK 2.0.0 (see [Migrating from 1.x to 2.0](#migrating-from-1x-to-20); for 1.x, 1.1.0 or higher is the minimum supported version)

Integrations authenticate with an **authData** token obtained from the SEON authentication API (contact your SEON account representative for access).

---

## Installation

The SDK will be available through Swift Package Manager and CocoaPods.

### Swift Package Manager

In Xcode, select **File → Add Package Dependencies…** and enter the package URL:

`https://github.com/seontechnologies/seon-ios-stream-sdk-swift-package`

Select version `2.0.0` or later.

### CocoaPods

Add the SDK to your `Podfile` once the pod name is published:

```ruby
pod 'SeonStreamSDK', '2.0.0'
```

Then run:

```bash
pod install
```

---

## Getting started

### Initialization

Call `SEONSTStream.initializeSdk()` once, as early as possible, typically in your `AppDelegate` or `@main` `App` initializer. The SDK no longer takes credentials at initialization; pass `authData` when starting each session (see [Authentication](#authentication)).

Initialization is the only place where errors are reported **synchronously** instead of through the delegate. `initializeSdk()` and `sharedManager()` are imported as `throws` in Swift and return `nil` with the `NSError **` out-parameter set in Objective-C. They report an `SEONSTInitializeError` (`SEONSTInitializeErrorCode.initialization`) when the SDK is used before it is initialized, or when it is initialized a second time. After a second `initializeSdk()` call the existing instance stays in place.

Set the delegate before calling `initializeSdk()` to ensure no events are missed.

```swift
// Swift
import SeonStreamSDK

@main
struct MyApp: App {
    init() {
        SEONSTStream.setDelegate(appDelegate)
        do {
            try SEONSTStream.initializeSdk()
        } catch {
            print("SEON SDK initialization failed: \(error)")
        }
    }

    var body: some Scene {
        WindowGroup {
            ContentView()
        }
    }
}
```

```objc
// Objective-C
#import <SeonStreamSDK/SEONSTSeonStreamSDK.h>

- (BOOL)application:(UIApplication *)application
    didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    [SEONSTStream setDelegate:self];
    NSError *error = nil;
    if (![SEONSTStream initializeSdkWithError:&error]) {
        NSLog(@"SEON SDK initialization failed: %@", error);
    }
    return YES;
}
```

Retrieve the initialized instance later with `try SEONSTStream.sharedManager()` (Swift) or `[SEONSTStream sharedManagerWithError:&error]` (Objective-C).

### Configuring tracking

By default the SDK tracks screens and UI elements automatically. To take over part of the tracking, pass an `SEONSTFeatureConfig` when initializing:

```swift
// Swift
let featureConfig = SEONSTFeatureConfig(autoScreenTracking: false, autoViewTracking: false)
try SEONSTStream.initializeSdk(featureConfig: featureConfig)
```

```objc
// Objective-C
SEONSTFeatureConfig *featureConfig = [[SEONSTFeatureConfig alloc] initWithAutoScreenTracking:NO
                                                                             autoViewTracking:NO];
NSError *error = nil;
[SEONSTStream initializeSdkWithFeatureConfig:featureConfig error:&error];
```


| Option               | Default | Description                                                                                                                                                                            |
| -------------------- | ------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `autoScreenTracking` | `true`  | The SDK tracks screens automatically. When `false`, your app reports screens itself with `changeScreen(_:)`. See [Manual screen tracking](#manual-screen-tracking).                     |
| `autoViewTracking`   | `true`  | The SDK resolves and tracks every touched view and input field automatically. When `false`, only views you tag are tracked. See [Manual UI tracking](#manual-ui-tracking).             |

`initializeSdk()` without a configuration is equivalent to both options being `true`.

### Delegate

Implement `SEONSTStreamDelegate` to receive session lifecycle events and errors. All delegate methods are required, and every callback is delivered on the main thread.


| Method                  | When called                                                                                                                                                   |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `onStreamStarted(_:)`   | A session has successfully started. The session ID is passed as the argument. Also called when a stored session is resumed.                                   |
| `onStreamFinished(_:)`  | A session has ended. The `SEONSTCriticalError` argument is `nil` after a normal `finishStream()` and carries the cause when the SDK stopped the stream itself.  |
| `onStreamError(_:)`     | A non-critical `SEONSTNonCriticalError` occurred. It never changes the session state: a running session keeps running. See [Error handling](#error-handling).   |


- `onStreamFinished` is the only "session ended" signal.
- `onStreamError` never means that the session stopped.
- A start failure is reported once, through `onStreamError`, and is never paired with `onStreamFinished`.
- A second error after the session has already ended is not reported again.

```swift
// Swift
extension AppDelegate: SEONSTStreamDelegate {
    func onStreamStarted(_ streamId: String) {
        print("Session started: \(streamId)")
    }

    func onStreamFinished(_ error: SEONSTCriticalError?) {
        guard let error else {
            print("Session ended")
            return
        }
        switch error.errorCode {
        case .authenticationFailed:
            break // fetch fresh authData and start again, see "Recovering from an expired auth failure"
        case .sessionTimeout, .ipBanned, .storage, .finishStream:
            break
        @unknown default:
            break
        }
    }

    func onStreamError(_ error: SEONSTNonCriticalError) {
        switch error.errorCode {
        case .streamAlreadyRunning: break
        case .sessionConfig:        break
        case .rateLimited:          break
        //...
        default: break
        }
    }
}
```

```objc
// Objective-C
- (void)onStreamStarted:(NSString *)streamId {
    NSLog(@"Session started: %@", streamId);
}

- (void)onStreamFinished:(SEONSTCriticalError *)error {
    if (error == nil) {
        NSLog(@"Session ended");
        return;
    }
    switch (error.errorCode) {
        case SEONSTCriticalErrorCodeAuthenticationFailed:
            // fetch fresh authData and start again
            break;
        default:
            break;
    }
}

- (void)onStreamError:(SEONSTNonCriticalError *)error {
    NSLog(@"Error: %@", error);
}
```

### SwiftUI view manager

If your app uses SwiftUI and you want the SDK to resolve custom SwiftUI view identifiers, install the view manager once at the root of your view hierarchy.

```swift
WindowGroup {
    ContentView()
        .seonInstallViewManager()
}
```

Alternatively, wrap the root view with `SeonRoot`:

```swift
WindowGroup {
    SeonRoot {
        ContentView()
    }
}
```

UIKit applications do not need this step.

---

## Authentication

> **Version note:** The `authData`-based authentication introduced in 1.1.0 is required for production integrations. Version 1.0.0 was an open preview release without this mechanism; **1.1.0 is the minimum supported version**.

Each stream session is authorized with an **authData** value from the SEON authentication API. It is short-lived; your app should fetch a new one before starting a session and again after an auth failure.

1. Your backend calls `POST {environment}/session-monitoring-api/v1/auth` with the header `X-API-KEY: <your api key>` and passes the response body to the app as the `authData` string. The API key must never be embedded in the mobile application.
2. Pass `authData` in `SEONSTSessionConfig.authData` when calling `startStreamWith(config:)`.
3. The SDK extracts the session JWT and additional configuration from `authData` automatically.
4. If the JWT inside `authData` expires and the backend rejects further uploads, the SDK stops the stream and reports a critical `SEONSTCriticalErrorCode.authenticationFailed` error through `onStreamFinished(_:)` (see [Error handling](#error-handling)). The stored session is **not** cleared on auth failure, so you can fetch a fresh `authData` and call `startStreamWith(config:)` again with the **same `label`** to continue the session, as long as you are still within `maxBackgroundDuration`.

---

## Starting and stopping a session

### Starting a session

Use `SEONSTSessionConfig` to configure a session. `authData` from the SEON authentication API is required for production integrations.

```swift
// Swift
let authData = await fetchAuthData() // POST /session-monitoring-api/v1/auth
let config = SEONSTSessionConfig(
    authData: authData,
    label: "checkout-flow",
    maxBackgroundDuration: 60
)
let stream = try SEONSTStream.sharedManager()
stream.startStreamWith(config: config)
```

```objc
// Objective-C
NSString *authData = [self fetchAuthData]; // POST /session-monitoring-api/v1/auth
SEONSTSessionConfig *config = [[SEONSTSessionConfig alloc]
    initWithAuthData:authData
    label:@"checkout-flow"
    maxBackgroundDuration:60];
[[SEONSTStream sharedManagerWithError:nil] startStreamWithConfig:config];
```

To start a session without a label or background resumption window:

```swift
// Swift
let config = SEONSTSessionConfig(authData: authData, label: nil, maxBackgroundDuration: 0)
let stream = try SEONSTStream.sharedManager()
stream.startStreamWith(config: config)
```

The delegate's `onStreamStarted(_:)` is called with the session ID once the session has started. If the session cannot start, the delegate receives `onStreamError(_:)` and `onStreamStarted(_:)` is not called.


| Parameter               | Type           | Description                                                                                                                                                                                                                                                                                                                                                                                                  |
| ----------------------- | -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `authData`              | `String`       | Opaque token from the SEON authentication API (`POST /session-monitoring-api/v1/auth`). Contains the session JWT and additional SDK configuration. Fetch a fresh value before each session. A missing or malformed value fails the session start with `SEONSTNonCriticalErrorCode.sessionConfig`.                                                                                                              |
| `label`                 | `String?`      | Optional session label. Maximum length is `32` characters. Passing `nil` is valid. A longer label fails the session start with `SEONSTNonCriticalErrorCode.sessionConfig`. Required for session resumption after app restart or after an auth error.                                                                                                                                                       |
| `maxBackgroundDuration` | `TimeInterval` | Time window in seconds. If the app is restarted (due to a crash, user close, or system termination) within this window and `startStream` is called with the same `label`, the SDK resumes the previous session instead of starting a new one. The same window applies when resuming after an authentication failure with fresh `authData`. Pass `0` to disable resumption. Maximum value is `172800` seconds (48 hours). A negative value or a value above the maximum fails the session start with `SEONSTNonCriticalErrorCode.sessionConfig`. |


### Stopping a session

```swift
// Swift
let stream = try SEONSTStream.sharedManager()
stream.finishStream()
```

```objc
// Objective-C
[[SEONSTStream sharedManagerWithError:nil] finishStream];
```

The delegate's `onStreamFinished(_:)` is called with `nil` once the session has stopped. Calling `finishStream()` while no session is running reports `SEONSTNonCriticalErrorCode.streamNotRunning` through `onStreamError(_:)`.

### Recovering from an expired auth failure

When authentication fails, the delegate receives a single `onStreamFinished(_:)` with an `SEONSTCriticalError` whose `errorCode` is `.authenticationFailed`. Fetch fresh `authData` from the authentication API and start again:

```swift
// Swift
let freshAuthData = await fetchAuthData() // POST /session-monitoring-api/v1/auth
let config = SEONSTSessionConfig(
    authData: freshAuthData,
    label: "checkout-flow",              // same label as before
    maxBackgroundDuration: 60            // still within the resumption window
)
let stream = try SEONSTStream.sharedManager()
stream.startStreamWith(config: config)
```

Use the same `label` and stay within `maxBackgroundDuration` so the SDK continues the previous session instead of creating a new stream ID.

### Custom events

Send named custom events with optional additional data. The event name is limited to `32` characters and `additionalData` is limited to `1024` characters.

```swift
// Swift
let stream = try SEONSTStream.sharedManager()
stream.createCustomEventWith(
    name: "purchase_attempt",
    additionalData: "{\"amount\":99.99}"
)
```

```objc
// Objective-C
[[SEONSTStream sharedManagerWithError:nil]
    createCustomEventWithName:@"purchase_attempt"
    additionalData:@"{\"amount\":99.99}"];
```

`createCustomEventWith(name:additionalData:)` does not return a value. If validation fails or no session is running, the event is dropped and the error is reported through the delegate's `onStreamError(_:)`. A successful event produces no callback.

### Checking session status

```swift
// Swift
let stream = try SEONSTStream.sharedManager()
let running = stream.isRunning()
```

```objc
// Objective-C
BOOL running = [[SEONSTStream sharedManagerWithError:nil] isRunning];
```

### Error handling

Errors are delivered in three ways:

- **Initialization errors** (`SEONSTInitializeError`) are reported synchronously by `initializeSdk()` and `sharedManager()`: thrown in Swift, returned through the `NSError **` out-parameter in Objective-C. They never go through the delegate.
- **Critical errors** (`SEONSTCriticalError`) end the session and are delivered through `onStreamFinished(_:)`.
- **Non-critical errors** (`SEONSTNonCriticalError`) do not change the session state and are delivered through `onStreamError(_:)`.

All errors derive from `SEONSTError` (an `NSError` subclass) and use the domain `SEONSTErrorDomain`. Every error carries a numeric code, available as `NSError.code`. Each class also exposes a typed `errorCode` property backed by its own enum:

- `SEONSTCriticalErrorCode`: `1005`, `1012`, `1013`, `1014`, `1015`
- `SEONSTNonCriticalErrorCode`: `1001`, `1002`, `1004`, `1006`, `1016`, `1017`
- `SEONSTInitializeErrorCode`: `1009`

| Code | Constant                                           | Cause                                                                                                           | What happens                                                                                                                                             | Reported via                             | Message                                                                                   |
| ---- | -------------------------------------------------- | --------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------- | ----------------------------------------------------------------------------------------- |
| 1001 | `SEONSTNonCriticalErrorCodeStreamAlreadyRunning`   | `startStreamWith` was called while a session is already running.                                                | Nothing changes. The running session continues.                                                                                                          | `onStreamError`                          | Stream is already running.                                                                |
| 1002 | `SEONSTNonCriticalErrorCodeStartStream`            | Internal failure while starting the session. | The session does not start.                                                                                                    | `onStreamError`, no `onStreamStarted`    | Internal error during stream start.                                                       |
| 1004 | `SEONSTNonCriticalErrorCodeStreamNotRunning`       | `finishStream` or a custom event while no session is running.                                                   | Nothing changes.                                                                                                                                         | `onStreamError`                          | Stream is not running.                                                                    |
| 1005 | `SEONSTCriticalErrorCodeFinishStream`              | Internal failure while stopping the session.                                                                 | The session stops.                                                                                                                                       | `onStreamFinished`                       | Internal error during stream stop.                                                        |
| 1006 | `SEONSTNonCriticalErrorCodeCustomEventInvalid`     | The custom event name is missing or longer than `32` characters, or `additionalData` is longer than `1024` characters. | The event is dropped. The session keeps running.                                                                                                         | `onStreamError`                          | Custom event name is mandatory and cannot be longer than 32 characters. / Custom event additional data is too long. Maximum length is 1024 characters. |
| 1009 | `SEONSTInitializeErrorCodeInitialization`          | The SDK was used before initialization, or `initializeSdk` was called a second time.                            | Returned/thrown synchronously. After a second `initializeSdk` the existing instance stays.                                                               | Synchronous (not the delegate)           | SeonStream SDK has not been initialized. / SeonStream SDK is already initialized.         |
| 1012 | `SEONSTCriticalErrorCodeSessionTimeout`            | The backend reported that the session reached its maximum allowed duration.                                     | The session stops and is cleared.                                                                                                                        | `onStreamFinished`                       | Stream is expired.                                                                        |
| 1013 | `SEONSTCriticalErrorCodeAuthenticationFailed`      | The backend rejected authentication, for example an expired or invalid JWT.                                     | The session stops. The stored session is kept: a later start with fresh `authData` and the same `label` resumes it within `maxBackgroundDuration`.       | `onStreamFinished`                       | Auth token is expired.                                                                    |
| 1014 | `SEONSTCriticalErrorCodeIpBanned`                  | The client IP address was banned by the service.                                                                | The session stops. The stored session is kept.                                                                                                           | `onStreamFinished`                       | IP address is banned.                                                                     |
| 1015 | `SEONSTCriticalErrorCodeStorage`                   | An internal storage failure occurred during a session.                                           | The session stops.                                                                                                                                       | `onStreamFinished`                       | Internal storage error.                                                                   |
| 1016 | `SEONSTNonCriticalErrorCodeSessionConfig`          | The session configuration is invalid: `label` longer than `32` characters, `maxBackgroundDuration` negative or above `172800` seconds, or `authData` missing or invalid. | The session does not start.                                                                                                    | `onStreamError`, no `onStreamStarted`    | Label is too long. Maximum length is 32 characters. / maxBackgroundDuration is invalid. It must be between 0 and 2 days (172 800 s). / authData cannot be null. / authData is invalid. |
| 1017 | `SEONSTNonCriticalErrorCodeRateLimited`            | The backend rate limit was reached (HTTP 429).                                                                  | The session keeps running and the SDK retries automatically. The error is reported once per rate-limit episode, and again only after a successful upload. | `onStreamError`                          | Rate limit reached. Data transmission is throttled.                                       |


Other network failures are not reported to the app: the SDK retries automatically.

Codes `1003`, `1007`, `1008`, `1010` and `1011` from SDK 1.x no longer exist. Their causes are reported under the codes above; see [Migrating from 1.x to 2.0](#migrating-from-1x-to-20).

---

## View identification

The SDK resolves the UI element associated with each touch and input event. You can annotate your views with semantic identifiers to make this resolution more precise.

### SwiftUI

```swift
Button("Pay now") { }
    .seonIdentifyButton("pay_button")

TextField("Email", text: $email)
    .seonIdentifyInput("email_field")

Image("hero_banner")
    .seonIdentify("hero_banner", type: .image, interactable: false, data: ["campaign": "spring"])
```

### UIKit

```swift
loginButton.seonIdentify("login_button", type: .button, interactable: true, data: nil)
```

```objc
[self.loginButton seonIdentify:@"login_button"
                          type:SEONSTViewTypeButton
                  interactable:YES
                          data:nil];
```

Available types: `unknown`, `button`, `image`, `label`, `textField`, `list`, `container`, `adjustable`.

If no identifier is set, the SDK falls back to the view's `accessibilityIdentifier`, then `accessibilityLabel`, then the class name.

---

## Form tagging

Form tagging allows you to define logical forms and assign UI elements to them, enabling the SDK to produce more precise form fill-out signals. Without it, every input is reported on its own; with it, SEON also knows which elements belong to the same form, how many elements the form consists of, and which of them are optional.

> **Availability:** SDK 1.2.0 or higher.

Create one `SEONSTFormTag` per logical form and pass it while identifying each element that belongs to that form.


| Parameter          | Type      | Description                                                                                                                                           |
| ------------------ | --------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| `identifier`       | `String`  | Stable identifier of the form. The value is normalised: it is trimmed, whitespace is replaced with `_`, and it is lowercased.                          |
| `displayName`      | `String?` | Human-readable form name. Optional — pass `nil` to report the identifier only.                                                                         |
| `numberOfElements` | `Int`     | Total number of elements the complete form consists of, including optional ones. Used to measure fill-out progress, so keep it in sync with your form. |


Call `optional()` on the tag when attaching it to an element that is not required for the form to count as filled out. It returns a copy of the tag, so the same instance can still be used for the required elements.

### SwiftUI

```swift
struct CheckoutForm: View {
    @State private var name = ""
    @State private var email = ""
    @State private var note = ""

    private let form = SEONSTFormTag(
        identifier: "checkout_form",
        displayName: "Checkout form",
        numberOfElements: 3
    )

    var body: some View {
        Form {
            TextField("Name", text: $name)
                .seonIdentifyInput("checkout_name", formTag: form)

            TextField("Email", text: $email)
                .seonIdentifyInput("checkout_email", formTag: form)

            TextEditor(text: $note)
                .seonIdentifyInput("checkout_note", formTag: form.optional())
        }
    }
}
```

As with any SwiftUI identification, `.seonInstallViewManager()` or `SeonRoot` must be installed at the root of the view hierarchy.

### UIKit

```swift
// Swift
let form = SEONSTFormTag(
    identifier: "checkout_form",
    displayName: "Checkout form",
    numberOfElements: 3
)

nameField.seonIdentifyInput("checkout_name", formTag: form)
emailField.seonIdentifyInput("checkout_email", formTag: form)
noteTextView.seonIdentifyInput("checkout_note", formTag: form.optional())
```

```objc
// Objective-C
SEONSTFormTag *form = [[SEONSTFormTag alloc] initWithIdentifier:@"checkout_form"
                                                    displayName:@"Checkout form"
                                               numberOfElements:3];

[self.nameField seonIdentifyInput:@"checkout_name" data:nil formTag:form];
[self.emailField seonIdentifyInput:@"checkout_email" data:nil formTag:form];
[self.noteTextView seonIdentifyInput:@"checkout_note" data:nil formTag:form.optional];
```

The same `formTag:` parameter is available on `seonIdentify(...)` and `seonIdentifyButton(...)` for elements that are not text inputs.

### Behaviour notes

- **Multi-page forms** — keep a single tag instance (for example in a view model) and attach it on every page. Elements reported from different screens are grouped into the same form as long as they share the identifier.
- **Multiple forms on one screen** — create a separate tag per form. Elements are grouped by form identifier, so the forms stay independent.
- **Untagged elements** — inputs without a form tag keep being tracked as standalone elements.
- **Re-apply after view recreation** — tags live on the view instance, so elements must be tagged again whenever the view is recreated (a recreated SwiftUI body, a reused cell, or a rebuilt view controller).

---

## Screen and app state tracking

During an active session, the SDK automatically tracks app foreground and background transitions. It also records screen transitions emitted through navigation-controller-based flows and uses the screen title or view-controller class name as the screen identity. You can give a screen a stable name and id with [Screen tagging](#screen-tagging), or report screens yourself with [Manual screen tracking](#manual-screen-tracking).

The SDK also records display lock and unlock state when those system notifications are available.

### Screen tagging

By default a screen is identified by its view controller: the `navigationItem.title` if set, otherwise the class name. That identity is often not meaningful, for example every pushed screen of a SwiftUI `NavigationStack` shares the same class name, and a title changes with the device language. Screen tagging lets you give a screen an explicit **name** and, optionally, a unique **id**.

> **Availability:** SDK 2.0.0 or higher. Screen tagging applies while automatic screen tracking is on (the default). It has no effect when you report screens with `changeScreen(_:)`.

| Parameter | Type      | Description                                                                                                                                              |
| --------- | --------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `name`    | `String`  | Human-readable screen name. Reported as the screen's title.                                                                                              |
| `id`      | `String?` | Optional unique identifier of the screen, reported as is. When omitted, the unique id is the `name`, normalised: trimmed, whitespace replaced with `_`, and lowercased (`Checkout Form` becomes `checkout_form`). |

#### UIKit

Tag the view controller before it appears, for example in `viewDidLoad()`:

```swift
// Swift
override func viewDidLoad() {
    super.viewDidLoad()
    seonIdentifyScreen("Checkout Form")
    // or with an explicit id: seonIdentifyScreen("Checkout Form", id: "checkout_v2")
}
```

```objc
// Objective-C
- (void)viewDidLoad {
    [super viewDidLoad];
    [self seonIdentifyScreen:@"Checkout Form"];
    // or with an explicit id: [self seonIdentifyScreen:@"Checkout Form" id:@"checkout_v2"];
}
```

#### SwiftUI

Add the modifier to the root view of the screen:

```swift
struct CheckoutView: View {
    var body: some View {
        Form {
            // ...
        }
        .seonIdentifyScreen("Checkout Form")
        // or with an explicit id: .seonIdentifyScreen("Checkout Form", id: "checkout_v2")
    }
}
```

The tag applies to the screen the view belongs to. It does not require `.seonInstallViewManager()` or `SeonRoot`.

#### Excluding a screen

If the `name` is an empty string (or only whitespace), the screen is excluded, whatever the `id` is, from tracking and no screen events are reported for it. Use this for view controllers that are part of another screen, for example a child view controller of a multi-page form:

```swift
// Swift
class CheckoutAddressPageViewController: UIViewController {
    override func viewDidLoad() {
        super.viewDidLoad()
        seonIdentifyScreen("")
    }
}
```

```objc
// Objective-C
[self seonIdentifyScreen:@""];
```

#### Behaviour notes

- **Untagged screens** keep being tracked as before, by title or class name.
- **Container view controllers** — an untagged view controller that has child view controllers is not reported (unchanged). A tagged container is reported, because the tag is an explicit request to track it. Tag the parent and exclude the children to report one screen for the whole flow.
- **One tag per view controller** — tagging again replaces the previous tag, also at runtime; the new values are used for the following screen events.
- **SwiftUI** — if several `seonIdentifyScreen` modifiers end up on the same screen (for example one on a parent view and one on a child), the last one applied wins. Apply it once per screen.

### Manual screen tracking

If you turn off `autoScreenTracking` (see [Configuring tracking](#configuring-tracking)), the SDK stops observing view controllers and your app reports the current screen itself:

```swift
// Swift
let stream = try SEONSTStream.sharedManager()
stream.changeScreen("Checkout")
```

```objc
// Objective-C
[[SEONSTStream sharedManagerWithError:nil] changeScreen:@"Checkout"];
```

Each call ends the previous screen and starts the new one, stamped with the time of the call. Calling it with the screen that is already current does nothing, and empty names are ignored. `changeScreen(_:)` is ignored while automatic screen tracking is on.

---

## Touch tracking

The SDK automatically captures touch interactions during an active session. No changes to gesture recognizers or responder chains are required.

For each completed gesture, the SDK records touch coordinates, relative position inside the target element, contact ellipse size, gesture duration, movement vector, and total path length.

---

## Input tracking

The SDK automatically monitors text input interactions during an active session, including focus changes, typing clusters, and clipboard actions.

> **Privacy:** The SDK does not capture the actual values typed into input fields. Only interaction metadata such as timing, character count deltas, focus changes, and clipboard action type is recorded.

The SDK works alongside your existing `UITextField` and `UITextView` delegates: your delegate callbacks keep being called as usual.

What is recorded:

- Focus gained and lost events
- Typing clusters with added character count, deleted character count, and typing duration
- Clipboard copy and paste actions without capturing clipboard contents

## Manual UI tracking

By default the SDK resolves every touched view and every text input automatically. If you prefer to control exactly which elements are reported, disable `autoViewTracking` (see [Configuring tracking](#configuring-tracking)) and tag the views you want tracked with the [view identification](#view-identification) API. 

> **Availability:** SDK 2.0.0 or higher.

```swift
// Swift
try SEONSTStream.initializeSdk(featureConfig: SEONSTFeatureConfig(autoScreenTracking: true, autoViewTracking: false))
```

With `autoViewTracking` off:

- **Touches** are reported only for views tagged with `seonIdentify`, `seonIdentifyButton` or `seonIdentifyInput`. Touches on untagged views are ignored.
- **Typing and focus** are reported only for text fields that are tagged. A UIKit `UITextField` or `UITextView` counts when it carries `seonIdentify` data. Tag SwiftUI fields themselves (`seonIdentifyInput`) rather than only their container.
- **Clipboard** copy and paste actions are reported only while a tagged field is focused.
- `UIApplication.sendEvent` is **not** swizzled. Touch tracking works only through the views you tagged.
- The recorded events are the same as in automatic mode; only the set of reported elements changes. The SDK still never captures typed values or clipboard contents.
- Untagged views are not resolved at all, so the accessibility-identifier and class-name fallbacks described under [View identification](#view-identification) do not apply.
- In SwiftUI, `.seonInstallViewManager()` or `SeonRoot` must be installed at the root of the hierarchy.

`autoScreenTracking` and `autoViewTracking` are independent: you can combine manual UI tracking with automatic screen tracking, screen tagging, or manual screen tracking.

---

## Common integration difficulties

> **Note:** Use SDK version 2.0.0 or higher. If you are still on 1.x, see [Migrating from 1.x to 2.0](#migrating-from-1x-to-20).

- **Initialize before first use** — `SEONSTStream.sharedManager()` throws an `SEONSTInitializeError` (`SEONSTInitializeErrorCode.initialization`) if it is called before `SEONSTStream.initializeSdk()`, and `initializeSdk()` throws the same error when called a second time. In Objective-C the `...WithError:` methods return `nil` and set the error out-parameter. These errors are never sent to the delegate. Initialize the SDK once during app startup.
- **Fetch authData per session** — Call `POST /session-monitoring-api/v1/auth` with your `X-API-KEY` header before each session and pass the response string as `SEONSTSessionConfig.authData`. Do not embed long-lived secrets in the app binary.
- **Resume after auth failure** — On authentication failure the SDK stops the running stream (`onStreamFinished` with `.authenticationFailed`) but keeps the stored session. Call `startStreamWith(config:)` again with fresh `authData`, the same `label`, and a `maxBackgroundDuration` window that still covers the gap since the last event.
- **Set the delegate before `initializeSdk()`** — To avoid missing any early lifecycle events or errors, set `SEONSTStream.setDelegate(_:)` before calling `initializeSdk()`.
- **`onStreamFinished` is the only "session ended" signal** — Handle critical errors there. `onStreamError` never means that the session stopped.
- **SwiftUI view identification requires root installation** — If you use SwiftUI and want custom `seonIdentify...` values to resolve correctly, install `.seonInstallViewManager()` or `SeonRoot` at the top of your SwiftUI hierarchy. This is also required for [Manual UI tracking](#manual-ui-tracking).
- **Session configuration is validated when the session starts** — An invalid `label` (longer than `32` characters), `maxBackgroundDuration` (negative or above `172800` seconds), or `authData` fails the session start. The delegate receives `SEONSTNonCriticalErrorCode.sessionConfig` via `onStreamError(_:)` and no `onStreamStarted(_:)`.
- **`maxBackgroundDuration` controls session resumption after app restart** — When set, if the app restarts within the specified number of seconds after an unfinished session (crash, user close, or system termination) and `startStream` is called with the same `label`, the SDK resumes the previous session instead of creating a new one. Pass `0` to disable resumption (every `startStream` call creates a new session). The maximum allowed value is `172800` seconds (48 hours).
- **Custom events require an active session** — `createCustomEventWith` reports `SEONSTNonCriticalErrorCode.streamNotRunning` through `onStreamError(_:)` if no session is active.
- **Tag every SwiftUI screen** — Pushed screens of a SwiftUI `NavigationStack` all share the same class name, so without a tag they cannot be told apart by their unique id. Tag each screen with `seonIdentifyScreen`.
- **Tag screens before they appear** — Call `seonIdentifyScreen` in `viewDidLoad()` (UIKit) or add the modifier to the screen's root view (SwiftUI), so that the first screen event already carries the tag.

---

## Limitations & auto-tagging behaviour

The SDK automatically assigns names to screens and UI elements where possible. The tables below summarise what is auto-tagged, what requires manual intervention, and what is not supported.

### Screen auto-tagging


| Scenario                                        | Auto-tagged? | Name source                   | Notes                                                                            |
| ----------------------------------------------- | ------------ | ----------------------------- | -------------------------------------------------------------------------------- |
| `UIViewController` (UIKit)                      | Yes          | View-controller class name    | Override by setting the view controller's `title` property, or give it an explicit name with [screen tagging](#screen-tagging). |
| SwiftUI views hosted in a `UIHostingController` | Yes          | Hosting controller class name | The resolved name reflects the hosting controller, not individual SwiftUI views. Use [screen tagging](#screen-tagging) (`.seonIdentifyScreen`) for a stable name. |
| Screen tagged with `seonIdentifyScreen`         | Yes          | Tag `name` and optional `id`  | Takes priority over title and class name. The unique id is the given `id`, or the normalised `name`. An empty `name` excludes the screen. |


### View identification


| Scenario                                                                            | Identified? | Name source                        | Notes                                                                                                                                                                                                                                                                                                                   |
| ----------------------------------------------------------------------------------- | ----------- | ---------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| UIKit view with `seonIdentify()`                                                    | Yes         | Custom identifier                  | Highest priority.                                                                                                                                                                                                                                                                                                       |
| UIKit view with `accessibilityIdentifier`                                           | Yes         | `accessibilityIdentifier` value    | Used when no `seonIdentify()` is set.                                                                                                                                                                                                                                                                                   |
| UIKit view with `accessibilityLabel`                                                | Yes         | `accessibilityLabel` value         | Used when neither `seonIdentify()` nor `accessibilityIdentifier` is set.                                                                                                                                                                                                                                                |
| UIKit view with none of the above                                                   | Partial     | Class name or `UNKNOWN`            | If no identifier can be found for the view or its surroundings, the class name is used, or `UNKNOWN` when nothing identifiable exists.                                                                                                                                                          |
| SwiftUI view with `seonIdentify()` / `seonIdentifyButton()` / `seonIdentifyInput()` | Yes         | Custom identifier                  | Requires `.seonInstallViewManager()` or `SeonRoot` at the root of the view hierarchy.                                                                                                                                                                                                                                   |
| SwiftUI view without a modifier                                                     | Partial     | Accessibility traits or class name | Automatic identification in SwiftUI has limitations because accessibility properties are not always reliable when no accessibility features are active on the device. The SDK resolves accessibility traits (button, image, static text, etc.) when available. Without traits, the internal SwiftUI class name is used. |


### Input tracking


| Scenario                                                         | Tracked? | Notes                                                              |
| ---------------------------------------------------------------- | -------- | ------------------------------------------------------------------ |
| `UITextField`                                                    | Yes      | Focus, typing characteristics, and clipboard actions are captured. |
| `UITextView`                                                     | Yes      | Focus, typing characteristics, and clipboard actions are captured. |
| Custom input views (not extending `UITextField` or `UITextView`) | No       | Only `UITextField` and `UITextView` instances are detected.        |


### Touch tracking


| Scenario      | Tracked? | Notes                                                                                                                                                 |
| ------------- | -------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| UIKit views   | Yes      | Touch events are captured via `UIApplication.sendEvent` swizzling, unless `autoViewTracking` is off (see [Manual UI tracking](#manual-ui-tracking)). All touches are recorded regardless of the view's interactability. |
| SwiftUI views | Yes      | Touch events are captured via `UIApplication.sendEvent` swizzling, unless `autoViewTracking` is off (see [Manual UI tracking](#manual-ui-tracking)). Use the `seonIdentify` modifiers to give the touched views stable identifiers. |


## App Store Privacy (Privacy Manifest)

The SDK ships a `PrivacyInfo.xcprivacy` privacy manifest inside the XCFramework. Xcode aggregates it, together with the manifests of your app and other SDKs, into your app's **Privacy Report** (Product → Archive → right-click the archive → *Generate Privacy Report*). Use that report when filling in the **App Privacy** section in App Store Connect. The manifest does **not** update your App Store privacy labels automatically; as the app developer you remain responsible for keeping them accurate.

### Declared data collection

| Data type | Linked to the user | Used for tracking | Purpose | Collected when |
| --- | --- | --- | --- | --- |
| Product Interaction | Yes | No | App Functionality | Always (touch interactions, screen and view transitions, form fill-out progress, and custom events created with `createCustomEventWith`) |
| Performance Data | Yes | No | App Functionality | Always (timing signals associated with tracked interactions, e.g. gesture duration, movement vector and typing cadence) |
| Other Diagnostic Data | Yes | No | App Functionality | Always (internal SDK error/exception reports, queued locally and transmitted for diagnostics) |
| Other Data Types | Yes | No | App Functionality | Always (device, OS, network, and app-state signals — e.g. app foreground/background transitions and screen lock state — used to compute behavioural risk signals) |

**Why "linked to the user":** every event collected during a session is tagged with a `streamId` — a UUID the SDK generates and persists so it can resume a session across an app restart within `maxBackgroundDuration` — and is uploaded together with the `authData`/JWT issued for your authenticated session (see [Authentication](#authentication)). Under [Apple's definition](https://developer.apple.com/app-store/app-privacy-details/#linked-data), data associated with a user's account or session counts as linked to the user's identity. If your App Store privacy labels currently list these data types under *Data Not Linked to You*, move them to *Data Linked to You*.

**Tracking:** the SDK declares `NSPrivacyTracking = false` and no tracking domains. The collected data is used solely for fraud prevention and behavioural risk assessment, which Apple explicitly excludes from its definition of tracking. The SDK does not require you to show the App Tracking Transparency prompt.

### Declared required reason APIs

| API category | Reason code | Used for |
| --- | --- | --- |
| System boot time | `35F9.1` | Measuring elapsed time between internally tracked events (e.g. gesture duration) |
| User Defaults | `CA92.1` | Persisting the session identifier (`streamId`) used to resume an interrupted session, and queuing internal SDK error reports for later transmission |

## Migrating from 1.x to 2.0

<details>
<summary>Show the 1.x to 2.0 migration guide</summary>

Version 2.0.0 unifies session lifecycle and error reporting with the Android Stream SDK. The changes below are source-breaking, so the compiler points you at most of them. Session behaviour, authentication, and the event stream itself are unchanged. There is nothing to migrate if you only use view identification, form tagging, or `SEONSTSessionConfig`.

### 1. Initialization is `throws`

`initializeSdk()`, `initializeSdk(featureConfig:)` and `sharedManager()` no longer report problems through the delegate. They throw in Swift and return `nil` with an `NSError **` out-parameter in Objective-C.

```swift
// 1.x
SEONSTStream.initializeSdk()
SEONSTStream.sharedManager()?.startStreamWith(config: config)

// 2.0
try SEONSTStream.initializeSdk()
let stream = try SEONSTStream.sharedManager()
stream.startStreamWith(config: config)
```

```objc
// 1.x
[SEONSTStream initializeSdk];
[[SEONSTStream sharedManager] startStreamWithConfig:config];

// 2.0
NSError *error = nil;
[SEONSTStream initializeSdkWithError:&error];
[[SEONSTStream sharedManagerWithError:&error] startStreamWithConfig:config];
```

If you relied on the delegate receiving `SdkNotInitialized` or `SdkAlreadyInitialized`, handle the thrown `SEONSTInitializeError` (`SEONSTInitializeErrorCode.initialization`) instead.

### 2. Delegate signatures changed

```swift
// 1.x
func onStreamFinished()
func onStreamError(_ error: NSError)

// 2.0
func onStreamFinished(_ error: SEONSTCriticalError?)
func onStreamError(_ error: SEONSTNonCriticalError)
```

```objc
// 1.x
- (void)onStreamFinished;
- (void)onStreamError:(NSError *)error;

// 2.0
- (void)onStreamFinished:(SEONSTCriticalError *)error;
- (void)onStreamError:(SEONSTNonCriticalError *)error;
```

`onStreamFinished` now also carries the reason when the SDK stopped the session itself (`nil` after a normal `finishStream()`). In 1.x, authentication failures, session timeouts, IP bans and storage failures produced `onStreamFinished` followed by `onStreamError`. In 2.0 they produce a **single** `onStreamFinished(error)`, so move that handling out of `onStreamError`. `onStreamError` now only reports errors that leave the session as it was. Callbacks are delivered on the main thread.

### 3. Error codes are split into three enums

The single `SEONSTErrorCode` enum is gone. Each error class exposes a typed `errorCode` property backed by its own enum, so replace `(error as? SEONSTError)?.code` with `error.errorCode`. The numeric values and `NSError.code` are unchanged, except for the merged codes below.


| 1.x code | 1.x constant                                 | 2.0 constant                                         | 2.0 code |
| -------- | -------------------------------------------- | ---------------------------------------------------- | -------- |
| 1001     | `SEONSTErrorCodeStreamAlreadyRunning`        | `SEONSTNonCriticalErrorCodeStreamAlreadyRunning`     | 1001     |
| 1002     | `SEONSTErrorCodeStartStream`                 | `SEONSTNonCriticalErrorCodeStartStream`              | 1002     |
| 1003     | `SEONSTErrorCodeLabelTooLong`                | `SEONSTNonCriticalErrorCodeSessionConfig`            | 1016     |
| 1004     | `SEONSTErrorCodeStreamNotRunning`            | `SEONSTNonCriticalErrorCodeStreamNotRunning`         | 1004     |
| 1005     | `SEONSTErrorCodeFinishStream`                | `SEONSTCriticalErrorCodeFinishStream`                | 1005     |
| 1006     | `SEONSTErrorCodeCustomEventDataTooLong`      | `SEONSTNonCriticalErrorCodeCustomEventInvalid`       | 1006     |
| 1007     | `SEONSTErrorCodeDatabase`                    | `SEONSTNonCriticalErrorCodeStartStream`              | 1002     |
| 1008     | `SEONSTErrorCodeCustomEventNameTooLong`      | `SEONSTNonCriticalErrorCodeCustomEventInvalid`       | 1006     |
| 1009     | `SEONSTErrorCodeSdkNotInitialized`           | `SEONSTInitializeErrorCodeInitialization`            | 1009     |
| 1010     | `SEONSTErrorCodeSdkAlreadyInitialized`       | `SEONSTInitializeErrorCodeInitialization`            | 1009     |
| 1011     | `SEONSTErrorCodeMaxBackgroundDurationTooBig` | `SEONSTNonCriticalErrorCodeSessionConfig`            | 1016     |
| 1012     | `SEONSTErrorCodeSessionTimeout`              | `SEONSTCriticalErrorCodeSessionTimeout`              | 1012     |
| 1013     | `SEONSTErrorCodeAuthenticationFailed`        | `SEONSTCriticalErrorCodeAuthenticationFailed`        | 1013     |
| 1014     | `SEONSTErrorCodeIpBanned`                    | `SEONSTCriticalErrorCodeIpBanned`                    | 1014     |
| 1015     | `SEONSTErrorCodeStorage`                     | `SEONSTCriticalErrorCodeStorage`                     | 1015     |
| 1016     | `SEONSTErrorCodeSessionConfig`               | `SEONSTNonCriticalErrorCodeSessionConfig`            | 1016     |
| n/a      | n/a                                          | `SEONSTNonCriticalErrorCodeRateLimited` (new)        | 1017     |


Where several 1.x codes were merged into one, the error message, not the code, tells the causes apart. Also note:

- A label longer than `32` characters and an invalid `maxBackgroundDuration` now report `1016` (they were `1003` and `1011`).
- The upper limit of `maxBackgroundDuration` is unchanged (`172800` seconds), and a negative value is now rejected as well.
- A missing `authData` is reported as `1016` ("authData cannot be null.").

### 4. `createCustomEventWith` no longer returns an error

```swift
// 1.x
if let error = SEONSTStream.sharedManager()?.createCustomEventWith(name: "purchase_attempt", additionalData: nil) {
    print(error)
}

// 2.0
let stream = try SEONSTStream.sharedManager()
stream.createCustomEventWith(name: "purchase_attempt", additionalData: nil)
// failures arrive in onStreamError(_:)
```

```objc
// 1.x
NSError *error = [[SEONSTStream sharedManager] createCustomEventWithName:@"purchase_attempt" additionalData:nil];

// 2.0
[[SEONSTStream sharedManagerWithError:nil] createCustomEventWithName:@"purchase_attempt" additionalData:nil];
```

### 5. Rate limiting is reported

When the backend answers with HTTP 429, the SDK now reports the non-critical error `SEONSTNonCriticalErrorCodeRateLimited` (`1017`) through `onStreamError(_:)` once per rate-limit episode. The session keeps running and the SDK retries with backoff, so no action is required. In 1.x this was silent.

### 6. `SEONSTFeatureConfig` options renamed

If you passed an `SEONSTFeatureConfig` to `initializeSdk(featureConfig:)`, the 1.x options `screenTrackerEnabled`, `inputTrackerEnabled` and `touchTrackerEnabled` are replaced by `autoScreenTracking` and `autoViewTracking`. See [Configuring tracking](#configuring-tracking).

### Migration checklist

1. Update the package or pod to `2.0.0`.
2. Add `try` to `initializeSdk()` and `sharedManager()` calls (Objective-C: use the `...WithError:` methods).
3. Update the delegate to the new `onStreamFinished(_:)` and `onStreamError(_:)` signatures, and move critical-error handling to `onStreamFinished`.
4. Replace `SEONSTErrorCode` usages with the typed `errorCode` of the new error classes (see the table above).
5. Remove the return value handling of `createCustomEventWith`.
6. Optional, new in 2.0: [screen tagging](#screen-tagging), [manual screen tracking](#manual-screen-tracking), and [manual UI tracking](#manual-ui-tracking).

</details>

---

## Changelog

### 2.0.0

> ⚠️ **Breaking changes.** See [Migrating from 1.x to 2.0](#migrating-from-1x-to-20).

- **Session lifecycle and error handling** — unified with the Android Stream SDK.
  - `onStreamFinished(_:)` takes an optional `SEONSTCriticalError`. Authentication, session timeout, IP ban and storage failures are now a single `onStreamFinished(error)` instead of `onStreamFinished` followed by `onStreamError`.
  - `onStreamError(_:)` takes an `SEONSTNonCriticalError` and never means that the session stopped.
  - `initializeSdk()` and `sharedManager()` are `throws` in Swift and return `nil` with an `NSError **` out-parameter in Objective-C, instead of reporting through the delegate.
  - `SEONSTErrorCode` is replaced by `SEONSTCriticalErrorCode`, `SEONSTNonCriticalErrorCode` and `SEONSTInitializeErrorCode`, with a typed `errorCode` on `SEONSTCriticalError`, `SEONSTNonCriticalError` and `SEONSTInitializeError`. Codes `1003`, `1007`, `1008`, `1010` and `1011` were merged into `1016`, `1002`, `1006`, `1009` and `1016`.
  - `createCustomEventWith(name:additionalData:)` no longer returns an `NSError`; failures are reported through `onStreamError(_:)`.
  - Backend rate limiting (HTTP 429) is reported as the non-critical error `1017` (`SEONSTNonCriticalErrorCodeRateLimited`). The SDK keeps the session running and retries with backoff.
  - All delegate callbacks are delivered on the main thread. Error messages are unified with Android.
- **Manual UI tracking** — `SEONSTFeatureConfig(autoScreenTracking:autoViewTracking:)` lets you turn off automatic view tracking, so only the views you tag are tracked for touches, typing, focus and clipboard actions. See [Manual UI tracking](#manual-ui-tracking).
- **Manual screen tracking** — with `autoScreenTracking` off, report screens with `changeScreen(_:)`. See [Manual screen tracking](#manual-screen-tracking).
- **Screen tagging** — `seonIdentifyScreen(_:id:)` for UIKit view controllers and SwiftUI views gives a screen an explicit name and an optional unique id for automatic screen tracking (the id defaults to the normalised name), and excludes the screen when the name is an empty string. See [Screen tagging](#screen-tagging).
- `SEONSTFeatureConfig` options `screenTrackerEnabled`, `inputTrackerEnabled` and `touchTrackerEnabled` are replaced by `autoScreenTracking` and `autoViewTracking`.

### 1.3.3

- Fixed minor internal bugs.

### 1.3.2

- Minor fixes and improvements.

### 1.3.1

- Fixed minor internal bugs.

### 1.3.0

> **Note:** the SDK now ships a privacy manifest. See the [App Store Privacy](#app-store-privacy-privacy-manifest) section for details.

- Added `PrivacyInfo.xcprivacy`, declaring the SDK's collected data types and required reason API usage for Apple's App Store privacy requirements.
- Internal changes and improvements for upcoming features.

### 1.2.1

- **Form tagging** — allows clients to define logical forms and assign UI elements to them, enabling to produce more precise form fill-out signals. See [Form tagging](#form-tagging).
- Improve network transmission logic

### 1.2.0

> ⚠️ **IMPORTANT! This version was built against a yet unsupported iOS SDK 27. Use v1.2.1 instead.** ⚠️

### 1.1.0

**First production-ready release.**

- `SEONSTSessionConfig.authData` replaces `token`: pass the opaque `authData` string from `POST /session-monitoring-api/v1/auth` to start a session. The SDK extracts the JWT and additional configuration from it automatically.
- Introduced new initialize method `[SEONSTStream initializeSdk]`
- `[SEONSTStream initialize:(SEONSTGlobalConfig *)configuration]`, `SEONSTGlobalConfig` and `setToken(_:)` are deprecated
- Authentication failures end the stream with `SEONSTErrorCodeAuthenticationFailed` while preserving the session for resumption within `maxBackgroundDuration`

### 1.0.1

- Fix internal view synchronisation issue

### 1.0.0

- Initial open preview release — not supported for production use.

