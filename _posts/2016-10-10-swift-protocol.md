---
title: Type of a class conforming to protocol
tags:
- iOS
- Swift
- Protocol
---
Although protocols are not by any means a new thing, Swift specifically encourages the developers to use it over inheritance. Not that Objective-C didn't make use of protocols, but due to the dynamic nature of Objective-C Runtime one would be tempted to put chunks of common declarations in a superclass instead.<!--more-->

Swift is somewhat less forgiving in that sense, and developers (including me) tend to start realising that sometimes a protocol would be a better fit. There are still some contraversial use cases, one of them being: how do I declare a variable in Swift being of a specific class and conforming to a protocol? This is actually quite common: say, you expect a `var` to hold a reference to a `UIViewController` (so that you could do some UI presentation stuff with it) that should also conform to a protocol (so that it does some more stuff on top of that). For instance, here is the protocol:

```swift
protocol Wibbling {
	func wibble()
}
```

And here is a class implementing this protocol:

```swift
class WibblingViewController: UIViewController, Wibbling {
	func wibble() {
		// Do the actual wibbling
	}
}
```

Now, imagine you need to declare a `var` that should be both a `UIViewController` and conform to `Wibbling` (but not neccessarilly a `WibblingViewController`). Most likely you are going to face some ugly type casting, but there is another way of handling this, which I personally prefer: add another protocol providing a `UIViewController` instance, and extend it with `Wibbling`:

```swift
/// Specifies behaviour of an object presentable within the application UI.
protocol UIPresentable: class {

    /// View controller to be added to the UI hierarchy.
    var viewController: UIViewController { get }
}

protocol Wibbling: UIPresentable {
	
	func wibble()
}
```

Now, since you probably only intend to use `Wibbling` with a `UIViewController`, you may want to provide a default implementation: 

```swift
// Default implementation returning `self` as `UIViewController`.
extension UIPresentable {

    var viewController: UIViewController {
        return self as! UIViewController
    }
}
```

With this extension in place you don't even need to alter implementation of the `WibblingViewController`. You can now treat the `var` as conforming to `Wibbling` and easily get hold of the view controller:

```swift
var wibbler: Wibbling
wibbler = WibblingViewController()
present(wibbler.viewController, animated: true) {
    wibbler.wibble()
}
```
