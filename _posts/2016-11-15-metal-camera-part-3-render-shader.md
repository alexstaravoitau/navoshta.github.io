---
title: Metal Camera Tutorial Part 3&#58; Rendering a Metal texture
tags:
- iOS
- Swift
- Metal
- Tutorial
crosspost_to_medium: true
author_profile: false
sidebar:
    nav: metal-camera-tutorial
---
In the <a target="_blank" href="/metal-camera-part-2-metal-texture">second part</a> of the Metal Camera Tutorial series we managed to convert frame data to a `Metal` texture. Now we are going to render it on screen with the help of a very simple `Metal` shader. <!--more-->

## What is a `Metal` shader?

Metal shader is a function written in a special *Metal shading language* that `Metal` framework executes on a very low level of the GPU hardware. It can either do some rendering stuff, or perform more generic computations.

## What do I do with it again?

You can do quite a lot of stuff with it! As I mentioned before, `Metal` is a very promising framework as GPU computations have a lot of potential applications nowadays. We are going to start with simply rendering an arbitrary `MTLTexture` on screen. Let's create a special `UIViewController` that will handle `Metal` shader execution. This shader will simply render a texture in the `UIViewController`'s view. The main steps would be as follows:

* **Initialise `MTKView`**, that we will use for rendering a texture.
* **Implement shaders** in Metal shading language.
* **Do the drawing!** Well, actually, it's not as easy as that. In order to instantiate a graphics rendering process, you are going to need three important things: *pipeline state*, *command encoder* and *command buffer*, with three protocols in `Metal` framework that define behaviour of each: <a target="_blank" href="https://developer.apple.com/library/ios/documentation/Metal/Reference/MTLRenderPipelineState_Ref/">`MTLRenderPipelineState`</a>, <a target="_blank" href="https://developer.apple.com/library/ios/documentation/Metal/Reference/MTLRenderCommandEncoder_Ref/index.html#//apple_ref/occ/intf/MTLRenderCommandEncoder">`MTLRenderCommandEncoder`</a> and <a target="_blank" href="https://developer.apple.com/library/ios/documentation/Metal/Reference/MTLCommandBuffer_Ref/index.html#//apple_ref/occ/intf/MTLCommandBuffer">`MTLCommandBuffer`</a>. So a command buffer is responsible for accumulating and storing a sequence of encoded commands that are eventually committed for execution by the GPU. Once created a command buffer, you then instantiate an encoder object, which would be an interface for you to specify those commands and fill the buffer with them, where pipeline state is going to define the shaders you are going to use in those commands. So the rendering process (in a very simplified way) is as follows:
	- **Create a render pipeline state** listing graphics shader functions you are intended to use.
	- **Create command buffer** that will eventually dispatch your commands for execution by the GPU.
	- **Create command encoder** that will let you specify shader functions context, like provide input and output for each.
	- **Commit the commands** to schedule the buffer for execution.

## Give me the code!

Ok, let's start with configuring our view controller.

```swift

public class MTKViewController: UIViewController {

    // MARK: - Public interface
    
    /// Metal texture to be drawn whenever the view controller is asked to render its view. 
    public var texture: MTLTexture?

    // MARK: - Public overrides
    
    override public func loadView() {
        super.loadView()
        assert(device != nil, "Failed creating a default system Metal device. Please, make sure Metal is available on your hardware.")
    }
    
    // MARK: - Private Metal-related properties and methods

    /// `UIViewController`'s view
    private var metalView: MTKView!

    /// Metal device
    private var device = MTLCreateSystemDefaultDevice()

}

```

We've only defined a couple of variables for now: `texture` which will hold a `MTLTexture` we are going to render, `metalView` which is a special kind of view from `MetalKit` (another Apple `Metal`-based framework), that should make rendering of `Metal` textures easier, and a `MTLDevice` that we have already mentioned in previous posts — it's a `Metal` object representing your GPU hardware.

### Initialise `MTKView`

Now let's add a method to configure the `MTKView`.

```swift
/**
 initializes and configures the `MTKView` we use as `UIViewController`'s view.
 
 */
private func initializeMetalView() {
    metalView = MTKView(frame: view.bounds, device: device)
    metalView.delegate = self
    metalView.framebufferOnly = true
    metalView.colorPixelFormat = .BGRA8Unorm
    metalView.contentScaleFactor = UIScreen.mainScreen().scale
    metalView.autoresizingMask = [.FlexibleWidth, .FlexibleHeight]
    view.insertSubview(metalView, atIndex: 0)
}

```

Now add a `initializeMetalView()` call at the end of `MTKViewController`'s `loadView()`.

### Implement shaders

Ok, let's add the actual shaders in Metal shading language! First add a new shaders file to your project by going to *File -> New -> Metal File*. Now, there are three types of shaders: kernel, vertex and fragment functions. We are only going to use per-vertex and per-fragment function here, leaving kernel ones aside.

> Although kernel shaders are arguably the most interesting ones, as kernels is a type of shader one would use for parallel computations. For instance, multiplying large vectors and matrices, which is the heart and soul of neural networks and other machine learning algorithms. 

The shaders are then compiled into a `MTLLibrary` object, which is an interface for referencing all of your shaders. The default library simply compiles `.metal` files you have in your main bundle (so beware of possible issues in case you decide to put shaders into a framework, you may need to use a customised library intead). 

First, let's get imports and namespaces out of the way. Make sure you have the following lines in your `.metal` file:

```c
#include <metal_stdlib>
using namespace metal;
```

We will start with defining a `struct` mapping a texture coordinate to a rendered coordinate:

```c
typedef struct {
    float4 renderedCoordinate [[position]];
    float2 textureCoordinate;
} TextureMappingVertex;
```

We've defined a new `struct` here named `TextureMappingVertex` with two fields: `renderedCoordinate` and `textureCoordinate`. Please, mind the `[[position]]` part: this is a so called <a target="_blank" href="https://developer.apple.com/library/ios/documentation/Metal/Reference/MetalShadingLanguageGuide/func-var-qual/func-var-qual.html#//apple_ref/doc/uid/TP40014364-CH4-DontLinkElementID_6">Attribute Qualifier</a>, which is labeling a struct field with one of predefined attributes. We use `position` here because we are going to use this `TextureMappingVertex` as an output of a vertex shader. Since vertex shader should output a vertex position, and we are going to use some custom `struct` as a return value, you need to explicitly mark one of the `struct` fields with a `position` attribute, in order for the command encoder to know which part of this `struct` is going to define the actual vertext position.

Now let's define a vertex `mapTexture()` and a fragment `displayTexture()` functions. Vertex function will map the texture vertices to rendering coordinates, and fragment will simply return the color of each pixel.

```c
vertex TextureMappingVertex mapTexture(unsigned int vertex_id [[ vertex_id ]]) {
    float4x4 renderedCoordinates = float4x4(float4( -1.0, -1.0, 0.0, 1.0 ),	  /// (x, y, depth, W)
                                            float4(  1.0, -1.0, 0.0, 1.0 ),
                                            float4( -1.0,  1.0, 0.0, 1.0 ),
                                            float4(  1.0,  1.0, 0.0, 1.0 ));

    float4x2 textureCoordinates = float4x2(float2( 0.0, 1.0 ), /// (x, y)
                                           float2( 1.0, 1.0 ),
                                           float2( 0.0, 0.0 ),
                                           float2( 1.0, 0.0 ));
    TextureMappingVertex outVertex;
    outVertex.renderedCoordinate = renderedCoordinates[vertex_id];
    outVertex.textureCoordinate = textureCoordinates[vertex_id];
    
    return outVertex;
}
```

This vertex shader maps the texture vertices to that of the view it's rendering texture on. The mapping is specified by `renderedCoordinates` (position in the coordinate space of the view) and `textureCoordinates` (position in the coordinate space of the texture). Each of the `renderedCoordinates` is a vertex defined by four values: x, y, depth and W coordinate. Don't worry about the latter two for now (they all have the same values as you see), and for x and y to make sense you will have to imagine the axes running through the exact center of the screen, which would have the coordinates (0, 0). So `renderedCoordinates` has coordinates of the screen edges in the following order: 

*left-bottom* (-1.0, -1.0), *right-bottom* (1.0, -1.0), *left-top* (-1.0,  1.0), *right-top* (1.0,  1.0).

Now, `textureCoordinates` contains exactly same points, but in the coordinate space of the texture, which is different from that of the view. The origin of the pixel coordinate system of a texture starts with its top left corner, which would have the coordinates (0, 0). Basically it's same as the coordinate system in UIKit, with x axis going left to right and y going top to bottom. So `textureCoordinates` would list the exactly same edges of the texture in the same order as `renderedCoordinates`: 

*left-bottom* (0.0, 1.0), *right-bottom* (1.0, 1.0), *left-top* (0.0, 0.0), *right-top* (1.0, 0.0).

```c
fragment half4 displayTexture(TextureMappingVertex mappingVertex [[ stage_in ]],
                              texture2d<float, access::sample> texture [[ texture(0) ]]) {
    constexpr sampler s(address::clamp_to_edge, filter::linear);

    return half4(texture.sample(s, mappingVertex.textureCoordinate));
}
```

This fragment function simply returns color value for each pixel in the texture.

### Do the drawing

In order to do the drawing we will override `MTKView`'s drawing methods. You may have noticed that we have set `self` as a delegate for the `MTKView` in `initializeMetalView()` function. Now we will implement the drawing delegate method:

```swift
extension MTKViewController: MTKViewDelegate {
    public func drawInMTKView(view: MTKView) {
	    guard
	        var texture = texture,
	        let device = device
	    else { return }
    
	    /// The rendering goes here.
    }
}
```

Now, let's get to the rendering code!

#### Create a render pipeline state

Now that we have defined the shaders, we can initialise render pipeline state. 

```swift
/**
 initializes render pipeline state with a default vertex function mapping texture to the view's frame and a simple fragment function returning texture pixel's value.
 */
private func initializeRenderPipelineState() {
    guard let
        device = device,
        library = device.newDefaultLibrary()
    else { return }
    
    let pipelineDescriptor = MTLRenderPipelineDescriptor()
    pipelineDescriptor.sampleCount = 1
    pipelineDescriptor.colorAttachments[0].pixelFormat = .BGRA8Unorm
    pipelineDescriptor.depthAttachmentPixelFormat = .Invalid
    
    /**
     *  Vertex function to map the texture to the view controller's view
     */
    pipelineDescriptor.vertexFunction = library.newFunctionWithName("mapTexture")
    /**
     *  Fragment function to display texture's pixels in the area bounded by vertices of `mapTexture` shader
     */
    pipelineDescriptor.fragmentFunction = library.newFunctionWithName("displayTexture")
    
    do {
        try renderPipelineState = device.newRenderPipelineStateWithDescriptor(pipelineDescriptor)
    }
    catch {
        assertionFailure("Failed creating a render state pipeline. Can't render the texture without one.")
        return
    }
}
```

We have defined the vertex function and the fragment function, so the render pipeline state now has enough information to render the texture on screen: vertex shader specifying the position, and fragment shader specifying which color to draw each pixel in. Now, add a call to `initializeRenderPipelineState()` at the end of your `loadView()` method.

#### Create command buffer

Next step would be to initialise a command buffer. We are going to do it in the `MTKView` drawing callback, so add this line at the end of the `drawInMTKView(_: MTKView)` function:

```swift
let commandBuffer = device.newCommandQueue().commandBuffer()
```

#### Create command encoder

`MTKView` is supposed to be a convinience wrapper for a bunch of `Metal` drawing routines, so will use this fact to get a couple other `Metal` specific objects required to create a command encoder: render pass descriptor and current drawable object. Add this at the end of your `drawInMTKView(_: MTKView)` callback:

```swift
guard let
    currentRenderPassDescriptor = metalView.currentRenderPassDescriptor,
    currentDrawable = metalView.currentDrawable,
    renderPipelineState = renderPipelineState
else { return }
```

Ok, we now can create the command encoder and specify the commands. We are going to wrap them into a debug group named "RenderFrame" (since that's exactly what we're doing here):

```swift
let encoder = commandBuffer.renderCommandEncoderWithDescriptor(currentRenderPassDescriptor)
encoder.pushDebugGroup("RenderFrame")
encoder.setRenderPipelineState(renderPipelineState)
encoder.setFragmentTexture(texture, atIndex: 0)
encoder.drawPrimitives(.TriangleStrip, vertexStart: 0, vertexCount: 4, instanceCount: 1)
encoder.popDebugGroup()
encoder.endEncoding()
```

Here we have specified the input texture and commands to draw a primitive with 4 vertices (since we are drawing a rectangle texture on a rectangle screen).

#### Commit the commands

Ok, we are almost there. Finish your `drawInMTKView(_: MTKView)` implementation by presenting current drawable and committing the commands for execution.

```swift
commandBuffer.presentDrawable(currentDrawable)
commandBuffer.commit()
```

## That's it? What do I do with this view controller?

Yes, that's it! You can now use `MTKViewController` in your iOS app, for example, wiring it up to use raw camera data from previous posts. You can check out an example project on GitHub that implements exactly this behaviour: it converts streaming camera frames to `Metal` textures and renders them on screen with the `Metal` shaders used in this post.

## Where do I go from here?

This was the final part of **Metal Camera Tutorial** series, where we explore ways of achieving lowest-overhead access to hardware to grab camera frames, convert them to textures and render on screen in real time:

* <a target="_blank" href="/metal-camera-part-1-camera-session">**Part 1: Getting raw camera data**</a>
* <a target="_blank" href="/metal-camera-part-2-metal-texture">**Part 2: Converting sample buffer to a Metal texture**</a>
* **Part 3: Rendering a Metal texture**
* <a target="_blank" href="/metal-camera-bonus-running-simulator">**Bonus: Running Metal project in iOS Simulator**</a>

You can check out the <a target="_blank" href="https://github.com/alexstaravoitau/MetalRenderCamera">final project</a> from this **Metal Camera Tutorial** on GitHub.

<!-- Place this tag where you want the button to render. -->
<a class="github-button" href="https://github.com/alexstaravoitau" data-style="mega" data-count-href="/navoshta/followers" data-count-api="/users/navoshta#followers" data-count-aria-label="# followers on GitHub" aria-label="Follow @navoshta on GitHub">Follow @navoshta</a>
<!-- Place this tag where you want the button to render. -->
<a class="github-button" href="https://github.com/alexstaravoitau/MetalRenderCamera" data-icon="octicon-star" data-style="mega" data-count-href="/navoshta/MetalRenderCamera/stargazers" data-count-api="/repos/navoshta/MetalRenderCamera#stargazers_count" data-count-aria-label="# stargazers on GitHub" aria-label="Star navoshta/MetalRenderCamera on GitHub">Star</a>
<!-- Place this tag where you want the button to render. -->
<a class="github-button" href="https://github.com/alexstaravoitau/MetalRenderCamera/fork" data-icon="octicon-repo-forked" data-style="mega" data-count-href="/navoshta/MetalRenderCamera/network" data-count-api="/repos/navoshta/MetalRenderCamera#forks_count" data-count-aria-label="# forks on GitHub" aria-label="Fork navoshta/MetalRenderCamera on GitHub">Fork</a>
<!-- Place this tag where you want the button to render. -->
<a class="github-button" href="https://github.com/alexstaravoitau/MetalRenderCamera/archive/master.zip" data-icon="octicon-cloud-download" data-style="mega" aria-label="Download navoshta/MetalRenderCamera on GitHub">Download</a>

<!-- Place this tag in your head or just before your close body tag. -->
<script async defer src="https://buttons.github.io/buttons.js"></script>
