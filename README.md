# SEON Stream SDK - iOS

The SEON Stream SDK continuously collects behavioural signals from your iOS application - touch patterns, typing dynamics, sensor data, network state, app state, and screen flows - and streams them to the SEON platform for fraud detection and risk assessment. The SDK supports both UIKit and SwiftUI applications.

---

## Requirements

- iOS 13.0 or higher
- Xcode 15 or higher
- Swift 5.9 or higher
- SEON Stream SDK 1.1.0 or higher (minimum supported version)

Integrations authenticate with an **authData** token obtained from the SEON authentication API (contact your SEON account representative for access).

---

## Installation

The SDK will be available through Swift Package Manager and CocoaPods.

### Swift Package Manager

In Xcode, select **File → Add Package Dependencies…** and enter the package URL:

`https://github.com/seontechnologies/seon-ios-stream-sdk-swift-package`

Select version `1.2.0` or later.

### CocoaPods

Add the SDK to your `Podfile` once the pod name is published:

```ruby
pod 'SeonStreamSDK', '1.2.0'
```

Then run:

```bash
pod install
```

---

## Getting started

### Initialization

Call `SEONSTStream.initializeSdk()` once, as early as possible, typically in your `AppDelegate` or `@main` `App` initializer. The SDK no longer takes credentials at initialization; pass a JWT when starting each session (see [Authentication](#authentication)).

Accessing `SEONSTStream.sharedManager()` before initialization returns nil and calls the delegate `onStreamError(_:)` method. Calling `initializeSdk()` more than once reports `SEONSTErrorCodeSdkAlreadyInitialized` through the delegate.

Set the delegate before calling `initializeSdk()` to ensure no events are missed.

```swift
// Swift
import SeonStreamSDK

@main
struct MyApp: App {
    init() {
        SEONSTStream.setDelegate(appDelegate)
        SEONSTStream.initializeSdk()
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
    [SEONSTStream initializeSdk];
    return YES;
}
```

### Delegate

Implement `SEONSTStreamDelegate` to receive session lifecycle events and errors. All delegate methods are required.


| Method                  | When called                                                                           |
| ----------------------- | ------------------------------------------------------------------------------------- |
| `onStreamStarted(_:)`   | A session has successfully started. The session ID is passed as the argument.         |
| `onStreamFinished`      | A session has ended, either via `finishStream()` or by the SDK internally.            |
| `onStreamError(_:)`     | An error occurred. See the [Error handling](#error-handling) section for error codes. |


```swift
// Swift
extension AppDelegate: SEONSTStreamDelegate {
    func onStreamStarted(_ streamId: String) {
        print("Session started: \(streamId)")
    }

    func onStreamFinished() {
        print("Session ended")
    }

    func onStreamError(_ error: NSError) {
        print("Error: \(error)")
        switch (error as? SEONSTError)?.code {
        case .streamAlreadyRunning: break
        case .labelTooLong:         break
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

- (void)onStreamFinished {
    NSLog(@"Session ended");
}

- (void)onStreamError:(NSError *)error {
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
4. If the JWT inside `authData` expires and the backend rejects further uploads, the SDK stops the stream and reports `SEONSTErrorCodeAuthenticationFailed` (see [Error handling](#error-handling)). The stored session is **not** cleared on auth failure, so you can fetch a fresh `authData` and call `startStreamWith(config:)` again with the **same `label`** to continue the session, as long as you are still within `maxBackgroundDuration`.

---

## Starting and stopping a session

### Starting a session

Use `SEONSTSessionConfig` to configure a session. `authData` from the SEON authentication API is required for production integrations.

```swift
// Swift
let authData = await fetchAuthData() // GET /session-monitoring-api/v1/auth
let config = SEONSTSessionConfig(
    authData: authData,
    label: "checkout-flow",
    maxBackgroundDuration: 60
)
SEONSTStream.sharedManager()?.startStreamWith(config: config)
```

```objc
// Objective-C
NSString *authData = [self fetchAuthData]; // GET /session-monitoring-api/v1/auth
SEONSTSessionConfig *config = [[SEONSTSessionConfig alloc]
    initWithAuthData:authData
    label:@"checkout-flow"
    maxBackgroundDuration:60];
[[SEONSTStream sharedManager] startStreamWithConfig:config];
```

To start a session without a label or background resumption window:

```swift
// Swift
let config = SEONSTSessionConfig(authData: authData, label: nil, maxBackgroundDuration: 0)
SEONSTStream.sharedManager()?.startStreamWith(config: config)
```

The delegate's `onStreamStarted(_:)` is called with the session ID once the session has started. Errors are reported via `onStreamError(_:)`.


| Parameter               | Type           | Description                                                                                                                                                                                                                                                                                                                                                                                                                            |
| ----------------------- | -------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `authData`              | `String?`      | Opaque token from the SEON authentication API (`GET /session-monitoring-api/v1/auth`). Contains the session JWT and additional SDK configuration. Fetch a fresh value before each session.                                                                                                                                                                                  |
| `label`                 | `String?`      | Optional session label. Maximum length is `32` characters. Passing `nil` is valid. If the label exceeds `32` characters, the session start fails with `SEONSTErrorCodeLabelTooLong`. Required for session resumption after app restart or after an auth error.                                                                                                                                                                         |
| `maxBackgroundDuration` | `TimeInterval` | Time window in seconds. If the app is restarted (due to a crash, user close, or system termination) within this window and `startStream` is called with the same `label`, the SDK resumes the previous session instead of starting a new one. The same window applies when resuming after `SEONSTErrorCodeAuthenticationFailed` with fresh `authData`. Pass `0` to disable resumption. Maximum value is `172800` seconds (48 hours). Exceeding this limit fails the session start with `SEONSTErrorCodeMaxBackgroundDurationTooBig`. |


### Stopping a session

```swift
// Swift
SEONSTStream.sharedManager()?.finishStream()
```

```objc
// Objective-C
[[SEONSTStream sharedManager] finishStream];
```

The delegate's `onStreamFinished` is called once the session has stopped. Errors are reported via `onStreamError(_:)`.

### Recovering from an expired auth failure

When authentication fails, the delegate receives `onStreamFinished`, then `onStreamError(_:)` with `SEONSTErrorCodeAuthenticationFailed`. Fetch fresh `authData` from the authentication API and start again:

```swift
// Swift
let freshAuthData = await fetchAuthData() // GET /session-monitoring-api/v1/auth
let config = SEONSTSessionConfig(
    authData: freshAuthData,
    label: "checkout-flow",              // same label as before
    maxBackgroundDuration: 60            // still within the resumption window
)
SEONSTStream.sharedManager()?.startStreamWith(config: config)
```

Use the same `label` and stay within `maxBackgroundDuration` so the SDK continues the previous session instead of creating a new stream ID.

### Custom events

Send named custom events with optional additional data. The event name is limited to `32` characters and `additionalData` is limited to `1024` characters.

```swift
// Swift
let error = SEONSTStream.sharedManager()?.createCustomEventWith(
    name: "purchase_attempt",
    additionalData: "{\"amount\":99.99}"
)

if let error {
    print("Failed to create custom event: \(error)")
}
```

```objc
// Objective-C
NSError *error = [[SEONSTStream sharedManager]
    createCustomEventWithName:@"purchase_attempt"
    additionalData:@"{\"amount\":99.99}"];

if (error) {
    NSLog(@"Failed to create custom event: %@", error);
}
```

Returns an `NSError` synchronously if validation fails or no session is running, and `nil` on success.

### Checking session status

```swift
// Swift
let running = SEONSTStream.sharedManager()?.isRunning()
```

```objc
// Objective-C
BOOL running = [[SEONSTStream sharedManager] isRunning];
```

### Error handling

The SDK reports errors through three mechanisms:

- Initialization misuse (calling `sharedManager()` before `initializeSdk()`) returns nil and calls delegate `onStreamError(_:)`.
- Session lifecycle events, runtime failures, and other errors are reported through the delegate's `onStreamError(_:)` method.
- `createCustomEventWith(name:additionalData:)` returns `NSError` synchronously.

All `NSError` values produced by the SDK use domain `SEONSTErrorDomain`. Codes are defined as `SEONSTErrorCode` in `SEONSTError.h`. In Swift, prefer matching on `SEONSTError.Code` (or the bridged enum); the integer in the **Code** column is the value exposed as `NSError.code` for logging and Objective-C.


| Code | Constant                                     | Meaning                                                                                                                                                                             |
| ---- | -------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1001 | `SEONSTErrorCodeStreamAlreadyRunning`        | `startStreamWith` was called while a session is already active.                                                                                                                     |
| 1002 | `SEONSTErrorCodeStartStream`                 | Starting the session failed because of an internal startup error.                                                                                                                   |
| 1003 | `SEONSTErrorCodeLabelTooLong`                | The session `label` exceeds `32` characters.                                                                                                                                        |
| 1004 | `SEONSTErrorCodeStreamNotRunning`            | `finishStream` or `createCustomEventWith` was called while no session is active.                                                                                                    |
| 1005 | `SEONSTErrorCodeFinishStream`                | Stopping the session failed because of an internal shutdown error.                                                                                                                  |
| 1006 | `SEONSTErrorCodeCustomEventDataTooLong`      | The custom event `additionalData` exceeds `1024` characters.                                                                                                                        |
| 1007 | `SEONSTErrorCodeDatabase`                    | The SDK could not create or write to its local event storage.                                                                                                                       |
| 1008 | `SEONSTErrorCodeCustomEventNameTooLong`      | The custom event `name` exceeds `32` characters.                                                                                                                                    |
| 1009 | `SEONSTErrorCodeSdkNotInitialized`           | An SDK method was called before `initializeSdk()`.                                                                                                                                     |
| 1010 | `SEONSTErrorCodeSdkAlreadyInitialized`       | `initializeSdk()` was called more than once.                                                                                                                                           |
| 1011 | `SEONSTErrorCodeMaxBackgroundDurationTooBig` | The `maxBackgroundDuration` value exceeds the maximum of `172800` seconds (48 hours).                                                                                               |
| 1012 | `SEONSTErrorCodeSessionTimeout`              | The server reported that the session reached its maximum allowed duration. The delegate receives `onStreamFinished`, then `onStreamError(_:)` with this code while the stream is torn down. |
| 1013 | `SEONSTErrorCodeAuthenticationFailed`        | The backend rejected authentication (for example an expired or invalid JWT). The delegate receives `onStreamFinished`, then `onStreamError(_:)` with this code while the stream is torn down. The session is kept so you can resume with a new JWT and the same `label` within `maxBackgroundDuration`. |
| 1014 | `SEONSTErrorCodeIpBanned`                    | The client IP address was banned by the service. The delegate receives `onStreamFinished`, then `onStreamError(_:)` with this code while the stream is torn down.                           |
| 1015 | `SEONSTErrorCodeStorage`                     | The SDK failed to write event data to local storage during an active session. The delegate receives `onStreamFinished`, then `onStreamError(_:)` with this code while the stream is torn down. |
| 1016 | `SEONSTErrorCodeSessionConfig`               | The `authData` passed to `startStreamWith(config:)` is malformed or missing required fields (JWT or domain list). The session does not start. |


Other `NSError` values may use different domains (for example URL or system errors). Rare internal failures may surface with domain `SEONSTErrorDomain` and code `-1` when no specific `SEONSTErrorCode` applies.

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
                          type:SeonViewTypeButton
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

During an active session, the SDK automatically tracks app foreground and background transitions. It also records screen transitions emitted through navigation-controller-based flows and uses the screen title or view-controller class name as the screen identity.

The SDK also records display lock and unlock state when those system notifications are available.

---

## Touch tracking

The SDK automatically captures touch interactions during an active session. No changes to gesture recognizers or responder chains are required.

For each completed gesture, the SDK records touch coordinates, relative position inside the target element, contact ellipse size, gesture duration, movement vector, and total path length.

---

## Input tracking

The SDK automatically monitors text input interactions during an active session, including focus changes, typing clusters, and clipboard actions.

> **Privacy:** The SDK does not capture the actual values typed into input fields. Only interaction metadata such as timing, character count deltas, focus changes, and clipboard action type is recorded.

The SDK observes `UITextField` and `UITextView` editing notifications and temporarily inserts itself as a delegate while preserving and forwarding your existing delegate callbacks.

What is recorded:

- Focus gained and lost events
- Typing clusters with added character count, deleted character count, and typing duration
- Clipboard copy and paste actions without capturing clipboard contents

## Common integration difficulties

> **Note:** Use SDK version 1.1.0 or higher. Version 1.0.0 was a preview release and is not supported for production use.

- **Initialize before first use** — Calling `SEONSTStream.sharedManager()` before `SEONSTStream.initialize()` returns `nil` and calls the delegate `onStreamError(_:)` method. Initialize the SDK once during app startup.
- **Fetch authData per session** — Call `GET /session-monitoring-api/v1/auth` with your `X-API-KEY` header before each session and pass the response string as `SEONSTSessionConfig.authData`. Do not embed long-lived secrets in the app binary.
- **Resume after auth failure** — On authentication failure the SDK clears the running stream but keeps the stored session. Call `startStreamWith(config:)` again with fresh `authData`, the same `label`, and a `maxBackgroundDuration` window that still covers the gap since the last event.
- **Set the delegate before `initialize()`** — To avoid missing any early lifecycle events or errors, set `SEONSTStream.setDelegate(_:)` before calling `initialize()`.
- **SwiftUI view identification requires root installation** — If you use SwiftUI and want custom `seonIdentify...` values to resolve correctly, install `.seonInstallViewManager()` or `SeonRoot` at the top of your SwiftUI hierarchy.
- **Session labels are validated immediately** — If `label` is longer than `32` characters, `startStreamWith` fails and the delegate receives `SEONSTErrorCodeLabelTooLong` via `onStreamError(_:)`.
- `**maxBackgroundDuration` controls session resumption after app restart** — When set, if the app restarts within the specified number of seconds after an unfinished session (crash, user close, or system termination) and `startStream` is called with the same `label`, the SDK resumes the previous session instead of creating a new one. Pass `0` to disable resumption (every `startStream` call creates a new session). The maximum allowed value is `172800` seconds (48 hours); exceeding it fails the session start with `SEONSTErrorCodeMaxBackgroundDurationTooBig`.
- **Custom events require an active session** — `createCustomEventWith` returns `SEONSTErrorCodeStreamNotRunning` if no session is active.

---

## Limitations & auto-tagging behaviour

The SDK automatically assigns names to screens and UI elements where possible. The tables below summarise what is auto-tagged, what requires manual intervention, and what is not supported.

### Screen auto-tagging


| Scenario                                        | Auto-tagged? | Name source                   | Notes                                                                            |
| ----------------------------------------------- | ------------ | ----------------------------- | -------------------------------------------------------------------------------- |
| `UIViewController` (UIKit)                      | Yes          | View-controller class name    | Override by setting the view controller's `title` property.                      |
| SwiftUI views hosted in a `UIHostingController` | Yes          | Hosting controller class name | The resolved name reflects the hosting controller, not individual SwiftUI views. |


### View identification


| Scenario                                                                            | Identified? | Name source                        | Notes                                                                                                                                                                                                                                                                                                                   |
| ----------------------------------------------------------------------------------- | ----------- | ---------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| UIKit view with `seonIdentify()`                                                    | Yes         | Custom identifier                  | Highest priority.                                                                                                                                                                                                                                                                                                       |
| UIKit view with `accessibilityIdentifier`                                           | Yes         | `accessibilityIdentifier` value    | Used when no `seonIdentify()` is set.                                                                                                                                                                                                                                                                                   |
| UIKit view with `accessibilityLabel`                                                | Yes         | `accessibilityLabel` value         | Used when neither `seonIdentify()` nor `accessibilityIdentifier` is set.                                                                                                                                                                                                                                                |
| UIKit view with none of the above                                                   | Partial     | Class name or `UNKNOWN`            | The SDK walks up to 5 superviews looking for an identifier. If none is found, the class name is used. Views with no identifiable ancestor appear as `UNKNOWN`.                                                                                                                                                          |
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
| UIKit views   | Yes      | Touch events are captured via `UIApplication.sendEvent` swizzling. All touches are recorded regardless of the view's interactability.                 |
| SwiftUI views | Yes      | Touch events are captured via `UIApplication.sendEvent` swizzling and resolved against SwiftUI view data registered through `seonIdentify` modifiers. |


---

## Changelog

### 1.2.0

- **Form tagging** — allows clients to define logical forms and assign UI elements to them, enabling to produce more precise form fill-out signals. See [Form tagging](#form-tagging).
- Improve network transmission logic

### 1.1.0

**First production-ready release.**

- `SEONSTSessionConfig.authData` replaces `token`: pass the opaque `authData` string from `GET /session-monitoring-api/v1/auth` to start a session. The SDK extracts the JWT and additional configuration from it automatically.
- Introduced new initialize method `[SEONSTStream initializeSdk]`
- `[SEONSTStream initialize:(SEONSTGlobalConfig *)configuration]`, `SEONSTGlobalConfig` and `setToken(_:)` are deprecated
- Authentication failures end the stream with `SEONSTErrorCodeAuthenticationFailed` while preserving the session for resumption within `maxBackgroundDuration`

### 1.0.1

- Fix internal view synchronisation issue

### 1.0.0

- Initial open preview release — not supported for production use.

