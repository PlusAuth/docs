---
title: iOS SwiftUI Tutorial
description: This tutorial demonstrates how to add user login, logout, and profile to an iOS SwiftUI application.
topics:
- quickstarts
- native
- login
- user profile
- logout
- ios
- swift
- swiftui
- xcode
contentType: tutorial
useCase: quickstart
type: native
icon: i-devicon-swift
label: iOS SwiftUI
---
This tutorial shows how to use PlusAuth with iOS SwiftUI Application. If you do not have a PlusAuth account, register from [here](https://dashboard.plusauth.com).

::alert{type=warning}
This tutorial demonstrates how to use PlusAuth with `iOS SwiftUI`. If you are looking for `iOS StoryBoard` tutorial, click [here](https://docs.plusauth.com/quickStart/native/ios/storyboard)
::

::alert{type=info}
This tutorial follows the [plusauth-ios-swiftui-starter](https://github.com/PlusAuth/plusauth-ios-swiftui-starter) sample project on Github. You can download and follow the tutorial via the sample project.
::

## Create PlusAuth Client

After you sign up or log in to PlusAuth, you need to create a client to get the necessary configuration keys in the dashboard.
Go to [Clients](https://dashboard.plusauth.com/~clients) and create a client with the type of `Native Application`


## Configure Client
### Get Client Properties

You will need your `Client Id` for interacting with PlusAuth. You can retrieve them from the created client's details.

### Configure Redirect and Logout URIs
When PlusAuth authenticates a user, it needs a URI to redirect back. That URI must be in your client's `Redirect URI` 
list. If your application uses a redirect URI that is not white-listed in your PlusAuth Client, you will receive an error.

The same thing applies to the logout URIs. After the user logs out, you need a URI to be redirected.


::alert{type=info}
If you are following the sample project, the Redirect URL you need to add to the Redirect URIs field is *`com.plusauth.iosexample.plusauth-ios-starter:/oauth2redirect/ios-provider`*.
The Logout URL you need to add to the Post Logout Redirect URIs field is the same as the Redirect URI field that is *`com.plusauth.iosexample.plusauth-ios-starter:/oauth2redirect/ios-provider`*.
::

## Configure iOS Application to use PlusAuth
Create an iOS application with SwiftUI or download the sample project from the link on the top of the page.

### Add Dependencies 

To get started, install the [AppAuth for iOS](https://openid.github.io/AppAuth-iOS/) SDK.

#### CocoaPods

```sh
pod 'AppAuth'
```

#### Carthage

```sh
github "openid/AppAuth-iOS" "master"
```

#### Swift Package Manager

```swift
dependencies: [
    .package(url: "https://github.com/openid/AppAuth-iOS.git", .upToNextMajor(from: "1.4.0"))
]
```

::alert{type=info}

The `plusauth-ios-swiftui-starter` project uses Swift Package Manager as a dependency manager. Xcode will download the required SDKs automatically after opening the project.
::

### Create the .plist File

Create the `PlusAuth.plist` file at the root of your module with the following and modify values accordingly.

```xml
<!-- plusauth-ios-starter/PlusAuth.plist -->

<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>credentials</key>
	<dict>
		<key>clientId</key>
		<string>CLIENT_ID</string>
		<key>issuer</key>
		<string>https://TENANT_ID.plusauth.com</string>
	</dict>
</dict>
</plist>
```

::alert{type=info}

If you are following the sample project, rename `PlusAuth.example.plist` to `PlusAuth.plist` and replace the values accordingly.
::

### Configure AppAuth Client

We need to read **Client Id** and **Issuer** from `PlusAuth.plist` file before discover the PlusAuth configuration.

```swift
// plusauth-ios-starter/ContentView.swift
import AppAuth

// PlusAuth.plist properties objects
struct PlusAuth : Decodable {
    let clientId, issuer : String
}

struct Root : Decodable {
    let credentials : PlusAuth
}

private var plusAuthCredentials = PlusAuth(clientId: "", issuer: "")

// Read clientId and issuer from PlusAuth.plist file
func readConfiguration(){
  let url = Bundle.main.url(forResource: "PlusAuth", withExtension:"plist")!
  do {
    let data = try Data(contentsOf: url)
    let result = try PropertyListDecoder().decode(Root.self, from: data)
    plusAuthCredentials = result.credentials
  } catch {
    print(error)
  }
}
```

After reading the required configuration, we need to discover PlusAuth endpoints.

```swift
// plusauth-ios-starter/ContentView.swift

private var config: OIDServiceConfiguration?

 // MARK: PlusAuth Methods
func discoverConfiguration() {
  guard let issuerUrl = URL(string: plusAuthCredentials.issuer) else {
    print("Error creating URL for : \(plusAuthCredentials.issuer)")
    return
  }
  // Get PlusAuth auth endpoints
  OIDAuthorizationService.discoverConfiguration(forIssuer: issuerUrl) { configuration, error in
    if(error != nil) {
      print("Error: \(error?.localizedDescription ?? "DEFAULT_ERROR")")
    } else {
      config = configuration
    }
  }
}
```

::alert{type=info}

We will be using the `config` object in the `login` and `logout` actions.
::

Finally call the `readConfiguration` and `discoverConfiguration` methods in `init`.

```swift
// plusauth-ios-starter/ContentView.swift

struct ContentView: View {

   //...

  init() {
    readCredentials()
    discoverConfiguration()
  }

  //...
}
```

### Configure AppDelegate to hold the session

You need to have a property in your `UIApplicationDelegate` implementation to hold the session. In this tutorial, the implementation of this delegate is a class named `AppDelegate`. If your app's application delegate has a different name, please update the class name in the samples below accordingly.

```swift
// plusauth-ios-starter/AppDelegate.swift

import AppAuth

class AppDelegate: UIResponder, UIApplicationDelegate {
  var currentAuthorizationFlow: OIDExternalUserAgentSession?

  //...

  func application(_ app: UIApplication, open url: URL, options: [UIApplication.OpenURLOptionsKey : Any] = [:]) -> Bool {

    if let authorizationFlow = self.currentAuthorizationFlow, authorizationFlow.resumeExternalUserAgentFlow(with: url) {
      self.currentAuthorizationFlow = nil
      return true
    }

    return false
  }

}
```

## Implement login, user profile, and logout

Add the following sections to your `ContentView.swift` file. We will store the retrieved `state` in local storage to exchange with new tokens.

### Add Login 

We will create an `OIDAuthorizationRequest` object to authenticate the user. The application redirects you to the `PlusAuth` login page using the browser.

Add the following section to start an authentication process.

```swift
// plusauth-ios-starter/ContentView.swift

func login() {
  // Get current view
  let windowScene = UIApplication.shared.connectedScenes.first as? UIWindowScene
  let presentingView = windowScene?.windows.first?.rootViewController

  // Create redirectURI from redirectURL string
  guard let redirectURI = URL(string: redirectUrl) else {
    print("Error creating URL for : \(redirectUrl)")
    return
  }
  
  // Create login request
  let request = OIDAuthorizationRequest(configuration: config!, clientId: plusAuthCredentials.clientId, 
                  clientSecret: nil,  scopes: ["openid", "profile", "offline_access"], 
                  redirectURL: redirectURI, responseType: OIDResponseTypeCode,  additionalParameters: nil)
  // performs authentication request
  appDelegate.currentAuthorizationFlow = OIDAuthState.authState(byPresenting: request, presenting: presentingView!) 
  { (authState, error) in
    if let authState = authState {
      setAuthState(state: authState)
      // Save the current state to local storage
      saveState()
    } else {
      print("Authorization error: \(error?.localizedDescription ?? "DEFAULT_ERROR")")
    }
  }
}
```

### Add Logout

We will create a `OIDEndSessionRequest` object to log out the user. Add the following section to log out from the application and PlusAuth:

```swift
// plusauth-ios-starter/ContentView.swift

func logout() {
  // Get current view
  let windowScene = UIApplication.shared.connectedScenes.first as? UIWindowScene
  let presentingView = windowScene?.windows.first?.rootViewController

  // Create redirectURI from redirectURL string
  guard let redirectURI = URL(string: redirectUrl) else {
    print("Error creating URL for : \(redirectUrl)")
    return
  }
  
  guard let idToken = authState?.lastTokenResponse?.idToken else { return }
  // Create logout request
  let request = OIDEndSessionRequest(configuration: config!, idTokenHint: idToken, postLogoutRedirectURL: redirectURI, additionalParameters: nil)
  guard let userAgent = OIDExternalUserAgentIOS(presenting: presentingView!) else { return }

  // performs logout request
  appDelegate.currentAuthorizationFlow = OIDAuthorizationService.present(request, externalUserAgent: userAgent, callback: { (_, error) in
    setAuthState(state: nil)
    // Save the current state to local storage
    saveState()
    viewModel.username = "-"
    viewModel.profileInfo = "-"
  })
}
```

### Get Authenticated User Info 

You can get authenticated user info by adding the following section:

```swift
// plusauth-ios-starter/ContentView.swift

@Published var username = "-"
@Published var profileInfo = "-"

// Get authenticaed user info from PlusAuth
func fetchUserInfo() {
  guard let userinfoEndpoint = authState?.lastAuthorizationResponse.request.configuration.discoveryDocument?.userinfoEndpoint else {
    print("Userinfo endpoint not declared in discovery document")
    return
  }

  let currentAccessToken: String? = authState?.lastTokenResponse?.accessToken

  authState?.performAction() { (accessToken, idToken, error) in
    if error != nil  {
      print("Error fetching fresh tokens: \(error?.localizedDescription ?? "ERROR")")
      return
    }
    guard let accessToken = accessToken else {
      print("Error getting accessToken")
      return
    }
    if currentAccessToken != accessToken {
      print("Access token was refreshed automatically (\(currentAccessToken ?? "CURRENT_ACCESS_TOKEN") to \(accessToken))")
    } else {
      print("Access token was fresh and not updated \(accessToken)")
    }

    var urlRequest = URLRequest(url: userinfoEndpoint)
    // Add access_token to header
    urlRequest.allHTTPHeaderFields = ["Authorization":"Bearer \(accessToken)"]

    let task = URLSession.shared.dataTask(with: urlRequest) { data, response, error in

      DispatchQueue.main.async {
        guard error == nil else {
          print("HTTP request failed \(error?.localizedDescription ?? "ERROR")")
          return
        }
        guard let response = response as? HTTPURLResponse else {
          print("Non-HTTP response")
          return
        }
        guard let data = data else {
          print("HTTP response data is empty")
          return
        }

        var json: [AnyHashable: Any]?
        do {
          json = try JSONSerialization.jsonObject(with: data, options: []) as? [String: Any]
        } catch {
          print("JSON Serialization Error")
        }

        if response.statusCode != 200 {
          let responseText: String? = String(data: data, encoding: String.Encoding.utf8)
          if response.statusCode == 401 {
            let oauthError = OIDErrorUtilities.resourceServerAuthorizationError(withCode: 0, errorResponse: json, underlyingError: error)
            authState?.update(withAuthorizationError: oauthError)
            print("Authorization Error (\(oauthError)). Response: \(responseText ?? "RESPONSE_TEXT")")
          } else {
            print("HTTP: \(response.statusCode), Response: \(responseText ?? "RESPONSE_TEXT")")
          }
          return
        }
        // Create profile info string
        if let json = json {
          viewModel.username = json["username"] as! String
          viewModel.profileInfo = ""
          for (key, value) in json {
            viewModel.profileInfo += "\(key): \(value is NSNull ? "-" : value), "
          }
        }
      }
    }
    task.resume()
  }
}
```

### Add Helper Methods

Set the current `authState` data after login and logout operations.

```swift
// plusauth-ios-starter/ContentView.swift

private var authState: OIDAuthState?

@Published var isLoggedIn: Bool = false

// Set user auth state
func setAuthState(state: OIDAuthState?) {
  if (authState == state) {
    return;
  }
  authState = state;
  viewModel.isLoggedIn = state?.isAuthorized == true
}
```

After successful login or logout, store the current `authState` to use later.

```swift
// plusauth-ios-starter/ContentView.swift

private let plusAuthStateKey: String = "authState";
private let storageSuitName = "com.plusauth.iosexample"

 // Save user state to local
func saveState() {
  guard let data = try? NSKeyedArchiver.archivedData(withRootObject: authState as Any, requiringSecureCoding: true) else {
    return
  }
  
  if let userDefaults = UserDefaults(suiteName: storageSuitName) {
    userDefaults.set(data, forKey: plusAuthStateKey)
    userDefaults.synchronize()
  }
}
```

Load the stored state data in the `init` method to get authenticated user data.

```swift
// plusauth-ios-starter/ContentView.swift

// Load local state info if exists
func loadState() {
  guard let data = UserDefaults(suiteName: storageSuitName)?.object(forKey: plusAuthStateKey) as? Data else {
    return
  }
  do {
    let authState = try NSKeyedUnarchiver.unarchiveTopLevelObjectWithData(data) as? OIDAuthState
    self.setAuthState(state: authState)
    // Fetch user info if user authenticated
    fetchUserInfo()
  } catch {
    print(error)
  }
}
```

Finally, create your UI in the `body` section and set the `@Published` variables to your views to see the changes accordingly.

## See it in action

That's it. Start your app and point to your device or emulator. Follow the **Log In** link to log in or sign up to your PlusAuth tenant. After you click the `Login` button, the browser launches and shows the PlusAuth Login page. Upon successful login or signup, you should be redirected back to the application.
