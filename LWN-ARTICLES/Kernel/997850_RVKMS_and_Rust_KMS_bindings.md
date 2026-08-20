---
title: RVKMS and Rust KMS bindings
url: https://lwn.net/Articles/997850/
date: "November 20, 2024"
category: "Development tools-Rust; Device drivers-Graphics"
author: "By Jake Edge November 20, 2024 XDC"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jake Edge**  
November 20, 2024

* * *

[XDC](<https://lwn.net/Archives/ConferenceByYear/#2024-X.Org_Developers_Conference>)

At the [2024 X.Org Developers Conference](<https://indico.freedesktop.org/event/6/>) (XDC), Lyude Paul gave a talk on the work she has been doing as part of the [Nova project](<https://gitlab.freedesktop.org/drm/nova>), which is an [effort build an NVIDIA GPU driver in Rust](<https://lwn.net/Articles/990736/>). She wanted to provide an introduction to [RVKMS](<https://lwn.net/ml/all/20240930233257.1189730-36-lyude@redhat.com/>), which is being used to develop Rust kernel mode setting (KMS) bindings; RVKMS is a port of the [virtual KMS](<https://docs.kernel.org/gpu/vkms.html>) (VKMS) driver to Rust. In addition, she wanted to give her opinion on Rust, and why she thinks it is a ""game-changer for the kernel"", noting that the reasons are not related to the oft-mentioned, "headline" feature of the language: memory safety. 

The Nova driver is written in Rust in part because of the lack of a stable firmware ABI for NVIDIA GPU system processors (GSPs). Handling that in C is difficult, Paul said. The inspiration came from the Asahi [driver for Apple GPUs](<https://lwn.net/Articles/995383/>), which uses a similar approach to handle unstable firmware ABIs. In addition, the Nova project can help prove Rust's readiness for the kernel by getting its drivers upstream, which will help make it easier for projects like Asahi get their work upstream as well. 

Writing a kernel driver for a new device is challenging and takes time. For Nova, there is also a need to develop the Rust bindings for a kernel graphics driver. ""Luckily, a lot of this has already been done in Asahi"". There are already lots of bindings available, though they are not yet upstream; doing so entails figuring out if there are changes needed in those bindings and getting them accepted into the kernel. 

The Asahi bindings do not cover kernel mode setting, however, which is surprising; KMS is one of the only parts of that driver that is written in C. So there are no KMS bindings to use for Nova and it is still too early in Nova development to add KMS support to it. On the other hand, though, ""KMS is a large enough surface that we wanted to be able to work on this sooner than later, and ideally in parallel to the rest of Nova"". 

#### RVKMS

So, while Nova was working toward needing KMS, the team decided that Paul would port a KMS driver to Rust in order to create the necessary bindings. VKMS was chosen because ""it's a pretty simple driver, it doesn't require any specific hardware"". VKMS ""pretends to be a display device""; it also supports CRC generation and [writeback connectors](<https://lwn.net/Articles/704647/>), which can be used for testing. 

[ ![\[Lyude Paul\]](https://static.lwn.net/images/2024/xdc-paul-sm.png) ](<https://lwn.net/Articles/998781/>)

For the Rust port, RVKMS, ""it's very early in development, driver-wise; it doesn't do a whole ton yet"". At this point it can basically just ""register a KMS driver and set up [VBLANK](<https://en.wikipedia.org/wiki/Vertical_blanking_interval>) emulation using high-resolution timers"". Eventually, she hopes that the driver will have CRC generation and connector writeback, as well. 

Even though it is still early in RVKMS development, it has already proved ""very useful in making progress with these bindings"". Paul said that she tried to anticipate the needs of other KMS drivers, such as i915 and nouveau, and not just focus on RVKMS, when designing the API. Most of her time has been spent on the bindings, rather than RVKMS itself, which is still quite small. 

There are several goals for the KMS bindings; one is to prevent undefined behavior by using safe code. Another is to make incorrect implementations of the KMS API nearly impossible; ""Rust gives us a lot of tools to actually be able to prove that the way things are implemented are correct at compile time."" The API should be ergonomic, as well; preventing mistakes should not make for code that is messier or more difficult to write. The intention is to mostly only support [atomic mode setting](<https://lwn.net/Articles/653071/>), though there will ""probably be some basic support for the various legacy helpers"" 

#### KMS bindings

The KMS bindings are currently working on top of the direct rendering management (DRM) bindings from Asahi and Nova. Unlike the KMS API in C, the Rust KMS bindings ""are mostly in control of the order of operations during device registration"". In order to support KMS in a Rust driver, it is only necessary to implement the `kernel::drm::kms::Kms` trait, which ""handles calling things in the right order, registering the device, and that sort of thing"". 

Paul then went into a fair amount of detail on the KMS bindings, which I will try to relay, though my graphics and Rust knowledge may not be fully up to the task. The [YouTube video](<https://www.youtube.com/watch?v=ckUx3otJ7FU>) of the talk and her [slides](<https://indico.freedesktop.org/event/6/contributions/304/attachments/233/314/RVKMS%20XDC2024.pdf>) will be of interest to those seeking more information. Background material on the Linux graphics stack can be found in [part one](<https://lwn.net/Articles/955376/>) of our two-part series looking at it; for this talk, [part two](<https://lwn.net/Articles/955708/>) may be the most relevant piece. The [Wikipedia article on DRM](<https://en.wikipedia.org/wiki/Direct_Rendering_Manager>) and its [section on the KMS device model](<https://en.wikipedia.org/wiki/Direct_Rendering_Manager#KMS_device_model>) may also be useful, especially for some of the terminology. 

There are two main parts to the `Kms` trait, she said. `mode_config_info()` is used for static information, like minimum and maximum resolution, various cursor capabilities, and others. `create_objects()` provides ""access to a special `UnregisteredKmsDevice` type"" that can be used to create both static (e.g. "cathode-ray-tube controller" (CRTC), plane) and non-static (e.g. connectors) objects. In the future, hooks for customizing the initial mode setting will likely be added, but those are not needed for the virtual display provided by RVKMS. 

""One of the neat things"" with the bindings is that drivers implementing the `Kms` trait, get a `KmsDriver` trait implemented automatically. That allows KMS-dependent methods to only be available to drivers that actually implement `Kms`. So all of the bindings can just assume that KMS is always present and set up, instead of having run-time checking and adding error paths. 

#### Mode objects

DRM has the concept of a "mode object" that is exposed to user space through an object ID. Mode objects can have a reference count and be created at any time, or not have a reference count, but those can only be created before driver registration. The `ModeObject` trait is used to represent them. Reference-counted objects fit in nicely with Rust lifetime requirements; an `RcModeObject` trait is used for those to reduce the reference-counting boilerplate needed. 

Static objects, such as CRTCs and planes, typically share the lifetime of a device and are more challenging to handle because that does not easily map to Rust lifetimes. The `StaticModeObject` and `KmsRef` traits are used for those types of objects; `KmsRef` acts as a reference count on the parent device, while allowing access to the static object, which allows owned references to the static objects. 

Implementing CRTCs, planes, and other components of that sort turned out to be ""a bit more complicated than one might expect"", she said. Most drivers do not use the DRM structures unmodified, and instead embed them into driver-private structures; for example, in VKMS, the `vkms_crtc` structure embeds `drm_crtc`. They contain and track driver-private information, including display state and static information. Drivers often have multiple subclasses of these types of objects; for example, both i915 and nouveau have multiple types of connectors, encoders, and others. 

It turns out that ""this is not the first time we've had to do something like this""; Asahi had to do something similar for its [Graphics Execution Manager](<https://en.wikipedia.org/wiki/Direct_Rendering_Manager#Graphics_Execution_Manager>) (GEM) support. In GEM infrastructure, this type of subclassing, where driver-private data is maintained with the object, is common. The needs for KMS subclassing are more variable than for GEM, because the technique is used more widely, but the Asahi work provided a good starting point, she said. 

In the KMS bindings, there are traits for the different object types, such as `DriverCrtc` and `DriverEncoder`; drivers can have multiple implementations of them as needed. Driver data can be stored in the objects either by passing immutable data to the constructor or at any other point using [send and sync containers](<https://doc.rust-lang.org/nomicon/send-and-sync.html>). KMS drivers typically switch between the common representation (e.g. `drm_crtc`) and the driver-specific one (`vkms_crtc`), which is also possible with the KMS Rust bindings. There are some operations that should apply to all instances of the class and others that are only for the specific subclass. So there is a ""fully-typed interface"" that provides access to the private data and the common DRM methods and an opaque interface that only provides access to the common methods. 

The same mechanism is used for atomic states, with fully-typed and opaque interfaces, which can be switched between at run time. If access to the private data is needed, objects can be fallibly converted to fully-typed. That required support for consistent [vtable](<https://en.wikipedia.org/wiki/Virtual_method_table>) memory locations, ""which is not something that Rust has by default"", since constants are normally inlined, rather than stored as static data. A Rust macro (`#[unique]`) was added to make that work. 

#### Atomic commits

""Things diverge a bit"" for atomic commits due to Rust's requirements. The Rust data-aliasing rules allow having an infinite number of immutable references to an object or a single mutable reference at any given time. If the atomic callbacks for checking, updating, and the like only affected the object they were associated with, it would be easy to handle, but that is not the case. The callbacks often iterate through the state of other objects, not just the one that the callback belongs to. 

She originally started implementing the callbacks using just references, but that did not really work at all. Instead, she took inspiration from [`RefCell`](<https://doc.rust-lang.org/beta/book/ch15-05-interior-mutability.html>), which is a ""Rust API for handling situations where the data-aliasing rules aren't exactly ideal"". Mutable and immutable borrows still exist, but they are checked at run time rather than compile time. 

When working with the atomic state, most of the code will use the `AtomicStateMutator` container object, which is a wrapper around an `AtomicState` object. There are always immutable references to the container available, and it manages handing out borrows for callbacks that want to examine or change the state. There can only be a single borrow for each state, but a callback can hold borrows for multiple states. Borrowing is fallible, but the interface is meant to be ergonomic; for example, callbacks are made with a pre-borrowed state, so that the callback does not need to obtain it. 

In order to enforce the order of operations and protect states from mutation once they are made visible outside of the atomic commit, the bindings use the [typestate](<https://docs.rs/typestate/latest/typestate/>) pattern. This is a feature that is not unique to Rust, but is not common in other languages; ""Rust generally makes it a lot easier to work with than other languages"". It allows the bindings to ""encode the run-time state of something into compile-time state""; the idea is that the object is represented by a different type at every stage of its lifetime. It provides ""a very powerful tool to actually enforce API correctness"", Paul said. 

For example, `AtomicCommitTail` is an `AtomicState` wrapper that lets the driver developer control the order in which commits are executed. It does so mostly by using tokens for each step of the process; the tokens prove that a certain prerequisite has been done. The checking is done at compile time and ""it lets you make it impossible to write an incomplete `atomic_commit_tail()` [callback] that actually compiles"". The code has to ""perform every step and you have to perform them in the correct order, otherwise the code just doesn't compile"". 

KMS drivers have lots of optional features, she said; for example, VBLANK is used everywhere to some extent, but some hardware does not have a VBLANK interrupt, so it must be emulated in the DRM core. The Rust bindings can use traits to only allow drivers that implement VBLANK to access the appropriate methods; other drivers will not be able to call those methods. If it implements the `DriverCrtcVblank` trait, it will have access to the VBLANK-exclusive methods; that pattern can be extended for other optional pieces of functionality. 

Paul closed the first part of her talk with thanks to various people and groups who have helped make RVKMS and the KMS bindings possible: the Asahi project, Maíra Canal, and her co-workers at Red Hat working on Nova. From there, she moved on to talk about her experience with Rust. 

#### Rust experiences

""I won't be talking about memory safety"", she said; one of the big mistakes made when people are trying to advocate for Rust is to entirely focus on memory safety. Kernel developers already know that C is unsafe, so pushing hard on the memory-safety point often sounds like the Rust advocates are talking down to the kernel developers. That is one of the reasons that she avoided looking at Rust for years. Instead, she believes that there are more compelling arguments for bringing Rust to the kernel. 

""Rust can be a kernel maintainer""; a huge part of being a maintainer is to stop bad patterns in the code. That is time-consuming, and requires constantly re-explaining problems, while hoping nothing important was missed. ""It can make you snippy; it can burn through your motivation"". 

Rust can help with that, because it provides a lot of tools to enforce code patterns that would have needed to be corrected over email. It is ""a lot more capable than anything we were really ever able to do in C"". The uses of the typestate pattern are a good example of that; they have little, usually no, run-time cost. There is an upfront cost to Rust, in learning the language and in rethinking how code is written to fit into the new model, but ""the potential for saving time long term is kind of astounding"". 

People often wonder about how to work with unsafe code, but its presence does not really change much in her experience. For one thing, unsafe code also acts as an enforcement tool; a "safety contract" must be present in the comments for unsafe code or the compiler will complain. That requires those writing unsafe code to think about and document why and how they are violating the language invariants, which gives reviewers and maintainers something to verify. Unsafe acts as a marker for a place where more scrutiny is needed. 

""It's sort of wild what the end result of this is""; when writing RVKMS, she spent almost no time debugging: around 24 hours over a few months of development. Writing drivers in C has always been a loop of adding a bunch of code, then spending a day or more debugging problems of various sorts (missed null checks, forgotten initialization, thread-safety issues, etc.), and going back to adding code. That is not how things go with Rust; ""if things compile, a lot of times it will actually work, which is a very weird concept and is almost unbelievable until you've actually dealt with it yourself"". 

Before Paul started working with Rust, she was put off by a lot of the patterns used, such as a lack of null, having to always handle option returns, and ""tons of types, that sounds kind of like a nightmare"". It turns out that ""Rust is ergonomic enough"" that you end up not really thinking about those things once a set of bindings has been developed. Much of the time, it also ""almost feels obvious what the right design is"". Most of the Rust constructs have lots of shortcuts for making them ""as legible and simple as possible"". Once you get past the design stage, you rarely need to think about all of the different types; ""a lot of the time, the language just sort of handles it for you"". 

She is not a fan of comparisons to C++, in part because ""Rust is kind of a shockingly small language"". It is definitely complicated and difficult to ""wrap your head around at first"", but its scope is limited, unlike C++ and other languages, which feel more like a framework than a language, she said. The Rust standard library is built around the "keep it simple, stupid" (KISS) philosophy, but it is also constantly being iterated on to make it easier to use, while not sacrificing compatibility. Once you get used to the Rust way of doing things, the correct way to do something generally feels like the obvious way to do it as well. 

She concluded her talk with a question: ""would you rather repeat yourself on the mailing list a million times"" to stop the same mistakes, ""or would you rather just have the compiler do it?"" She suggested: ""Give Rust a try"". 

#### Q&A

An audience member asked about how the Rust code would fare in the face of changes to the DRM API in the kernel. Paul said that refactoring Rust code ""tends to be very easy, even with a lot of subtly more complicated changes than you might have to work around in C"". It is not free, of course, but refactoring in Rust is not any harder than it is for C. 

Another question was about Rust development finding problems in the existing C APIs and code; Paul said that has happened and she thinks Rust is helpful in that regard because it forces people to clearly think things through. DRM, though, has been pretty well thought-out, she said, so most of what she has seen has been elsewhere in the kernel; in the response to a separate question, she reiterated that DRM was never really an impediment to the Rust work, in part because it is so well designed and documented. 

Adding functionality to DRM using Rust was also asked about; does it make sense to do so? Paul said that it would make sense because Rust forces the developer to think about things up front, rather than to just get something working quickly and deal with locking or other problems as they arise. That leads to the "if it compiles, it will likely work" nature of Rust code. But, calling Rust from C is difficult, at least for now, so that would limit the ability to use any new Rust features from existing C drivers and other code. 

Another question was about getting started today on a KMS driver; would she suggest doing that in C or in Rust? For now, she would recommend C, though that may change eventually. The problem is that there are a lot of missing bindings at this point and whenever she adds functionality to RVKMS, she ends up adding more bindings. Designing bindings requires more overall knowledge of DRM and other KMS drivers in addition to Rust itself. Once most of the bindings are available, though, starting out with Rust will be a reasonable approach. 

The last question was about compile time, which is often a problem for larger Rust projects. Paul said that she was ""actually surprisingly happy"" with the compile time at this point, but it is probably too early to make that determination. Once more Rust code is added into the mix, that will be when the compile-time problem pops up. 

[ I would like to thank LWN's travel sponsor, the Linux Foundation, for travel assistance to Montreal for XDC. ]
