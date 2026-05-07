# SEON Stream SDK - iOS

The SEON Stream SDK continuously collects behavioural signals from your iOS application - touch patterns, typing dynamics, sensor data, network state, app state, and screen flows - and streams them to the SEON platform for fraud detection and risk assessment. The SDK supports both UIKit and SwiftUI applications.

---

## Requirements

- iOS 13.0 or higher
- Xcode 15 or higher
- Swift 5.9 or higher

An API token issued by SEON is required. Contact your SEON account representative to obtain one.

---

## Installation

The SDK will be available through Swift Package Manager and CocoaPods.

### Swift Package Manager

In Xcode, select **File → Add Package Dependencies…** and enter the package URL:

`https://github.com/seontechnologies/seon-ios-stream-sdk-swift-package`

### CocoaPods

Add the SDK to your `Podfile` once the pod name is published:

```ruby
pod 'SeonStreamSDK', '1.0.1'
```

Then run:

```bash
pod install
```

---

## Getting started

### Configuration

The SDK is configured through `SEONSTGlobalConfig`, which is set once at initialization.


| Parameter | Type           | Description                                                     | Default |
| --------- | -------------- | --------------------------------------------------------------- | ------- |
| `token`   | `String`       | Your SEON API token.                                            | —       |
| `region`  | `SEONSTRegion` | Target region. Currently `.eu` (`SEONSTRegionEU`) is supported. | `.eu`   |


### Initialization

Call `SEONSTStream.initialize()` once, as early as possible, typically in your `AppDelegate` or `@main` `App` initializer.

Accessing `SEONSTStream.sharedManager()` before initialization returns nil and calls the delegate `onStreamError(_:)` method. Calling `initialize()` more than once reports `SEONSTErrorCodeSdkAlreadyInitialized` through the delegate.

Set the delegate before calling `initialize()` to ensure no events are missed.

```swift
// Swift
import SeonStreamSDK

@main
struct MyApp: App {
    init() {
        SEONSTStream.setDelegate(appDelegate)
        SEONSTStream.initialize(
            SEONSTGlobalConfig(token: "YOUR_API_TOKEN", region: .eu)
        )
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
    [SEONSTStream initialize:[[SEONSTGlobalConfig alloc]
        initWithToken:@"YOUR_API_TOKEN"
        region:SEONSTRegionEU]];
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

## Starting and stopping a session

### Starting a session

Use `SEONSTSessionConfig` to configure a session with an optional label and background duration limit.

```swift
// Swift
let config = SEONSTSessionConfig(label: "checkout-flow", maxBackgroundDuration: 60)
SEONSTStream.sharedManager()?.startStreamWith(config: config)
```

```objc
// Objective-C
SEONSTSessionConfig *config = [[SEONSTSessionConfig alloc]
    initWithLabel:@"checkout-flow"
    maxBackgroundDuration:60];
[[SEONSTStream sharedManager] startStreamWithConfig:config];
```

To start a session with default settings, omit the config:

```swift
// Swift
SEONSTStream.sharedManager()?.startStream()
```

```objc
// Objective-C
[[SEONSTStream sharedManager] startStream];
```

The delegate's `onStreamStarted(_:)` is called with the session ID once the session has started. Errors are reported via `onStreamError(_:)`.


| Parameter               | Type           | Description                                                                                                                                                                                                                                                                                                                                                                                                                            |
| ----------------------- | -------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `label`                 | `String?`      | Optional session label. Maximum length is `32` characters. Passing `nil` is valid. If the label exceeds `32` characters, the session start fails with `SEONSTErrorCodeLabelTooLong`.                                                                                                                                                                                                                                                   |
| `maxBackgroundDuration` | `TimeInterval` | Time window in seconds. If the app is restarted (due to a crash, user close, or system termination) within this window and `startStream` is called with the same `label`, the SDK resumes the previous session instead of starting a new one. Pass `0` to disable resumption. Maximum value is `172800` seconds (48 hours). Exceeding this limit fails the session start with `SEONSTErrorCodeMaxBackgroundDurationTooBig`. |


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

### Updating the token

Use `setToken(_:)` to replace the API token after initialization, for example when rotating credentials.

```swift
// Swift
SEONSTStream.sharedManager()?.setToken("NEW_API_TOKEN")
```

```objc
// Objective-C
[[SEONSTStream sharedManager] setToken:@"NEW_API_TOKEN"];
```

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

- Initialization misuse (calling `sharedManager()` before `initialize()`) returns nil and calls delegate `onStreamError(_:)`.
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
| 1009 | `SEONSTErrorCodeSdkNotInitialized`           | An SDK method was called before `initialize()`.                                                                                                                                     |
| 1010 | `SEONSTErrorCodeSdkAlreadyInitialized`       | `initialize()` was called more than once.                                                                                                                                           |
| 1011 | `SEONSTErrorCodeMaxBackgroundDurationTooBig` | The `maxBackgroundDuration` value exceeds the maximum of `172800` seconds (48 hours).                                                                                               |
| 1012 | `SEONSTErrorCodeSessionTimeout`              | The server reported that the session reached its maximum allowed duration. The delegate receives `onStreamFinished`, then `onStreamError(_:)` with this code while the stream is torn down. |
| 1013 | `SEONSTErrorCodeAuthenticationFailed`        | The backend rejected authentication (for example an invalid API token). The delegate receives `onStreamFinished`, then `onStreamError(_:)` with this code while the stream is torn down.      |
| 1014 | `SEONSTErrorCodeIpBanned`                    | The client IP address was banned by the service. The delegate receives `onStreamFinished`, then `onStreamError(_:)` with this code while the stream is torn down.                           |


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

- **Initialize before first use** — Calling `SEONSTStream.sharedManager()` before `SEONSTStream.initialize()` returns `nil` and calls the delegate `onStreamError(_:)` method. Initialize the SDK once during app startup.
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

### 1.0.1

- Fix internal view synchronisation issue

### 1.0.0

- Initial release

