---
title: An update on the freedreno graphics driver
url: https://lwn.net/Articles/638908/
date: "April 8, 2015"
category: "Device drivers-Graphics"
author: "By Jake Edge April 8, 2015 ELC 2015"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jake Edge**  
April 8, 2015

* * *

[ELC 2015](<https://lwn.net/Archives/ConferenceByYear/#2015-Embedded_Linux_Conference>)

The [freedreno project](<https://freedreno.github.io/>) was started by Rob Clark to create a free-software driver for the Adreno family of GPUs, which are used by the Qualcomm Snapdragon system-on-chip (SoC) family. He presented a status report on the project, along with some history and future plans, at the [Embedded Linux Conference](<http://events.linuxfoundation.org/events/embedded-linux-conference>), which was held in San Jose, CA, March 23-25. 

The Adreno 2xx, 3xx, and 4xx are all supported by freedreno. The 2xx GPUs support [OpenGL ES](<https://www.khronos.org/opengles/>) 2.0, the 3xx devices support OpenGL ES 3.0 and the embedded profile of [OpenCL](<https://www.khronos.org/opencl/>) 1.1, while the Adreno 4xx GPUs support OpenGL ES 3.1 (with the [Android Extension Pack](<https://www.khronos.org/registry/gles/extensions/ANDROID/ANDROID_extension_pack_es31a.txt>)) and the full profile of OpenCL 1.2. The 3xx is the first of the modern Adrenos, Clark said, and the 4xx was announced with the Snapdragon 805 SoC, which is the first Adreno to support [DirectX](<http://en.wikipedia.org/wiki/DirectX>) 11\. 

Adrenos have a tile-based renderer, though it is implemented differently than other tile-based devices. Adrenos have a relatively large internal memory for the tile buffer, either in the GPU core itself (GMEM) or elsewhere on the SoC (OCMEM). The driver manages the tile buffers, including handling restore and resolve operations (moving tile data between the CPU and GPU memory). It also handles partitioning the rendering target into tiles. On Adreno 3xx and later GPUs, the driver can decide to bypass the tile buffer to do immediate rendering in certain scenarios. 

#### Motivation and history

Clark got involved because of the lack of free drivers for various Snapdragon-based boards that were becoming available. Graphics progress was being held back because the GPUs were all locked down. These days, developers expect to have GPU acceleration available for user interfaces and other purposes, but for these ARM boards, you were "left with Android or Android" for driver choices. Those drivers didn't come with source, so you couldn't even recompile them. There were some "clever hacks" like [libhybris](<http://en.wikipedia.org/wiki/Hybris_%28software%29>), but "piling on more duct tape doesn't solve the issue". 

[ ![\[Rob Clark\]](https://static.lwn.net/images/2015/elc-clark-sm.jpg) ](<https://lwn.net/Articles/639250/>)

So in mid-2012 he decided to do something about it. He was working for TI at the time, so PowerVR-based GPUs (as used by TI) were off-limits. He found some hardware that had an Adreno 220 and started to reverse engineer it. He began work on a [Gallium](<http://www.freedesktop.org/wiki/Software/gallium/>) driver in November 2012\. By early 2013, he had most of the "normal stuff" working. He could run GNOME Shell and some games on the hardware. 

One of the nice things about a Gallium driver, Clark said, is that it supports both desktop graphics and GL ES. Lots of games are not ported to GL ES but can still be played using the Gallium driver. 

That early work was done mostly in the evenings and on weekends, but that changed somewhat when he joined the Red Hat graphics team in February 2013\. While his freedreno work is not his full-time job, he does get some work time to do freedreno development. 

In March 2013, he ordered a Nexus 4, which has an Adreno 320 GPU, and started looking at that. Everything had changed between the two Adreno revisions, including the shader instruction set and the registers. Adreno 2xx support was working well at that point, so it has pretty much been left behind. By mid-2013, he had Adreno 3xx support working at a basic level. 

The Direct Rendering Manager (DRM) driver for the MSM (Mobile Station Modem, a Qualcomm hardware designation that is used as the Adreno driver name) devices was merged into the mainline in August 2013 for the 3.12 kernel, which provided a "nice platform" for further development work. In January 2014, he added hardware binning support, which is a pre-pass made on the triangles to be rendered to see which fit in each tile. That sped up rendering so that he was able to get a "fairly playable" 30 frames per second (fps) in [Xonotic](<http://www.xonotic.org/>). 

He also started work on a new shader compiler in early 2014. The earlier compiler had just translated [TGSI](<http://gallium.readthedocs.org/en/latest/tgsi.html>) directly to native instructions, but didn't do proper instruction scheduling. Adding that scheduling fixed a number of problems while also giving a big performance boost, he said. 

He started work on OpenGL 2.0 and 2.1 support in May 2014\. Xonotic supports both OpenGL 1.0 and 2.0, so it can easily be used to test the changes made for 2.0 support. 

In perhaps something of a surprise move, Qualcomm posted the first patches for the hardware in June 2014\. The initial patches were for display support, but subsequent patches were for the DRM/MSM driver itself. Clark strongly commended Qualcomm for doing that work upstream. 

In mid-2014, he got his hands on his first Adreno 4xx device (an [Inforce6540](<http://mydragonboard.org/ifc6540/>) with an Adreno 420). He found that nearly all of the registers had all changed again, which required another round of reverse engineering to figure them out. The shader instruction set was quite similar to that of the 3xx, so he was able to share that code between the two GPU versions. 

By October 2014, 90% of the [Piglit](<http://people.freedesktop.org/~nh/piglit/>) OpenGL tests were passing for the Adreno 3xx. Piglit is useful to test "weird corner cases", he said. It is often the case that games will work fine, but lots of Piglit tests still fail. 

Kernel driver patches to enable the 4xx were submitted by Qualcomm in November 2014 for the 3.19 kernel, which was another big step for the company. The initial Gallium support for the 4xx was also released around the same time. As of February 2015, the 4xx support is behind where the 3xx is, but most games and other programs are largely working at this point. By March 2015, the new shader compiler was able to handle everything that the old compiler could, so the old one has been retired. 

The architecture for freedreno is much the same as for the Intel, Radeon, or Nouveau drivers. There are multiple user-space pieces (Gallium driver, X video driver, and libdrm_freedreno) that talk to the kernel driver. By implementing it that way, they got Wayland support for free. GNOME Shell works just fine on Adreno hardware using the Wayland compositor. 

Clark said that he doesn't know much about the Android graphics stack. He would like to see Android support free graphics for Adreno devices, but that requires further investigation. 

#### User space

The biggest single component of an open-source graphics driver is the Gallium driver, he said. The freedreno Gallium driver has a common core that handles tasks like dirty state and buffer tracking, tile-buffer management, and fences (i.e. synchronization operations). There are also separate components (fd2, fd3, and fd4) to handle each generation of Adreno devices, as well as a shader compiler (ir3) that is shared between 3xx and 4xx devices. 

The user-space piece of freedreno builds up a command stream that has lots of register references. That gets sent to the kernel driver, which is a "glorified register writer", Clark said. The shader compiler is the single largest piece of the user-space driver. He also went into some details of the [tiling implementation](<https://github.com/freedreno/freedreno/wiki/Adreno-tiling>) and how [queries](<https://github.com/freedreno/freedreno/wiki/Queries>) (e.g. occlusion queries) are handled by the driver and the hardware. 

Turning to debugging, Clark said that the `FD_MESA_DEBUG` environment variable is the most useful technique for debugging. This command: 

```
$ FD_MESA_DEBUG=help glxinfo
```

will produce a help message that lists other values for the environment variable. Another useful tool is [apitrace](<https://github.com/apitrace/apitrace>), which will save all of the GL commands into a trace file. That file can be used to reproduce some problem entirely separate from the program that produced the output. Command-stream traces can also be grabbed from the kernel driver (using the `/sys/kernel/debug/dri/0/rd` file). 

The envytools from the Nouveau project were used to generate header files that describe the Adreno registers for both the Gallium and DRM/MSM kernel driver. As he figures out the registers of the GPUs, Clark enters them into an XML file that is used by other tools, both to create the headers and to decode various kinds of trace files. For example, cffdump can decode the command-stream trace files from both the free and binary blob drivers to compare the output of each. Information about the tools used to reverse engineer the GPUs is [available](<https://github.com/freedreno/freedreno/wiki/Reverse-engineering-tools>) on the freedreno wiki. 

#### GL/GL ES 3.0

There is a lot of work going on "behind the scenes" to enable GL/GL ES 3.0, Clark said. A lot of the work to do so has been done by Ilia Mirkin. Supporting 3.0 will enable more games to work and will provide more advanced rendering features. Clark has also done work on the shader compiler to support the new version of the [GLSL](<https://www.opengl.org/documentation/glsl/>) shader language. 

For GL ES 3.0, the "big ticket item" that is left to do is to implement [transform feedback](<https://www.opengl.org/wiki/Transform_Feedback>) and [uniform buffer objects](<https://www.opengl.org/wiki/Uniform_Buffer_Object>) (UBOs). For those, the reverse engineering has been done, all that's left is to "write code". The code for [multiple render targets](<http://en.wikipedia.org/wiki/Multiple_Render_Targets>) (MRTs) exists on a branch, but has not yet been merged. There is also some shader compiler work to support advanced flow control, but that is not used in games much. 

Support for the [NV_conditional_render](<https://www.opengl.org/registry/specs/NV/conditional_render.txt>) extension is the biggest piece left for GL 3.0 support. Clark said that the extension could be called the "we hate tilers" extension, since handling it will perform badly on a tile-based GPU. 

The shader compiler is already the biggest piece of the user-space driver, but it will be getting somewhat bigger to handle some of these new features. Clark has started documenting the [compiler design](<http://gallium.readthedocs.org/en/latest/drivers/freedreno/ir3-notes.html>) and the [Adreno instruction set architecture](<https://github.com/freedreno/freedreno/wiki/A3xx-shader-instruction-set-architecture>). He has also has some preliminary work toward supporting the [new internal representation](<http://lists.freedesktop.org/archives/mesa-dev/2014-December/072761.html>) (NIR), which is an Intel effort aimed at a new shader compiler IR for better optimization. He has a TGSI to NIR translation patch, but it is not ready to land yet. 

Getting the freedreno driver should be easy, since all of the pieces are already upstream. Distributions generally already have it enabled. He recommends Mesa 10.4.x for Adreno 3xx devices and 10.5.1 (or higher) for 4xx GPUs. 

As with most free-software projects, freedreno could use some help. To start with, if a distribution doesn't have the driver enabled, help make that happen, he suggested. Bug reports are also welcome. For those interested in actually working on the driver, there is help needed in "everything from the kernel to compilers". For those that know GL/GL ES, adding more tests would be helpful as well. 

Clark described the process of reverse engineering a bit in answer to a question from the audience. He will start with a simple GL program that draws a "quad" (a four-sided polygon), then he will change the frame buffer size to see what the blob driver does differently. As he sees different registers and values, he records what he finds in the XML file that is used by cffdump and other tools. He runs a series of tests, just varying one parameter at a time to see what changes and slowly works out all of the registers. For the most part, these tests are rendered off-screen as he doesn't actually care much about what the output looks like. 

[I would like to thank the Linux Foundation for travel support to San Jose for ELC.]
