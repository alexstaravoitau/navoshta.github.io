---
title: Metal Camera Tutorial Part 1&#58; Getting raw camera data
tags:
- iOS
- Swift
- Camera
---
A lot of apps nowadays use iPhone and iPad cameras. Some even do pretty badass things with it (performance wise), like running each frame through a neural network or applying a realtime filter. Either way you may want to get as low as you can in terms of the level at which you interact with the device  hardware, be it getting data from a camera sensor or computations involving GPU — you still want to minimise the impact on device's limited computational resources.<!--more--> In this tutorial we will get raw camera frames data. If you are trying to achieve the lowest-overhead access to camera sensor, using a `AVCaptureSession` would be a really good start!

## What is `AVCaptureSession`?

`AVCaptureSession` is a class in `AVFoundation` framework — whenever you write any code for audio/video recording or playback, the chances are you will end up using some parts of `AVFoundation` (AV standing for "Audio/Video"). As Apple reference points out, you use an `AVCaptureSession` object to coordinate the flow of data from audio or video input devices to outputs. So it is supposed to be a pretty generic interface between something generating media data (for example, camera sensor) and something receiving it (say, your app).

## What do I do with it again?

The workflow is pretty straightforward, and basically consists of the following steps:

* **Initialise session**. Session being actual `AVCaptureSession` instance.
* **Request access to hardware** (for instance, the camera) and get a `AVCaptureDevice` instance. This instance represents a single hardware piece — say, a front camera or a back camera.
* **Add input to session**. Input is a `AVCaptureDeviceInput` instance initialised with your `AVCaptureDevice`.
* **Add output to session**. In our case it's going to be a `AVCaptureVideoDataOutput` instance that you need to configure with the format you want your data in.
* **Start the session** and off you go. The data should be streaming now.

## Just give me the code man.

There is a bit more to each of those steps of course, so let's step into more detail.

### Initialise session

Well, this part is pretty easy — you simply initialise an instance.

```swift
let captureSession = AVCaptureSession()
```

### Request access to hardware

We are still on relatively easy side of things with this one. You only need to provide the media type you want to get access to — in out case it's `AVMediaTypeVideo`. This is not an enum by the way, but rather a `String` constant.

```swift
AVCaptureDevice.requestAccessForMediaType(AVMediaTypeVideo) {
    (granted: Bool) -> Void in
    guard granted else {
        /// Report an error. We didn't get access to hardware.
        return
    }
	
    /// All good, access granted.
}
```

Now you need to get a `AVCaptureDevice` instance representing the camera. I would suggest adding a routine method for that.

```swift
func device(mediaType: String, position: AVCaptureDevicePosition) -> AVCaptureDevice? {
    guard let devices = AVCaptureDevice.devices(withMediaType: mediaType) as? [AVCaptureDevice] else { return nil }
    
    if let index = devices.index(where: { $0.position == position }) {
        return devices[index]
    }
    
    return nil
}
```

What this method does is simply gets an `Array` of devices of specified type and then filters out the one with requested `AVCaptureDevicePosition`. So now you can get a `AVCaptureDevice` instance representing the front camera!

```swift
guard let inputDevice = device(mediaType: AVMediaTypeVideo, position: .front) else { 
	/// Handle an error. We couldn't get hold of the requested hardware.
	return 
}

/// Do stuff with your inputDevice!
```

### Add input to session

Here we initialise a `AVCaptureDeviceInput` instance, and try to add it to the camera session.

```swift
var captureInput: AVCaptureDeviceInput!

do {
    captureInput = try AVCaptureDeviceInput(device: inputDevice)
}
catch {
    /// Handle an error. Input device is not available.
}
```

Now, we are going to configure a bunch of the capture session things in the next paragraphs, and we would like it to be all applied atomically. You can do that by wrapping the configuration into `beginConfiguration()`/`commitConfiguration()` calls:

```swift
captureSession.beginConfiguration()

guard captureSession.canAddInput(captureInput) else {
    /// Handle an error. We failed to add an input device.
}

captureSession.addInput(captureInput)
```

### Add output to session

Now it's time to configure the output, e.g. something that you're going to use in your code to actually get hold of the raw camera data. It starts easy:


```swift
let outputData = AVCaptureVideoDataOutput()
```

This would be a good time to configure the output, for instance specify data format. In case of video streaming the big choice you have to make is the color space: you can either use `RGB` or `YCbCr`. You almost certainly heard of `RGB` which stands for **R**ed **G**reen **B**lue, which in its turn means we store three values for each pixel. This is actually quite a bit redundant, which is why there is another `YCbCr` option. 

> YCbCr also stores 3 components, **Y** being the luma component and **CB** and **CR** are the blue-difference and red-difference chroma components. Basically you get a greyscale image as **Y** and the color offsets in those **Cb** and **Cr**. You can get a better insight of the whole thing <a target="_blank" href="https://en.wikipedia.org/wiki/YCbCr">on Wikipedia</a>. 
If you're worried about performance and try to squeeze every bit of the processor juice you may want to use `YCbCr` instead of `RGB` — not only you get smaller textures for each frame, but also operate in hardware native format, which means it won't use any of the device resources to convert output data to `RGB`.

```swift
outputData.videoSettings = [
    kCVPixelBufferPixelFormatTypeKey as AnyHashable : Int(kCVPixelFormatType_420YpCbCr8BiPlanarVideoRange)
]
```

Now it's time to set the delegate that is going to receive each and every video frame. You will also need to specify a dispatch queue:

```swift
let captureSessionQueue = DispatchQueue(label: "CameraSessionQueue", attributes: [])
outputData.setSampleBufferDelegate(self, queue: captureSessionQueue)
```

> You have set `self` as sample buffer delegate, so don't forget to implement the `AVCaptureVideoDataOutputSampleBufferDelegate` protocol's `didOutputSampleBuffer` method. It will be your output callback that will be called for every frame, passing the frame data in a `CMSampleBuffer`.

Finally, add your configured output to the session instance.

```swift
guard captureSession.canAddOutput(outputData) else {
    /// Handle an error. We failed to add an output device.
}

captureSession.addOutput(outputData)
```

### Start session

Ok, we are done with configuration for now! Let's call `commitConfiguration()` and fire up the session.

```swift
captureSession.commitConfiguration()
captureSession.startRunning()
```
Now, if you have configured everything correctly, you should start receiving sample buffers with live front camera video frames in the `AVCaptureVideoDataOutputSampleBufferDelegate` protocol callback. 


```swift
@objc func captureOutput(_ captureOutput: AVCaptureOutput!, didOutputSampleBuffer sampleBuffer: CMSampleBuffer!, from connection: AVCaptureConnection!) {
	/// Do more fancy stuff with sampleBuffer.
}
```

## That's it? What do I do with those sample buffers?

`CMSampleBuffer` is a `Core Foundation` object representing a generic container for media data. It can contain video frames, audio or other media types, as well as context info, like timestamps, playback speed and so on. Normally you extract the data you need before working with it, and in case of video streaming this data would be frame textures. 

Next time we are going to explore a way to grab those frames from a `CMSampleBuffer` and convert it to a texture that we can later use with Apple's GPU-powered framework <a target="_blank" href="https://developer.apple.com/metal/">Metal</a>.

## Where do I go from here?

This was the first part of **Metal Camera Tutorial** series, where we explore ways of achieving lowest-overhead access to hardware to grab camera frames, convert them to textures and render on screen in real time:

* **Part 1: Getting raw camera data**
* <a target="_blank" href="/metal-camera-part-2-metal-texture">**Part 2: Converting sample buffer to a Metal texture**</a>
* **Part 3: Rendering a Metal texture**
* **Bonus: Running Metal project on iOS Simulator**

You can check out the <a target="_blank" href="https://github.com/navoshta/MetalRenderCamera">final project</a> from this **Metal Camera Tutorial** on GitHub.


