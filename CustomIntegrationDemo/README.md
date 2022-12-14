# OwnID Custom-iOS SDK
The OwnID Custom-iOS SDK is a client library written in Swift that provides a passwordless login alternative for your iOS application by using cryptographic keys to replace the traditional password. Integrating the SDK with your iOS app adds a Skip Password option to its registration and login screens. For more general information about OwnID SDKs, see [OwnID iOS SDK](../README.md).

## Table of contents
* [Before You Begin](#before-you-begin)
* [Add Package Dependency](#add-package-dependency)
* [Add Property List File to Project](#add-property-list-file-to-project)
* [Create URL Type (Custom URL Scheme)](#create-url-type-custom-url-scheme)
* [Import OwnID Modules](#import-ownid-modules)
* [Initialize the SDK](#initialize-the-sdk)
* [Pass Redirection URL to SDK](#pass-redirection-url-to-sdk)
* [Implement the Registration Screen](#implement-the-registration-screen)
  + [Customize View Model](#customize-view-model)
  + [Add the OwnID View](#add-the-ownid-view)
* [Implement the Login Screen](#implement-the-login-screen)
  + [Customize View Model](#customize-view-model-1)
  + [Add OwnID View](#add-ownid-view)
* [Errors](#errors)
* [Advanced Configuration](#advanced-configuration)
  + [OwnID Web App language](#ownid-web-app-language)
  + [Directing Users to the OwnID iOS App](#directing-users-to-the-ownid-ios-app)
* [Logging](#logging)

## Before You Begin
Before incorporating OwnID into your iOS app, you must create an OwnID application and integrate it with your backend. For step-by-step instructions, see [OwnID-Custom Integration Basics](https://docs.ownid.com/Integrations/custom-integration).

## Add Package Dependency
The OwnID iOS SDK is distributed as an SPM package. Use the Swift Package Manager to add the following package dependency to your project:

```
pod 'ownid-core-ios-sdk'
```

## Add Property List File to Project

When the application starts, the OwnID SDK automatically reads `OwnIDConfiguration.plist` from the file system to configure the default instance that is created. At a minimum, this PLIST file defines a redirection URI and unique app id. Create `OwnIDConfiguration.plist` and define the following mandatory parameters:

[Complete example](./OwnIDConfiguration.plist)
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>OwnIDRedirectionURL</key>
	<string>com.myapp.demo://bazco</string>
	<key>OwnIDAppID</key>
	<string>4tb9nt6iaur0zv</string>
</dict>
</plist>
```
Where:

- The `OwnIDAppID` is the unique AppID, which you can obtain from the [OwnID Console](https://console.ownid.com).
- The `OwnIDRedirectionURL` is the full redirection URL, including its custom scheme. This URL custom scheme must match the one that you defined in your target.

## Create URL Type (Custom URL Scheme)
You need to open your project and create a new URL type that corresponds to the redirection URL specified in `OwnIDConfiguration.plist`. In Xcode, go to **Info > URL Types**, and then use the **URL Schemes** field to specify the redirection URL. For example, if the value of the `OwnIDRedirectionURL` key is `com.myapp.demo://bazco`, then you could copy `com.myapp.demo` and paste it into the **URL Schemes** field.

## Import OwnID Module
Once you have added the OwnID package dependency, you need to import the OwnID module so you can access the SDK features. As you implement OwnID in your project, add the following to your source files:

[Complete example](./DemoApp.swift)
```swift
import OwnIDCoreSDK
```

## Initialize the SDK
The OwnID SDK must be initialized properly using the `configure()` function, preferably in the main entry point of your app (in the `@main` `App` struct). For example, enter:

[Complete example](./DemoApp.swift)
```swift
@main
struct ExampleApp: App {
    init() {
        OwnID.CoreSDK.shared.configure(userFacingSDK: ("CustomIntegrationDemoApp", "0.0.1"), underlyingSDKs: [])
    }
}
```

## Implement the Registration
To make registration workig, you will need to provide your implementation of register process in your system.
In order to achieve this, supply register logic by implementing `RegistrationPerformer` protocol.

[Complete example](./RegisterViewModel.swift)
```swift
final class CustomRegistration: RegistrationPerformer {
    func register(configuration: OwnID.FlowsSDK.RegistrationConfiguration, parameters: RegisterParameters) -> AnyPublisher<OperationResult, OwnID.CoreSDK.Error> {
        let ownIdData = configuration.payload.dataContainer
        return //Your register logic goes here
    }
}
```

When the user selects Skip Password, your app waits for events while the user interacts with the OwnID Web App, then calls your function to register the user once they have completed the Skip Password process.

### Customize View Model
The OwnID view that inserts the Skip Password UI is bound to an instance of the OwnID view model. Before modifying your View layer, create an instance of this view model, `OwnID.FlowsSDK.RegisterView.ViewModel`, within your ViewModel layer with `CustomRegistration` you created earlier:

[Complete example](./RegisterViewModel.swift)
```swift
final class MyRegisterViewModel: ObservableObject {
    // MARK: OwnID
    let ownIDViewModel = OwnID.FlowsSDK.RegisterView.ViewModel(registrationPerformer: CustomRegistration(),
                                                               loginPerformer: CustomLoginPerformer(),
                                                               sdkConfigurationName: clientName,
                                                               webLanguages: languages)
}
```

After creating this OwnID view model, your View Model layer should listen to events from the OwnID Event Publisher, which allows your app to know what actions to take based on the user's interaction with the OwnID Web App. Simply add the following to your existing ViewModel layer to subscribe to the OwnID Event Publisher and respond to events (it can be placed just after the code that creates the OwnID view model instance).

[Complete example](./RegisterViewModel.swift)
```swift
final class MyRegisterViewModel: ObservableObject {
    // MARK: OwnID
    let ownIDViewModel = OwnID.FlowsSDK.RegisterView.ViewModel(registrationPerformer: CustomRegistration(),
                                                               loginPerformer: CustomLoginPerformer(),
                                                               sdkConfigurationName: clientName,
                                                               webLanguages: languages)

    init() {
     subscribe(to: ownIDViewModel.eventPublisher)
    }

     func subscribe(to eventsPublisher: OwnID.RegistrationPublisher) {
       eventsPublisher
           .sink { [unowned self] event in
               switch event {
               case .success(let event):
                   switch event {
                   // Event when user successfully
                   // finishes Skip Password
                   // in OwnID Web App
                   case .readyToRegister:
                     // If needed, ask user to enter
                     // email (mandatory) and call
                     // OwnID.FlowsSDK.RegisterView.ViewModel.
                     // register(with email: String)
                     // to finish registration.
                     // This will prepare data and
                     // call your implementation of registration
                     ownIDViewModel.register(with: email)

                   // Event when OwnID creates
                   // account account in your system
                   // and logs in user
                   case .userRegisteredAndLoggedIn:
                     // User is registered and logged in with OwnID

                   case .resetTapped:
		                 // User tapped activeted button. Rest any data if
		                 // needed. 

                   case .loading:
                     // Display loading indicator according to your designs
                     
                   }

               case .failure(let error):
                // Handle OwnID.CoreSDK.Error here
               }
           }
           .store(in: &bag)
   }
}
```

**Important:** The OwnID `ownIDViewModel.register` function must be called in response to the `.readyToRegister` event. 

### Add the OwnID View
Inserting the OwnID view into your View layer results in the Skip Password option appearing in your app. When the user selects Skip Password, the SDK opens a sheet to interact with the user. The code that creates this view accepts the OwnID view model as its argument. It is suggested that you pass user's email binding for properly creating accounts.

It is reccomended to set height of button the same as text field and disable text field when OwnID is enabled. 

[Complete example](./RegisterView.swift)
```swift
//Put RegisterView inside your main view, preferably besides password field
var body: some View {
    OwnID.FlowsSDK.RegisterView(viewModel: ownIDViewModel,
                                 usersEmail: email,
                                 visualConfig: visualLookConfig)
}
```

## Implement the Login Screen
To make login workig, you will need to provide your implementation of login process in your system.
In order to achieve this, supply login logic by implementing `LoginPerformer` protocol.

[Complete example](./LogInViewModel.swift)
```swift
final class CustomLoginPerformer: LoginPerformer {
    func login(payload: OwnID.CoreSDK.Payload,
               email: String) -> AnyPublisher<OperationResult, OwnID.CoreSDK.Error> {
        let ownIdData = payload.dataContainer
        return //Your login logic goes here
    }
}
```

The process of implementing your Login screen is very similar to the one used to implement the Registration screen. When the user selects Skip Password on the Login screen and if the user has previously set up OwnID authentication, allows them to log in with OwnID.

Like the Registration screen, you add Skip Password to your application's Login screen by including an OwnID view. In this case, it is `OwnID.LoginView`. This OwnID view has its own view model, `OwnID.LoginView.ViewModel`.

### Customize View Model
You need to create an instance of the view model, `OwnID.LoginView.ViewModel`, that the OwnID login view uses.

[Complete example](./LogInViewModel.swift)
```swift
final class MyLogInViewModel: ObservableObject {
    // MARK: OwnID
    let ownIDViewModel = OwnID.FlowsSDK.LoginView.ViewModel(loginPerformer: CustomLoginPerformer(),
                                                            sdkConfigurationName: clientName,
                                                            webLanguages: languages)
}
```

After creating this OwnID view model, you should listen to events from the OwnID Event Publisher, which allows your app to know what actions to take based on the user's interaction with the Skip Password option. Simply add the following to subscribe to the OwnID Event Publisher and respond to events.

[Complete example](./LogInViewModel.swift)
```swift
final class MyLogInViewModel: ObservableObject {
    // MARK: OwnID
    let ownIDViewModel = OwnID.FlowsSDK.LoginView.ViewModel(loginPerformer: CustomLoginPerformer(),
                                                            sdkConfigurationName: clientName,
                                                            webLanguages: languages)

 	  init() {
       subscribe(to: ownIDViewModel.eventPublisher)
   	}

     func subscribe(to eventsPublisher: OwnID.LoginPublisher) {
       eventsPublisher
           .sink { [unowned self] event in
               switch event {
               case .success(let event):
                   switch event {
                   // Event when user who previously set up
                   // OwnID logs in with Skip Password
                   case .loggedIn:
                     // User is logged in with OwnID

                   case .loading:
                     // Display loading indicator according to your designs
                   }

               case .failure(let error):
                 // Handle OwnID.CoreSDK.Error here
               }
           }
           .store(in: &bag)
   }
}
```

### Add OwnID View
Inserting the OwnID view into your View layer results in the Skip Password option appearing in your app. When the user selects Skip Password, the SDK opens a sheet to interact with the user. It is recommended that you place the OwnID view, `OwnID.LoginView`, immediately after the password text field. The code that creates this view accepts the OwnID view model as its argument. It is suggested that you pass user's email binding for properly creating accounts.

[Complete example](./LogInView.swift)
```swift
//Put LoginView inside your main view, preferably below password field
var body: some View {
  //...
  // User's email binding `$viewModel.email` is used to display identity
  // name when logging in. Additionally, this email is used to get
  // information if user already has OwnID account
  OwnID.FlowsSDK.LoginView(viewModel: ownIDViewModel,
                                 usersEmail: email,
                                 visualConfig: visualLookConfig())
  //...
}
```

[Complete example](./LogInView.swift)

## Errors
All errors from the SDK have an `OwnID.CoreSDK.Error` type. You can use them, for example, to properly ask the user to perform an action.

Here are some of the possible errors:
[Complete example](https://github.com/OwnID/ownid-core-ios-sdk/blob/master/Core/Sources/Types/CoreError.swift)
```swift
switch error {
case .unsecuredHttpPassed:
    print("unsecuredHttpPassed")

case .notValidRedirectionURLOrNotMatchingFromConfiguration:
    print("notValidRedirectionURLOrNotMatchingFromConfiguration")

case .emailIsInvalid:
    print("emailIsInvalid")

case .flowCancelled:
    print("flowCancelled")

case .statusRequestResponseIsEmpty:
    print("statusRequestResponseIsEmpty")

case .statusRequestFail(underlying: let underlying):
    print("statusRequestFail: \(underlying)")

case .plugin(let pluginError):
    print("plugin: \(pluginError)")
}
```
## Advanced Configuration

### Button Apperance
It is possible to set button visual settings by passing `OwnID.UISDK.VisualLookConfig`.

```swift
let tooltipConfig = OwnID.UISDK.TooltipVisualLookConfig(backgroundColor: .pink,
                                                        borderColor: .accentColor)
let config = OwnID.UISDK.VisualLookConfig(biometryIconColor: .red,
                                          shadowColor: .cyan,
                                          tooltipVisualLookConfig: tooltipConfig)
OwnID.FlowsSDK.LoginView(viewModel: ownIDViewModel,
                         usersEmail: usersEmail,
                         visualConfig: config)
```

### OwnID Web App language

By default, the OwnID Web App is launched with a language TAGs list (well-formed [IETF BCP 47 language tag](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Accept-Language)) based on the device locales set by the user in system. You can override this behavior and set the Web App language list manually by passing languages in an array. Example:

```swift
let languages = OwnID.CoreSDK.Languages.init(rawValue: ["he"]))
OwnID.FlowsSDK.LoginView.ViewModel(loginPerformer: CustomLoginPerformer(),
                                   sdkConfigurationName: clientName,
                                   webLanguages: languages)
```

## Logging
You can enable console logging by calling `OwnID.startDebugConsoleLogger()`.