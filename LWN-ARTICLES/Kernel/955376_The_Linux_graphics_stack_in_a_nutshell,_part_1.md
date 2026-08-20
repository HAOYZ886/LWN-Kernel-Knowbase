---
title: "The Linux graphics stack in a nutshell, part 1"
url: https://lwn.net/Articles/955376/
date: "December 19, 2023"
category: "Device drivers-Graphics"
author: "December 19, 2023 This article was contributed by Thomas Zimmermann"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

December 19, 2023

This article was contributed by Thomas Zimmermann

Linux graphics developers often speak of _modern_ Linux graphics when they refer to a number of individual software components and how they interact with each other. Among other things, it's a mix of kernel-managed display resources, Wayland for compositing, accelerated 3D rendering, and decidedly not X11. In a two-part series, we will take a fast-paced journey through the graphics code to see how it converts application data to pixel data and displays it on the screen. In this installment, we look at application rendering, Mesa internals, and the necessary kernel features. 

####  Application rendering 

Graphics output starts in the application, which processes and stores formatted data that is to be visualized. The common data structure for visualization is the [scene graph](<https://en.wikipedia.org/wiki/Scene_graph>), which is a tree where each node stores either a model in 3D space or its attributes. Model nodes contain the data to be visualized, such as a game's scenery or elements of a scientific simulation. Attribute nodes set the orientation or location of the models. Each attribute node applies to the nodes below. To render its scene graph into an on-screen image, an application walks the tree from top to bottom and left to right, sets or clears the attributes and renders the 3D models accordingly. 

In the example scene graph shown below, rendering starts at the root node, which prepares the renderer and sets the output location. The application first takes the branch to the left and renders "Rectangle 1" at position (0, 0) with the surface pattern stored in texture 1\. The application then moves back to the root node and takes the branch to the right where it enters the attribute node named "Transform". Applications describe transformations, such as positioning or scaling, in 4x4 matrices that the algorithm applies during the rendering process. In the example, the transform node scales all of its child nodes by a factor of 0.5. So rendering "Rectangle 2" and "Rectangle 3" displays them at half their original sizes, with their positions adjusted to (10, 10) and (15, 15), respectively. Both rectangles use different textures: 2 and 3, respectively. 

> ![\[Scene graph\]](https://static.lwn.net/images/2023/scenegraph.png)

To simplify rendering and make use of hardware acceleration, most applications utilize one of the standard APIs, such as [OpenGL](<https://opengl.org/>) or [Vulkan](<https://vulkan.org/>). Details vary among the individual APIs, but each provides interfaces to manage graphics memory, fill it with data, and render the stored information. The result is an image that the application can either display as-is or use as input data to further processing. 

All graphics data is held in buffer objects, each of which is a range of graphics memory with a handle or ID attached. For example, each 3D model is stored in a [vertex-buffer object](<https://en.wikipedia.org/wiki/Vertex_buffer_object>), each texture is stored in a [texture-buffer object](<https://www.khronos.org/opengl/wiki/Texture>), each object's [surface normals](<https://en.wikipedia.org/wiki/Normal_\(geometry\)>) are stored in a buffer object, and so on. The output image is itself [stored in a buffer object](<https://www.khronos.org/opengl/wiki/Framebuffer_Object>). So graphics rendering is, in large part, an exercise in memory management. 

The application can provide input data in any format, as long as the graphics shader can process it. A [shader](<https://en.wikipedia.org/wiki/Shader>) is a program that contains the instructions to transform the input data into an output image. It is provided by the application and executed by the graphics card. 

Real-world shader programs can implement complex, multi-pass algorithms, but for this example we break it down to the minimum required. Probably the two most common operations in shaders are vertex transformations and texture lookups. We can think of a vertex as a corner of a polygon. Written in [OpenGL Shading Language](<https://www.khronos.org/opengl/wiki/OpenGL_Shading_Language>) (GLSL), transforming a vertex looks like this: 

```
uniform mat4 Matrix; // same for all of a rectangles's vertices
        in vec4 inVertexCoord; // contains a different vertex coordinate on each invocation
    
        gl_Position = Matrix * inVertexCoord;
```

The variable `inVertexCoord` is an input coordinate coming from the application's scene graph. The variable `gl_Position` is the coordinate within the application's output buffer. In broad terms, the former coordinate is within the displayed scenery, while the latter is within the application window. `Matrix` is the 4x4 matrix that describes the transformation between the two coordinate systems. This shader operation runs for each vertex in the scene graph. In the example of the application's scene graph of rectangles above, `inVertexCoord` contains each vertex of each rectangle at least once. The matrix `Matrix` then contains that vertex's transformation, such as moving it to the correct position or scaling it by the factor of 0.5 as specified in the transform node. 

Once the vertices are transformed to the output coordinate system, the shader program computes the values of the covered "fragments", which is graphics jargon for an output pixel with a depth value along the Z axis and maybe other information. Each fragment requires a color. In GLSL, the shader's `texture()` function retrieves the color from a texture like this: 

```
uniform sampler2D Tex; // the texture object of the current rectangle
        in vec2 vsTexCoord; // interpolated texture coordinate for the fragment
    
        Color = texture(Tex, vsTexCoord);
```

Here, `Tex` represents a texture buffer. The value `vsTexCoord` is the texture coordinate; the position where to read within the texture. Using both values, `texture()` returns a color value. Assigning it to `Color` writes a colored pixel to the output buffer. To fill the output buffer with pixel data, this shader code runs for each individual fragment. The texture buffer is designated by the model that is being drawn, the texture coordinate is provided by OpenGL's internal computation. For the example scene graph, the application invokes this code for each of the rectangles using that rectangle's texture buffer. 

Running these shader instructions on the whole scene graph generates the complete output image for the application. 

####  Mesa 

Nothing we have discussed so far is specific to Linux, but it gives us the framework to look at how it's all implemented. On Linux, the [Mesa 3D](<https://mesa3d.org/>) library, Mesa for short, implements 3D rendering interfaces and support for various graphics hardware. To applications, it provides OpenGL or Vulkan for desktop graphics, [OpenGL ES](<https://www.khronos.org/opengles/>) for mobile systems, and [OpenCL](<https://www.khronos.org/opencl/>) for computation. On the hardware side, Mesa implements drivers for most of today's graphics hardware. 

Mesa drivers generally do not implement these application interfaces by themselves as Mesa contains plenty of helpers and abstractions. For stateful interfaces, such as OpenGL, Mesa's [Gallium3D](<https://www.freedesktop.org/wiki/Software/gallium/>) framework connects interfaces and drivers with each other. This is called a state tracker. Mesa contains state trackers for various versions of OpenGL, OpenGL ES, and OpenCL. When the application uses the API, it modifies the state tracker for the given interface. 

A hardware driver within Mesa further converts the state-tracker information to hardware state and rendering instructions. For example, calling OpenGL's [`glBindTexture()`](<https://registry.khronos.org/OpenGL-Refpages/gl4/html/glBindTexture.xhtml>) selects the current texture buffer within the OpenGL state tracker. The hardware driver then installs the texture-buffer object in graphics memory and links the active shader program to refer to the buffer object as its texture. In our example above, the texture becomes available as `Tex` in the shader program. 

Vulkan is a stateless interface, so Gallium3D is not useful for those drivers; Mesa instead offers the [Vulkan runtime](<https://docs.mesa3d.org/vulkan/index.html>) to help with their implementation. If there is a Vulkan driver available, though, there might not be a need for Gallium3D-based OpenGL support at all for that hardware. [Zink](<https://docs.mesa3d.org/drivers/zink.html>) is a Mesa driver that maps Gallium3D to Vulkan. With Zink, OpenGL state turns into Gallium3D state, which is then forwarded to hardware via standard Vulkan interfaces. In principle, this works with any hardware's Vulkan driver. One can imagine that future drivers within Mesa only implement Vulkan and rely on Zink for OpenGL compatibility. 

Besides Gallium3D, Mesa provides more helpers, such as winsys or GBM, to its hardware drivers. Winsys wraps the details of the window system. GBM, the Generic Buffer Manager, simplifies buffer-object allocation. There are also a number of shader languages, such as GLSL or [SPIR-V](<https://www.khronos.org/spir/>), available to the application. Mesa compiles the application-provided shader code to the "New Interface Representation" or NIR, which Mesa drivers turn into hardware instructions. To get the shader instructions and the associated data processed by Mesa's hardware acceleration, their buffer objects have to be stored in memory locations accessible to the graphics card. 

####  Kernel memory management 

Any memory accessible to the graphics hardware is commonly subsumed under the umbrella term of graphics memory; it is the graphics stack's central resource, as all of the stack's components interact with it. On the hardware side, graphics memory comes in a variety of configurations that range from dedicated memory on discrete graphics adapters to regular system memory of system-on-chip (SoC) boards. In between are graphics chips with DMA-able or [shared graphics memory](<https://en.wikipedia.org/wiki/Shared_graphics_memory>), [graphics address remapping table](<https://en.wikipedia.org/wiki/Graphics_address_remapping_table>) (GART) memory of discrete devices, or the so-called stolen graphics memory of on-board graphics. 

Being a system-wide resource, graphics memory is maintained by the kernel's [Direct Rendering Manager](<https://en.wikipedia.org/wiki/Direct_Rendering_Manager>) (DRM) subsystem. To access DRM functionality, Mesa opens the graphics card's device file under `/dev/dri`, such as `/dev/dri/renderD128`. As required by its user-space counterparts, DRM exposes graphics memory in the form of buffer objects, where each buffer object represents a slice of the available memory. 

The DRM framework provides a number of memory managers for the most common cases. The DRM drivers for the discrete graphics cards from AMD, NVIDIA, and (soon) Intel use the [Translation Table Manager](<https://docs.kernel.org/gpu/drm-mm.html>) (TTM). It supports discrete graphics memory, GART memory, and system memory. TTM can move buffer objects between these areas, so if the device's discrete memory fills up, unused buffer objects can be swapped out to system memory. 

Drivers for simple framebuffer devices often use the SHMEM helpers, which allocate buffer objects in shared memory. Here, regular system memory acts as a shadow buffer for the device's limited resources. The graphics driver maintains the device's graphics memory internally, but exposes buffer objects in system memory to the outside. This also makes it possible to memory-map buffer objects of devices on the USB or I2C bus, even though these buses do not support page mappings of device memory; the shadow buffer can be mapped instead. 

The other common allocator, the DMA helper, manages buffer objects in DMA-able areas of the physical memory. This design is often used in SoC boards, where graphics chips fetch and store data via DMA operations. Of course, DRM drivers with additional requirements have the option of extending one of the existing memory managers or implementing their own. 

The `ioctl()` interface for managing buffer objects is called the [Graphics Execution Manager](<https://docs.kernel.org/gpu/drm-mm.html#the-graphics-execution-manager-gem>) (GEM). Each DRM driver implements GEM according to its hardware's features and requirements. The GEM interface allows mapping a buffer object's memory pages to user-space or kernel address space, allows pinning the pages at a certain location, or exporting them to other drivers. For example, an application in user space can get access to a buffer object's memory pages by invoking `mmap()` with the correct offset on the DRM device file's file descriptor. The call will eventually end up in the DRM driver's GEM code, which sets up the mapping. As we will see below, it's a useful feature for software rendering. 

The one common operation that GEM does not provide is buffer allocation. Each buffer object has a specific use case, which affects and is affected by the object's allocation parameters, memory location, or hardware constraints. Hence, each DRM driver offers a dedicated `ioctl()` operation for buffer-object allocation that captures these hardware-specific settings. The DRM driver's counterpart in Mesa invokes said `ioctl()` operation accordingly. 

####  Rendering operations 

Just having buffer objects for storing the output image, the input data, and the shader programs is not enough. To start rendering, Mesa instructs DRM to put all necessary buffer objects in graphics memory and invokes the active shader program. It's again all specific to the hardware and provided as `ioctl()` operations by each DRM driver individually. As with buffer allocation, the hardware driver within Mesa invokes the DRM driver's `ioctl()` operations. For Mesa drivers based on Gallium3D, this happens when the driver converts the state tracker information into hardware state. 

The graphics driver ideally acts only as a proxy between the application in user space and the hardware. The hardware renderer runs asynchronously to the rest of the graphics stack and reports back to the driver only in case of an error or on successful completion; much like the system CPU informs the operating system on page faults or illegal instructions. As long as there's nothing to report, driver overhead is minimal. There are exceptions; for example, older models of Intel graphics chips do not support vertex transformations, so the driver within Mesa has to implement them in software. On the Raspberry Pi, the kernel's DRM driver has to validate each shader's memory access, as the VideoCore 4 chip does not contain an I/O MMU to isolate the shader from the system. 

####  Software rendering 

So far, we have assumed hardware support for graphics rendering. What if there's no such support or the user-space application cannot use it? For example, a user-space GUI toolkit might prefer rendering in software because hardware-centric interfaces like OpenGL do not fit its needs. And there's Plymouth, the program that displays the boot-up logo and prompts for disk-encryption passwords during boot, which usually does not have a full graphics stack available. For these scenarios, DRM offers the dumb-buffer `ioctl()` interface. 

By utilizing dumb buffers, an application allocates buffer objects in graphics memory, but without support for hardware acceleration. So any returned buffer object is only usable with software rendering. The application in user space, such as a GUI toolkit or Plymouth, maps the buffer object's pages into its address space and copies over the output image. Mesa's software renderer works similarly: the input buffer objects all live in system memory and the system CPU processes the shader instructions. The output buffer is a dumb-buffer object that stores the rendered image. While that's neither fast nor fancy, it's good enough to run a modern desktop environment on simple hardware that does not support accelerated rendering. 

We have now gone through the application's graphics stack for rendering. After having completed the traversal of the scene graph, the application's output buffer object contains the visualized scenery or data that it wants to display. But the buffer is not yet on the screen. Whether accelerated or dumb, putting the buffer on the screen requires compositing and mode setting, which form the other half of the graphics stack. In [part 2](<https://lwn.net/Articles/955708/>), we will look at Wayland compositing, setting display modes with DRM, and a few other features of the graphics stack.
