---
title: Render a simple 3D object - part 2
weight: 7

### FIXED, DO NOT MODIFY
layout: learningpathall
---

## 3D meshes

Once Render Pipeline is created, using WebGPU APIs to create and render a 3D mesh is very similar to other graphics APIs. The flow follows:

* Creating Vertex Buffer(s)
* Creating Index Buffer(s)
* Creating Uniform Buffer(s)
* Creating Depth Buffer (Z-Buffer algorithm)
* Creating Depth Texture and TextureView
* Creating Depth Stencil
* Creating Transformation and Projection matrices

All these steps are common in graphics programming and WebGPU offers capability to perform all the operations. It is recommended to go through individual chapters in [3D rendering](https://eliemichel.github.io/LearnWebGPU/basic-3d-rendering/index.html) section.

### Loading 3D objects

In our project we use OBJ files to define 3D meshes. Instead of manually parsing OBJ files, we use the [TinyOBJLoader](https://github.com/tinyobjloader/tinyobjloader) library. The file format is not that complex, but parsing files is not the main point of this learning path, and this library has been intensively tested and has a very small footprint. You can use open-source softwares like Blender to create your own 3D objects.

{{% notice Note %}}
Exactly one of your source files must define `TINYOBJLOADER_IMPLEMENTATION` before including it:

```C++
#define TINYOBJLOADER_IMPLEMENTATION // add this to exactly 1 of your C++ files
#include "tiny_obj_loader.h"
```

{{% /notice %}}

We have developed a helper function [`loadGeometryFromObj`](https://github.com/varunchariArm/Android_DawnWebGPU/blob/main/app/src/main/cpp/webgpuRenderer.cpp#L475) using the above library.

With all the setup done, we are now ready to render our 3D object

## Rendering using WebGPU

Now that we have done all the setup, we can run a rendering pass and *draw* something onto our *surface*. To encode any commands to be issued to GPU, we need to create a `CommandEncoder`. , Modern APIs record commands into command buffers,rather than issuing commands one by one and submit all of them at once. In WebGPU, this is done through CommandEncoder".

```C++
wgpu::CommandEncoderDescriptor commandEncoderDesc;
commandEncoderDesc.label = "Command Encoder";
wgpu::CommandEncoder encoder = device.createCommandEncoder(commandEncoderDesc);
```

Next step is to create `RenderPassEncoder`. It encodes commands related to controlling the vertex and fragment shader stages, as issued by RenderPipeline. It forms part of the overall encoding activity of a CommandEncoder. A render pipeline renders graphics to Texture attachments, typically intended for displaying on a surface, but it could also render to textures used for other purposes that never appear onscreen. It has two main stages:

* A vertex stage, in which a vertex shader takes positioning data fed into the GPU and uses it to position a series of vertices in 3D space by applying specified effects like rotation, translation, or perspective.
* A fragment stage, in which a fragment shader computes the color for each pixel covered by the primitives produced by the vertex shader

We can create a RenderPassEncoder using `encoder.beginRenderPass()` API:

```C++
wgpu::RenderPassDescriptor renderPassDesc{};

wgpu::RenderPassColorAttachment renderPassColorAttachment{};
renderPassColorAttachment.view = nextTexture;
renderPassColorAttachment.resolveTarget = nullptr;
renderPassColorAttachment.loadOp = wgpu::LoadOp::Clear;
renderPassColorAttachment.storeOp = wgpu::StoreOp::Store;
renderPassColorAttachment.clearValue = Color{ 1, 1, 1, 1.0 };
renderPassColorAttachment.depthSlice = WGPU_DEPTH_SLICE_UNDEFINED;
renderPassDesc.colorAttachmentCount = 1;
renderPassDesc.colorAttachments = &renderPassColorAttachment;
renderPassDesc.timestampWrites = nullptr;
wgpu::RenderPassEncoder renderPass = encoder.beginRenderPass(renderPassDesc);
```

{{% notice Note %}}
`ColorAttachment` is the only mandatory field. Also make sure you have specified `renderPassColorAttachment.depthSlice`. It is recommended to go through the ColorAttachment [members](https://gpuweb.github.io/gpuweb/#color-attachments)
{{% /notice %}}

Now we can invoke the following APIs to draw our 3D object:

* `renderPass.setPipeline(...);`
* `renderPass.setVertexBuffer(...)`
* `renderPass.setBindGroup(...)`
* `renderPass.draw(...)`

To finish encoding the sequence of commands and issue them to the GPU, few more API calls are needed:

* End render pass `renderPass.end()`
* Finish the command
  
  ```C++
  wgpu::CommandBufferDescriptor cmdBufferDescriptor{};
  cmdBufferDescriptor.label = "Command buffer";
  wgpu::CommandBuffer command = encoder.finish(cmdBufferDescriptor);
  encoder.release();
  ```

* Submit the Queue `queue.submit(command)`
* Present the object onto surface `surface_.present()`

{{% notice Tip %}}
Make sure you release the created encoders and buffers by calling the respective `.release()` in order to avoid dangling pointer or other errors.
{{% /notice %}}

{{% notice Note %}}
By default Dawn runs callbacks only when the device “ticks”, so the error callbacks are invoked in a different call stack than where the error occurred, making the breakpoint less informative. To force Dawn to invoke error callbacks as soon as there is an error, you can enable an instance toggle:

```C++
#ifdef WEBGPU_BACKEND_DAWN
// Make sure the uncaptured error callback is called as soon as an error
// occurs rather than at the next call to "wgpuDeviceTick".
WGPUDawnTogglesDescriptor toggles;
toggles.chain.next = nullptr;
toggles.chain.sType = WGPUSType_DawnTogglesDescriptor;
toggles.disabledToggleCount = 0;
toggles.enabledToggleCount = 1;
const char* toggleName = "enable_immediate_error_handling";
toggles.enabledToggles = &toggleName;

desc.nextInChain = &toggles.chain;
#endif // WEBGPU_BACKEND_DAWN
```

Toggles are Dawn’s special way of enabling/disabling features at the scale of the whole WebGPU instance. See the whole list in [Toggle.cpp](https://dawn.googlesource.com/dawn/+/refs/heads/main/src/dawn/native/Toggles.cpp#33).
{{% /notice %}}

## Building and Running the application

Now we can build and run the application, but first copy the file with shader code and 3D object files to connected phone

``` bash
cd app/src/main/cpp
adb shell "mkdir /data/local/tmp/webgpu/"
adb push resources/shader_texture_file.wgsl /data/local/tmp/webgpu/
adb push resources/cone_in_turdis.obj /data/local/tmp/webgpu/
adb push resources/cone_in_turdis.mtl /data/local/tmp/webgpu/
```

Now click the **Run** icon in Android Studio, which builds the application and launches it on the connected device, producing the following output

!["Output"](./images/output.gif, "Output")
