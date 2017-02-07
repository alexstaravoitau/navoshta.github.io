---
title: Metal Camera Tutorial Part 2&#58; Converting sample buffer to a Metal texture
tags:
- iOS
- Swift
- Camera
- Metal
crosspost_to_medium: true
---
In the <a target="_blank" href="/metal-camera-part-1-camera-session/">first part</a> of Metal Camera Tutorial series we managed to fire up a session that would continuously send us frames from device's camera via a delegate callback. Now, this is already pretty exciting, but we need to get hold of actual textures to do something useful with it — and we are going to use `Metal` for that. <!--more-->

## What is `Metal`?

`Metal` is a relatively new Apple graphics framework backed up by the iOS device's GPU. However it's not just graphics: GPUs recently got a whole new world of applications in numeric computations due to a rise of neural networks and other machine learning algorithms. That is why Apple has extended `Metal` APIs in iOS 10 SDK to cater for various machine learning applications as well, claiming to provide the lowest-overhead access to the GPU (as opposed to `OpenGL`), so the whole thing sounds very promising.

## What do I do with it again?

Anyway, for now let's stick with the graphics side of `Metal`, and try to convert camera frames to a sequence of `Metal` textures. So the plan would be as follows:

* **Get each frame data**. We are going to grab it from `CMSampleBuffer`.
* **Convert frame to a `Metal` texture**. `Metal` has its own class for textures — `MTLTexture`.

## Just give me the code man.

Ok, let's start with getting frame's data.

### Get frame data

If you followed through the Camera Session tutorial, you should know how to get hold of the `CMSampleBuffer` — a `Core Foundation` object representing a generic container for media data. Now, there are a couple of other `Core Foundation` methods to grab frame data from it. The first step would be to get a `CVImageBuffer` from `CMSampleBuffer`, `CVImageBuffer` being a more specific container for image data. This one is relatively easy, since there is a `CMSampleBufferGetImageBuffer` function:

```swift
/// Assuming you have the CMSampleBuffer in sampleBuffer variable

guard let imageBuffer = CMSampleBufferGetImageBuffer(sampleBuffer) else {
    /// Handle an error. We failed to get image buffer.
}	

/// Ok, we got our image buffer now.	
```

### Convert frame to a `Metal` texture

This part is slightly more tricky, since we need to know which pixel format we are using. Remember last time I've mentioned you have two options: either use `RGB` or `YCbCr`? Well, now our steps would be slightly different for each of those two formats, since in case of `RGB` iOS will do a bit more work and provide you with a *joint single **RGB** texture* in the sample buffer, whereas for `YCbCr`, being a hardware native format, it's not doing any extra effort and provides you the camera data as is, which is actually *two textures*: one would hold **Y** pixel component, and the other one the **CbCr** component. For the sake of simplicity we are going to assume that you use `RGB` color space, although final project on GitHub supports both pixel formats.

Since things start turning `Metal` at this point, we will need a couple of variables specific to this framework: metal device and texture cache. Metal device is a `Metal` object representing your GPU hardware. Texture cache is a container we are going to use when converting frame data to `MTLTexture`s.

```swift
/// Texture cache we will use for converting frame images to textures
var textureCache: CVMetalTextureCache?

/// `MTLDevice` we need to initialize texture cache
var metalDevice = MTLCreateSystemDefaultDevice()
```

We need to initialise the cache first (and make sure to do this once, as we are going to reuse this container for each and every frame).

```swift
guard let
    metalDevice = metalDevice
    where CVMetalTextureCacheCreate(kCFAllocatorDefault, nil, metalDevice, nil, &textureCache) == kCVReturnSuccess
else {
    /// Handle an error, as we failed to create texture cache
}

/// We have our texture cache now.
```

With this out of our way it get pretty straightforward. We first get frame's dimensions:

```swift
let width = CVPixelBufferGetWidth(imageBuffer)
let height = CVPixelBufferGetHeight(imageBuffer)

```

Now get an unmanaged reference to a `CVMetalTexture`. This is not `MTLTexture` yet, but we are getting there! 

> Previously things were getting a bit `Unmanaged` at this point, as we were entering a world of Objective-C APIs in our purely Swift code. To get a better insight of why we use `Unmanaged` and to make sure you don't shoot yourself in the leg with it, you may want to read <a target="_blank" href="http://nshipster.com/unmanaged/">this great article</a>. 

> But ever since Swift 3 was introduced, `CoreVideo` APIs seemed to be updated, making the `Unmanaged` part redundant. It means that now you don't have to worry about specifying a `Unmanaged<CVMetalTextureRef>?` type — you can simply use `CVMetalTextureRef?`.

```swift
var imageTexture: CVMetalTexture?

let result = CVMetalTextureCacheCreateTextureFromImage(kCFAllocatorDefault, textureCache.takeUnretainedValue(), imageBuffer, nil, pixelFormat, width, height, planeIndex, &imageTexture)
```

Ok, almost there. Now we only need to grab the actual texture from the CVMetalTexture container and make sure to manually release the unwrapped optional texture reference.

```swift
guard
    let unwrappedImageTexture = imageTexture,
    let texture = CVMetalTextureGetTexture(unwrappedImageTexture),
    result == kCVReturnSuccess
else {
    throw MetalCameraSessionError.failedToCreateTextureFromImage
}

/// We have our `MTLTexture` in the `texture` variable now.
```

## That's it? What do I do with that texture?

We now have the frame image data as `MTLTexture`, and can use `Metal` APIs to do something fancy with it. Next time, for instance, we are going to explore a way to render this texture on screen using a simple `Metal` shader.

## Where do I go from here?

This was the second part of **Metal Camera Tutorial** series, where we explore ways of achieving lowest-overhead access to hardware to grab camera frames, convert them to textures and render on screen in real time:

* <a target="_blank" href="/metal-camera-part-1-camera-session">**Part 1: Getting raw camera data**</a>
* **Part 2: Converting sample buffer to a Metal texture**
* <a target="_blank" href="/metal-camera-part-3-render-shader">**Part 3: Rendering a Metal texture**</a>
* <a target="_blank" href="/metal-camera-bonus-running-simulator">**Bonus: Running Metal project on iOS Simulator**</a>

You can check out the <a target="_blank" href="https://github.com/navoshta/MetalRenderCamera">final project</a> from this **Metal Camera Tutorial** on GitHub.


<!-- Place this tag where you want the button to render. -->
<a class="github-button" href="https://github.com/navoshta" data-style="mega" data-count-href="/navoshta/followers" data-count-api="/users/navoshta#followers" data-count-aria-label="# followers on GitHub" aria-label="Follow @navoshta on GitHub">Follow @navoshta</a>
<!-- Place this tag where you want the button to render. -->
<a class="github-button" href="https://github.com/navoshta/MetalRenderCamera" data-icon="octicon-star" data-style="mega" data-count-href="/navoshta/MetalRenderCamera/stargazers" data-count-api="/repos/navoshta/MetalRenderCamera#stargazers_count" data-count-aria-label="# stargazers on GitHub" aria-label="Star navoshta/MetalRenderCamera on GitHub">Star</a>
<!-- Place this tag where you want the button to render. -->
<a class="github-button" href="https://github.com/navoshta/MetalRenderCamera/fork" data-icon="octicon-repo-forked" data-style="mega" data-count-href="/navoshta/MetalRenderCamera/network" data-count-api="/repos/navoshta/MetalRenderCamera#forks_count" data-count-aria-label="# forks on GitHub" aria-label="Fork navoshta/MetalRenderCamera on GitHub">Fork</a>
<!-- Place this tag where you want the button to render. -->
<a class="github-button" href="https://github.com/navoshta/MetalRenderCamera/archive/master.zip" data-icon="octicon-cloud-download" data-style="mega" aria-label="Download navoshta/MetalRenderCamera on GitHub">Download</a>

<!-- Place this tag in your head or just before your close body tag. -->
<script async defer src="https://buttons.github.io/buttons.js"></script>

