# Today I Learn

In this series, I will write about mistakes I have made during my first year working full time as an iOS developer.
This covers wrong design choices, misunderstanding of the iOS SDK, poor engineering practice, etc.
Also, I will consider writing tutorials/explanation for things I have done (successfully).

## Livestreaming application
The first serious application I create is a livestream broadcast application.
In the application, we want to let users rotate freely before they begin to go live.
In other words, when you hit 'go live' button, the camera's `UIViewController` orientation should be locked.

When I first work with Apple's iOS SDK, I did not pay close attention to the documentation, as a result, it took a tremendous effort to achieve the behavior I expected.
After I realized I was using the SDK wrongly, I felt like I had reinvented the wheel, only much worse in quality.

1. Overriding `AppDelegate`'s `application(_:supportedInterfaceOrientationsFor:)` method with a static class:


```swift
class AppDelegate {
    func application(UIApplication, supportedInterfaceOrientationsFor: UIWindow?) -> UIInterfaceOrientationMask {
        return OrientationHelper.orientations
    }
}

class OrientationHelper {
    private(set) static var orientations: UIInterfaceOrientationMask

    func rotate(to orientation: UIInterfaceOrientation, using orientationMask: UIInterfaceOrientationMask) {
        self.orientations = orientationMask
        UIDevice.current.setValue(orientation.rawValue, forKey: "orientation")
    }
}
```

The snippet below was shamelessly copied into my project without careful inspection, but it was not correct.
If you have implemented the rotation like this, you have ran into the same mistake I made when I first implement this functionality...

If you look at the [documentation for UIDevice class](https://developer.apple.com/documentation/uikit/uidevice), you can see that the variable associated with key 'orientation' is a `UIDeviceOrientation`, while the orientation value we receive in `rotate(to:using:)` is `UIInterfaceOrientation`.
I was lucky (or unlucky because the problem did not surface earlier) that the 2 portrait orientations shares the same `rawValue`, and the 2 landscape orientations `rawValue` are still landscape.
However, the 2 landscape orientations `rawValue` is flipped.
As a result, when a client using `rotate(to:using:)` request to rotate to `landscapeLeft`, this method actually rotates the device to `landscapeRight`.
The more correct implementation of `rotate(to:using:)` would look like this:

```swift
class OrientationHelper {
    private(set) static var orientations: UIInterfaceOrientationMask

    func rotate(to orientation: UIInterfaceOrientation, using orientationMask: UIInterfaceOrientationMask) {
        
        let deviceOrientation: UIDeviceOrientation

        switch orientation {
            case .portrait: 
                deviceOrientation = .portrait
            case .landscapeLeft:
                deviceOrientation = .landscapeLeft
            case .landscapeRight:
                deviceOrientation = .landscapeRight
            default:
                // In my application, I dont allow portraitUpsideDown
                deviceOrientation = .portrait
        }

        self.orientations = orientationMask
        UIDevice.current.setValue(deviceOrientation.rawValue, forKey: "orientation")
    }
}
```

2. Call rotate and lock orientation to current orientation whenever the user gone live
3. Call rotate and lock orientation to native supported orientation when the user ends

It all went well,... for a while...  
Sometimes the interface would not rotate...

In particular, when I hold the device horizontally (while the interface is locked in portrait), and call rotate to landscape programmatically).
I did not realize the cause of the problem until much later during my first year.
As it turns out, when we call `UIDevice.setValue` for key `orientation`, the OS checks the current orientation of the device first, if the target orientation is the same as the requested orientation, it will not rotate.
The device does not undergo a rotation event, so the interface did not rotate and did not receive a `viewWillTransition(to:with:)`...
To force the rotation of the viewcontroller, we have to call a static method `UIViewController.attemptRotationToDeviceOrientation`.
Finally the `OrientationHelper` method `rotate(to:using:)` looks like this

```swift
class OrientationHelper {
    private(set) static var orientations: UIInterfaceOrientationMask

    func rotate(to orientation: UIInterfaceOrientation, using orientationMask: UIInterfaceOrientationMask) {
        
        let deviceOrientation: UIDeviceOrientation

        switch orientation {
            case .portrait: 
                deviceOrientation = .portrait
            case .landscapeLeft:
                deviceOrientation = .landscapeLeft
            case .landscapeRight:
                deviceOrientation = .landscapeRight
            default:
                // In my application, I dont allow portraitUpsideDown
                deviceOrientation = .portrait
        }

        self.orientations = orientationMask
        UIDevice.current.setValue(deviceOrientation.rawValue, forKey: "orientation")
        UIViewController.attemptRotationToDeviceOrientation()
    }
}
```

Finally, I have arrived at a solution to achieve desired behavior..
But seriously, I have been going through so much more trouble than I should have..
I have reinvented the wheel, a much worse wheel, in fact.
(This implementation should be broken in case Apple decided to change the name for their UIDevice orientation)...
We can instead use the two methods from `UIViewController` class, which are `shouldAutoRotate()` and `supportedInterfaceOrientation()`.
However, to force a rotation (ie. on a button click), we cannot avoid using `UIDevice.setValue(_:forKey:)`...

[Apple's documentation to supported orientation](https://developer.apple.com/documentation/uikit/uiviewcontroller/1621435-supportedinterfaceorientations).
If the link above is not working, try [this link](https://www.google.com/search?q=ios+supported+interface+orientation+apple).


## 1. Orientation

## 2. The Keyboard

## 3. List of items

## 4. Failing fast... and safe

## 5. Protecting the client
