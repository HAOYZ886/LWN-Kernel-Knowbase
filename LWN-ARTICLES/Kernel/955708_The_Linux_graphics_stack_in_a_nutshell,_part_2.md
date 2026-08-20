---
title: "The Linux graphics stack in a nutshell, part 2"
url: https://lwn.net/Articles/955708/
date: "December 28, 2023"
category: "Device drivers-Graphics"
author: "December 28, 2023 This article was contributed by Thomas Zimmermann"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

December 28, 2023

This article was contributed by Thomas Zimmermann

Displaying an application's graphical output onto the screen requires compositing and mode setting that are correctly synchronized among the various pieces, with low overhead. In this second and final article in the series, we will look at those pieces of the Linux graphics stack. In the [first installment](<https://lwn.net/Articles/955376/>), we followed the path of graphics from the application, through Mesa, while using the memory-management features of the kernel's [Direct Rendering Manager](<https://en.wikipedia.org/wiki/Direct_Rendering_Manager>) (DRM) subsystem. We ended up with an application's graphics data stored in an output buffer, so now it's time to display the image to the user. 

####  Compositing 

User-space applications almost never display their output by themselves, but instruct a screen compositor to do so. The compositor is a system service that receives each application's output buffers and draws them to an on-screen image. The layout is up to the to the compositor's implementation, but [stacking](<https://en.wikipedia.org/wiki/Stacking_window_manager>) and [tiling](<https://en.wikipedia.org/wiki/Tiling_window_manager>) are the most common. The compositor is also responsible for gathering user input and forwarding it to the application in focus. 

Compositing, as well as almost everything else in the graphics stack, used to be provided by the [X Window System](<https://en.wikipedia.org/wiki/X_Window_System>), which implements a network protocol for displaying graphics on the screen. Since the everything-else part includes drawing, mode setting, screen sharing, and even [printing](<https://www.x.org/releases/X11R6.8.2/doc/Xprint.7.html>), X suffers from software bloat and is hard to adapt to changes in the graphics hardware and Linux system; a lightweight replacement was needed. Its modern successor is [Wayland](<https://wayland.freedesktop.org/>), which is another client-server design where each application acts as a client to the display service provided by the compositor. Wayland's reference compositor is [Weston](<https://wayland.pages.freedesktop.org/weston/>), but GNOME's [Mutter](<https://gitlab.gnome.org/GNOME/mutter>) or KDE's [KWin](<https://invent.kde.org/plasma/kwin>) are more commonly used. 

There's no drawing or printing in Wayland; the [protocol](<https://wayland-book.com/>) only provides the functionality required for compositing. A Wayland surface represents an application window; it is the application's handle to display its output and to receive input events from the compositor. Attached to the surface is a Wayland buffer that contains the displayable pixel data plus color-format and size information. The pixel data is in the output buffer that the client application has rendered to. Changing a surface's attached buffer object or its content results in a Wayland-protocol surface-damage message from the application to the compositor, which updates the on-screen content; possibly with the contents of a new buffer object. The application's output buffer becomes an input buffer for the Wayland compositor. 

Rendering in the compositor works exactly as described for applications in the first installment. The compositor maintains a list of all of the Wayland surfaces that represent application windows. Those windows and the compositor's interface elements form yet another [scene graph](<https://en.wikipedia.org/wiki/Scene_graph>). The background contains a wallpaper image, background pattern, or color. On top of the background, the compositor draws the application windows. The easiest way to implement this is by drawing a rectangle for each window and using the application-provided buffer object as a texture image. 

On top of the application windows, the compositor draws its own user interface, such as a taskbar where the user can interact with the compositor itself. Finally the topmost layer is an indicator of what the user is currently interacting with; typically a mouse pointer on desktop systems. Like applications, the compositor renders with the regular user-space interfaces, such as Mesa's OpenGL or Vulkan. 

The final building block to make all of this possible is the transfer mechanism for buffer objects. In contrast to X, Wayland applications always run on the same host as their compositor. Implementations are thus free to optimize for this case: there's no network encoding, buffer compression, and so on involved. 

For transferring a buffer object that resides in system memory, the application creates a file descriptor that refers to the buffer's memory, sends it over the connection's stream socket (in a single, low-cost message), and lets the compositor map the file descriptor's memory pages into its address space. Both the application and the compositor have now established a low-overhead channel for exchanging pixel data. The application draws into the shared-memory region and the compositor renders from it. In practice it's also common to use multiple buffer objects for double buffering. Wayland's surface-damage messages serve as a synchronization method with low overhead. 

Transferring data via shared memory is good enough for software rendering but, for high-performance hardware rendering, it is insufficient. The application would have to render on the graphics hardware and read back the result over the slow hardware bus into the region of shared memory. 

To avoid that penalty, the graphics buffer has to remain in graphics memory. Wayland provides a protocol extension to share buffer objects via a Linux [dma-buf](<https://www.kernel.org/doc/html/latest/driver-api/dma-buf.html>), which represents a memory buffer that is shareable among hardware devices, drivers, and user-space programs. An application renders its scene graph via the Mesa interfaces using hardware acceleration as described in part 1, but, instead of transferring a reference to shared memory, the application sends a dma-buf object that references the buffer object while it is still located in graphics memory. The Wayland compositor uses the stored pixel data without ever reading it over the hardware bus. 

Hardware-accelerated rendering is inherently asynchronous and therefore requires synchronization. After the application has sent the current frame's final rendering command to Mesa, it is not guaranteed that the hardware has finished rendering. This is intentional and required for high performance. But having the compositor display the content of a buffer object before the hardware has completed rendering results in distorted output. To avoid this from happening, the hardware signals when it has completed rendering. This is called fencing and the associated data structure is called a fence. The fence is attached to the dma-buf object that the application transfers to the compositor. The compositor waits for the fence to signal completion before it uses the resulting data for generating its own output. 

####  Pixels to the monitor 

After rendering the on-screen image, the compositor has to display it to the user. DRM's mode-setting code controls all aspects of reading pixel data from graphics memory and sending it to an output device. To do so, each driver sets up a pipeline that models the pixel data's flow through the graphics hardware. Each pipeline stage represents a piece of hardware functionality that processes pixel data on its way to the monitor. 

The minimum stages necessary are the framebuffer, plane, CRTC, encoder, and connector, each of which is described below. For a working display output, there has to be at least one active instance of each. But most hardware provides more than the minimum functionality and allows for enabling and disabling pipeline stages at will. The DRM framework comes with software abstractions for each stage upon which drivers can build. 

The pipeline's first stage is the DRM framebuffer. It is the buffer object that stores the compositor's on-screen image, plus information about the image's color format and size. Each DRM driver programs the hardware with this information and points the hardware to the first byte of the buffer object, so that the hardware knows where to find the pixel data. 

Fetching the pixel data is called scanout, and the pixel data's buffer object is called the scanout buffer. The number of scanout buffers per framebuffer depends on the framebuffer's color format. Many formats, such as the common [RGB](<https://en.wikipedia.org/wiki/RGB_color_model>)-based ones, store all pixel data in a single buffer. With other formats, such as [YUV](<https://en.wikipedia.org/wiki/Y%E2%80%B2UV>)-based ones, the pixel data might need to be split up into multiple buffers. 

Depending on the hardware's capabilities, the framebuffer can be larger or smaller than the output's display mode. For example, if the monitor is set to 1920x1080 pixels, it might only show a section of a much larger framebuffer. Or, if the framebuffer is smaller than the display mode, it might only cover a small area of the monitor, leaving some areas blank. Hence, the pipeline's next stage locates the scanout buffer within the overall screen. In DRM terminology, this is called a plane. It sets the scanout buffer's position, orientation, and scaling factors. Depending on the hardware, there can be multiple active planes using different framebuffers. All active planes feed their pixel output into the pipeline's third stage, which is called the cathode-ray tube controller (CRTC) for historical reasons. 

The CRTC controls everything related to display-mode settings. The DRM driver programs the CRTC hardware with a display mode and connects it with all of its active planes and outputs. There can also be multiple CRTCs with different settings programmed to them. The exact configuration is only limited by hardware features. 

Planes are stacked, so they can overlap each other or cover different parts of the output. According to the programmed display mode and each plane's location, the CRTC hardware fetches pixel data from the planes, blends overlapping planes where necessary, and forwards the result to its outputs. 

Outputs are represented by encoders and connectors. As its name suggests, the encoder is the hardware component that encodes pixel data for an output. An encoder is associated with a specific connector, which represents the physical connection to an output device, such as HDMI or VGA ports with a connected monitor. The connector also provides information on the output device's supported display modes, physical resolution, color space, and the like. Outputs on the same CRTC mirror the CRTC's screen on different output devices. 

The image below shows a simple mode-setting pipeline with an additional plane for the mouse pointer, plus the buffer objects that act as scanout buffers. Arrows indicate the logical flow of pixel data from buffer objects to a VGA connector. This is a typical mode-setting pipeline for an older, discrete graphics card. 

> [ ![\[Mode-setting pipeline\]](https://static.lwn.net/images/2023/kms-pipeline.svg) ](<https://lwn.net/Articles/955751/>)

####  Pipeline setup 

Deciding on policies for connecting and configuring the individual stages of the mode-setting pipeline is not the DRM driver's job. This is left to user-space programs, which brings us back to the compositor. As part of its initial setup, the compositor opens the device file under `/dev/dri`, such as `/dev/dri/card1`, and invokes the respective `ioctl()` calls to program the display pipeline. It also fetches the available display modes from a connector and picks a suitable one. 

After the compositor has finished rendering the first on-screen image, it programs the mode-setting pipeline for the first time. To do so, it creates a framebuffer for the on-screen image's buffer object and attaches the framebuffer to a plane. It then sets the display mode for its on-screen buffer on the CRTC, connects all of the pipeline stages, from framebuffer to connector, and enables the display. 

To change the displayed image in the next frame, no full mode setting is required. The compositor only has to replace the current framebuffer with a new one. This is called page flipping. 

The individual stages of the mode-setting pipeline can be connected in a variety of ways. A CRTC might mirror to multiple encoders, or a framebuffer might be scanned out by multiple CRTCs. While this offers flexibility, it also means that not all combinations of pipeline stages are supported. 

A naive implementation would apply each stage's settings individually. It would first program the display mode in the CRTC, then upload all buffer objects into graphics memory, then set up the framebuffers and planes for scanout, and finally enable the encoders and connectors. If anything fails during this procedure, the screen remains off or (even worse) in a distorted state. For example, with limited device memory, it might not be possible to store the framebuffers for more than one plane at a time. Switching modes, or even simple page flips, might fail. Failing display updates have been a common problem of graphics stacks ever since. 

DRM's atomic mode setting solves this problem to some extent. The mode-setting code tracks the complete state of all elements of the pipeline in a compound data structure called [`drm_atomic_state`](<https://elixir.bootlin.com/linux/latest/source/include/drm/drm_atomic.h#L348>), plus a sub-state for each stage in the pipeline. This mode setting is atomic in the sense that it either applies the full compound state of all pipeline stages, or none of it. To make this work, mode-setting involves two phases: first a check of the complete new atomic state and second, if successfully checked, a commit of the same. 

For checking, the DRM core, its helpers, and the DRM driver test the proposed state against the limitations and constraints of the available graphics hardware. For example, a plane has to verify that the attached framebuffer is of a compatible color format and the CRTC has to verify that the given display resolution is supported by the hardware. If checking succeeds, the DRM driver programs the new state to hardware during the commit phase. If state checking fails for one or more of the stages, DRM stops the mode-setting operations and returns an error to the user-space program. 

So, when our compositor intends to program a display mode, it sets the atomic state of all pipeline stages and applies them all at once. If successful, the display output changes accordingly. For successive page flipping operations, the compositor duplicates the current state, changes the framebuffers to the new ones, and applies the new state. Applying the page flip again results in an atomic-check/atomic-commit sequence within the kernel's DRM code, but with less overhead than a full mode-setting operation. 

DRM's state-checking phase is independent of the hardware's state and does not modify it. If checking an atomic state fails, the compositor receives an error code, but the display output remains unchanged. It is also possible for the compositor to verify atomic states without committing them. This allows building a list of supported configurations beforehand. 

For further reading, the inner workings of atomic mode setting have been covered in detail on LWN back in 2015: [part 1](<https://lwn.net/Articles/653071/>) and [part 2](<https://lwn.net/Articles/653466/>). 

####  Additional features 

In the discussion of planes, it has been assumed that all of the hardware's planes are the same. But that's not always the case. There is usually a plane, called the primary plane, for RGB-like color formats, which covers the whole display. The compositor sets up the primary plane to display its on-screen image. 

But most hardware provides an additional plane for the mouse pointer, called the cursor plane. This plane only covers a small area and floats above the primary plane. As the name suggests, the compositor uses the cursor plane to display the mouse-pointer image, which can now be moved around without changing the primary plane's on-screen image at all. 

Between the primary and cursor plane are overlay planes, which are of various sizes and often support YUV-like color formats. This makes them suitable for displaying streams of video data with low CPU overhead. For that, the video-player application provides the compositor with buffer objects that contain the YUV-based pixel data. 

The compositor sets up the overlay plane with a framebuffer of the pixel data. The plane scans out the YUV pixel data and performs the color conversion to RGB in hardware. Using dma-buf, the video player can forward individual YUV frames from a hardware video decoder directly to the compositor, thus leaving the entire video processing to hardware. 

If latency of the display update is of critical concern, it can be helpful to hand over mode-setting capabilities to a single application. The compositor therefore leases the functionality to the application. While an application holds an active DRM lease, it has full control over the mode-setting pipeline. This is useful for 3D headsets, which need to tightly coordinate the output frequency and latency of their internal displays to make the 3D illusion work. DRM leases expire or can be revoked, so the compositor ultimately remains in control of mode setting. 

While modern compositors use Wayland as their protocol, applications for the X Window System are still common. Xwayland is an X server that runs within a Wayland session. It lets X applications participate in the Wayland sessions transparently by translating between Wayland and X protocols. This works for most use cases. 

The common use case Xwayland cannot emulate is screen capturing and screen sharing. X applications have access to the X session's whole window tree, which makes screen capturing easy. For security purposes, the Wayland protocol does not allow applications to read the screen or other application's windows. Wayland compositors therefore provide dedicated implementations for capturing or sharing the screen's content. [PipeWire](<https://www.pipewire.org/>), [VNC](<https://en.wikipedia.org/wiki/Virtual_Network_Computing>), or [RDP](<https://en.wikipedia.org/wiki/Remote_Desktop_Protocol>) are commonly used for this functionality. 

If no compositor is active, Linux displays a text console. DRM supports the kernel's framebuffer console for text output. This DRM fbdev emulation acts like a DRM client from user space, but runs entirely within the kernel. It also provides the old framebuffer interfaces, such as `/dev/fb0`. Fbdev and DRM's fbdev emulation are on their way to retirement, though. There are [ideas](<https://airlied.blogspot.com/2022/09/lpc-2022-gpu-bof-user-console-and.html>) for moving much of the console functionality to user space. 

At the time of writing this article, one quickly evolving topic for Linux graphics is [High Dynamic Range (HDR) rendering](<https://en.wikipedia.org/wiki/High-dynamic-range_rendering>), which displays the output with more nuanced colors and lighting, thus showing details that are often lost with traditional rendering. Support for this will position Linux to fulfill the needs of professional graphics artists. Currently, support is still uneven, but it's possible to [use HDR in games](<https://twitter.com/Plagman2/status/1610200412854046720?t=5fb1Oi7zgMj8gOWGod8kWA>) and Linux desktops are [beginning to implement HDR](<https://zamundaaa.github.io/wayland/2023/12/18/update-on-hdr-and-colormanagement-in-plasma.html>) as well. 

At this point, we have followed the path of getting the application's content onto the screen in the modern Linux graphics stack—from rendering and memory management to compositing and mode setting. But we've really just scratched the surface. The stack keeps evolving and constantly adds support for new features and hardware.
