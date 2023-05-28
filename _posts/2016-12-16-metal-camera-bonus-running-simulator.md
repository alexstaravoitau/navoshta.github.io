---
title: Metal Camera Tutorial Bonus&#58; Running Metal project in iOS Simulator
tags:
- iOS
- Swift
- Metal
- Unit Tests
- Tutorial
crosspost_to_medium: true
author_profile: false
sidebar:
    nav: metal-camera-tutorial
---
In the Metal Camera Tutorial series we have created a simple app that renders camera frames on screen in real time. However, this app uses `Metal` framework, which is not available in iOS Simulator. Basically, your app won't even build if you select simulator as a build device, which is a shame in case you want to add unit tests for example, being able to run them without actual device connected to your machine. <!--more--> 

Some parts of `Metal` have stub implementations for desktop processor architectures, which means you can at least build the app (only to find out it's not working as expected on the simulator). Other parts, like `MetalKit` don't even exist for Simulator, so you will have to wrap the imports into conditional compilation blocks, like that:

```swift
import UIKit
import Metal

#if arch(i386) || arch(x86_64)
#else
    import MetalKit
#endif
```

We check if current processor architecture is a desktop one, and in this case simply don't import any ARM-only frameworks. Beware that the classes from `MetalKit` are not available either, so you will need to wrap any code using them too:

```swift
public class MTKViewController: UIViewController {

#if arch(i386) || arch(x86_64)
#else
    /// `UIViewController`'s view
    private var metalView: MTKView!
#endif

    // MARK: - Public overrides
    
    override public func loadView() {
        super.loadView()
#if arch(i386) || arch(x86_64)
        NSLog("Failed creating a default system Metal device, since Metal is not available in iOS Simulator.")
#else
        assert(device != nil, "Failed creating a default system Metal device. Please, make sure Metal is available on your hardware.")
#endif
        initializeMetalView()
        initializeRenderPipelineState()
    }
    
    // MARK: - Private Metal-related properties and methods
    
    /**
     initializes and configures the `MTKView` we use as `UIViewController`'s view.
     
     */
    private func initializeMetalView() {
#if arch(i386) || arch(x86_64)
#else
        metalView = MTKView(frame: view.bounds, device: device)
        metalView.delegate = self
        metalView.framebufferOnly = true
        metalView.colorPixelFormat = .BGRA8Unorm
        metalView.contentScaleFactor = UIScreen.mainScreen().scale
        metalView.autoresizingMask = [.FlexibleWidth, .FlexibleHeight]
        view.insertSubview(metalView, atIndex: 0)
#endif
    }
}
```

With those compilation time checks you clearly won't get your app running normally in iOS Simulator. However, it will let you build it, and, for example, cover the code not requring `Metal` with unit tests. You could add a stubbed camera session to verify camera permissions and delegates, for instance — well, you get the idea.

## Where do I go from here?

This was the bonus part of **Metal Camera Tutorial** series, where we explore ways of achieving lowest-overhead access to hardware to grab camera frames, convert them to textures and render on screen in real time:

* <a target="_blank" href="/metal-camera-part-1-camera-session">**Part 1: Getting raw camera data**</a>
* <a target="_blank" href="/metal-camera-part-2-metal-texture">**Part 2: Converting sample buffer to a Metal texture**</a>
* <a target="_blank" href="/metal-camera-part-3-render-shader">**Part 3: Rendering a Metal texture**</a>
* **Bonus: Running Metal project in iOS Simulator**

You can check out the <a target="_blank" href="https://github.com/alexstaravoitau/MetalRenderCamera">final project</a> from this **Metal Camera Tutorial** on GitHub.

<!-- Place this tag where you want the button to render. -->
<a class="github-button" href="https://github.com/alexstaravoitau" data-style="mega" data-count-href="/navoshta/followers" data-count-api="/users/navoshta#followers" data-count-aria-label="# followers on GitHub" aria-label="Follow @alexstaravoitau on GitHub">Follow @alexstaravoitau</a>
<!-- Place this tag where you want the button to render. -->
<a class="github-button" href="https://github.com/alexstaravoitau/MetalRenderCamera" data-icon="octicon-star" data-style="mega" data-count-href="/navoshta/MetalRenderCamera/stargazers" data-count-api="/repos/navoshta/MetalRenderCamera#stargazers_count" data-count-aria-label="# stargazers on GitHub" aria-label="Star navoshta/MetalRenderCamera on GitHub">Star</a>
<!-- Place this tag where you want the button to render. -->
<a class="github-button" href="https://github.com/alexstaravoitau/MetalRenderCamera/fork" data-icon="octicon-repo-forked" data-style="mega" data-count-href="/navoshta/MetalRenderCamera/network" data-count-api="/repos/navoshta/MetalRenderCamera#forks_count" data-count-aria-label="# forks on GitHub" aria-label="Fork navoshta/MetalRenderCamera on GitHub">Fork</a>
<!-- Place this tag where you want the button to render. -->
<a class="github-button" href="https://github.com/alexstaravoitau/MetalRenderCamera/archive/master.zip" data-icon="octicon-cloud-download" data-style="mega" aria-label="Download navoshta/MetalRenderCamera on GitHub">Download</a>

<!-- Place this tag in your head or just before your close body tag. -->
<script async defer src="https://buttons.github.io/buttons.js"></script>
