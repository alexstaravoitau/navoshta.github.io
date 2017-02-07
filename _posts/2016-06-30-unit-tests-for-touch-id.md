---
title: Unit tests for Touch ID
tags:
- iOS
- Swift
- XCTest
crosspost_to_medium: true
---
Writing unit tests is like having sex in high school: everybody is talking about it, although very few are actually doing it. In the iOS world it had a couple of additional roadblocks for a while due to a lack of solid and stable testing capabilities out of the Xcode's box. However, with the Apple's `XCTest` framework things have improved greatly: you no longer have an excuse of needing 3rd party frameworks to test your code properly. <!--more--> However, there are still things here and there that might not be so straightforward to write unit tests for, and Touch ID is one of them.

Let's assume you have some kind of a manager class that handles Touch ID authentication, say, `TouchIDManager`. Surely it will be using `LAContext` instance from the `LocalAuthentication` framework, at least those two methods:

```swift
func evaluatePolicy(policy: LAPolicy, localizedReason: String, reply: (Bool, NSError?) -> Void)
func canEvaluatePolicy(policy: LAPolicy, error: NSErrorPointer) -> Bool
```

So, first, let's update your `TouchIDManager` implementation, so that you have a single `LAContext` instance that you can easily mock later. Just add a `authenticationContext` variable and use it throughout the class whenever you need to call a `LAContext` method. Mark it as `internal`, so that you can get access to it from within your `XCTestCase` class.

Basically, at this point your Touch ID related class should look something like this:

```swift
final public class TouchIDManager {

    /** 
     Authentication context object we use for Touch ID. 
     Typically it's just a `LAContext` instance that is mocked when testing.
     */
    internal var authenticationContext = LAContext()

    /**
     Checks Touch ID availability.

     - returns: Flag indicating if Touch ID is available
     */
    public func touchIDAvailable() -> Bool {
        return authenticationContext.canEvaluatePolicy(LAPolicy.DeviceOwnerAuthenticationWithBiometrics, error: nil)
    }
	
    /**
     Authenticates the user with Touch ID

     - parameter completion: Completion handler
     */
    public func authenticate(completion: (success: Bool) -> ()) {
        guard touchIDAvailable() else {
            completion(false)
            return
        }
        authenticationContext.evaluatePolicy(LAPolicy.DeviceOwnerAuthenticationWithBiometrics, localizedReason: "Wanna Touch my ID?") {
            (success: Bool, error: NSError?) -> Void in
            completion(success: success)
        }
    }
}
```

Now writing unit test is going to be a piece of cake! All you have to do is to mock the `LAContext` object. Add a new `XCTestCase` subclass and call it, say, `TouchIDManagerTests`. Don't forget all the neccessary imports. You will need the `LocalAuthentication` module, as well as your application's module imported as `@testable` (so that you have access to its `internal` variables):

```swift
import XCTest
import LocalAuthentication
@testable import YourAppModule 
```

Ok, you are all set! You can now write a test that would fake the `LAContext` behaviour, pretending that you have Touch ID available in Simulator and that the user's identity was successfully verified.

```swift
func testSuccessfulAuthentication() {
    /// A class faking Touch ID availability and successful user verification
    class StubLAContext: LAContext {
        override func evaluatePolicy(policy: LAPolicy, localizedReason: String, reply: (Bool, NSError?) -> Void) { reply(true, nil) }
        override func canEvaluatePolicy(policy: LAPolicy, error: NSErrorPointer) -> Bool { return true }
    }

    let manager = TouchIDManager()
    manager.authenticationContext = StubLAContext()
    
    manager.authenticate { (success) in
        XCTAssertTrue(success)
    }
}
```
Clearly this particular test doesn't do much, as it only verifies the stubbed output. But no doubts you have much more going on in your own TouchIDManager class, and this may be a good starting point to test this logic.
