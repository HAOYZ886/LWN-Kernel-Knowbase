---
title: Efficient Rust tracepoints
url: https://lwn.net/Articles/992455/
date: "October 8, 2024"
category: "Development tools-Kernel tracing; Development tools-Rust; Releases-6.13"
author: "By Daroc Alden October 8, 2024 Kangrejos 2024"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Daroc Alden**  
October 8, 2024

* * *

[Kangrejos 2024](<https://lwn.net/Articles/990496/>)

Alice Ryhl has been working to enable [ tracepoints](<https://www.kernel.org/doc/html/latest/trace/tracepoints.html>) — which are widely used throughout the kernel — to be seamlessly placed in Rust code as well. She spoke about her approach at Kangrejos. Her [ patch set](<https://lwn.net/ml/all/20241001-tracepoint-v9-0-1ad3b7d78acb@google.com/>) enables efficient use of static tracepoints, but supporting dynamic tracepoints will take some additional effort. 

Ryhl described tracepoints as a kind of logging that records information from specific places in the kernel when they are reached. She gave [ `binder_ioctl()`](<https://elixir.bootlin.com/linux/v6.11.1/source/drivers/android/binder_trace.h#L22>) as an example of a trace event in her [slides](<https://kangrejos.com/2024/Tracepoints.pdf>); that tracepoint is triggered every time an `ioctl()` for Android's binderfs filesystem occurs. A developer trying to debug kernel problems can look at the log of tracepoints hit by a driver to figure out what's happening. 

In C, the programmer places a tracepoint with a line that looks like a normal function call. Most of the time, this call does nothing. When in use, a programmer can attach an arbitrary function to it at run time that will be called when the tracepoint is hit. Since most tracepoints are disabled most of the time, Linux uses [ static keys](<https://docs.kernel.org/staging/static-keys.html#abstract>) (patching the call into the code at run time) to make this efficient. 

Production-ready Rust drivers must be able to support the same standard of debugging, and therefore be able to place tracepoints, Ryhl said. That could be done today, by wrapping existing C tracepoints in Rust wrappers, but this loses one of the most important benefits of tracepoints: their low overhead. Ideally, hitting a disabled tracepoint from Rust should have the same performance cost as C (i.e., almost none). 

Her solution is a small Rust macro that creates the necessary static-key machinery on the Rust side. Rust code uses `declare_trace!()` to refer to a tracepoint defined in C; the macro creates an inline unsafe function on the Rust side that can be used to trigger the tracepoint. The generated function uses inline assembly to define a place for the static-key machinery to patch in a call to the C tracepoint when necessary. 

Ryhl took this approach because it represents implementing the bare minimum in Rust, leaving most of the tracepoint implementation in unchanged C, she said. The static-key functionality has to be implemented on the Rust side for performance, but this way she does not have to reimplement any of the functionality for defining tracepoints, and can instead just link to the C code. 

There is a catch, though. Static keys in C also use inline assembly to create a target for the patched-in jump. In her first attempt, Ryhl copied the inline assembly to use on the Rust side. This was [ rejected](<https://lwn.net/ml/all/20240606172320.GF8774@noisy.programming.kicks-ass.net/>) for introducing code duplication, which is usually frowned upon in the kernel. 

To solve that, Ryhl took the ""horrible"" approach of having a Rust source file generated using the C preprocessor that gets included in the macro. The original C sources have a comment to show where the shared inline assembly is located, and the build system uses `sed` to extract it and put it in the generated Rust file. This avoids any code duplication, at the cost of complicating the build. 

The attendees were a bit surprised at the presented solution. Paul McKenney gave some background information on the reason that kernel developers care so much about avoiding code duplication: in addition to the normal reasons of code quality, it makes rebasing changes much easier. The kernel deals with a lot of patches flying around, and any code that exists in two places can easily get out of sync. Ryhl agreed, saying that there are good reasons not to duplicate code. It made her life difficult, she joked, but she sees why the static-key maintainer insisted. 

Gary Guo said that it is probably not a good idea to use the C preprocessor to generate Rust code. Ryhl replied that it might be possible to generate both the C and Rust from a common format, if that would be preferable. An alternative would be to teach Rust to read C header files itself, but that is much more work. Some other alternate ideas were floated around. McKenney was of the opinion that any approach was acceptable — as long as it actually gets documented, because otherwise all this unusual code-sharing is going to confuse future programmers. 

#### Dynamic tracing

Richard Weinberger asked about dynamic tracepoints ([Kprobes](<https://www.kernel.org/doc/html/latest/trace/kprobes.html>)) — which allow the user to attach a tracepoint anywhere in the code using BPF. Does this work with Rust? Ryhl was unfamiliar with the mechanism. Andreas Hindborg suggested that addressing static tracepoints first, and then looking into dynamic tracepoints later would make sense. Weinberger did think that support for dynamic tracepoints would be needed eventually, because people want their debug tooling to work throughout the whole kernel. 

Ryhl thought that support for dynamic tracing would need to be added to the Rust compiler, based on Hindborg's description of the kernel's [function tracing](<https://www.kernel.org/doc/html/latest/trace/ftrace.html>) code. Static tracepoints would still be needed, however, since they are also used as a way for vendors to hook into the functions of a driver in some cases. (Some Android hardware vendors rely on tracepoints to react to events in the kernel, for example.) Boqun Feng agreed, saying that both kinds of tracepoint were needed for different use cases. Hindborg pointed out that function tracing also interacts strangely with function inlining — finding the location of the hook after inlining depends on having BTF information available. So Rust will need [ native BTF support](<https://lwn.net/Articles/991719/>) before that is possible. 

Hindborg was worried that having a solution which requires defining the tracepoint in C as well will make it harder to have a pure-Rust solution in the future. Ryhl responded that, although she has so far only tackled the declaration of tracepoints in Rust, someone could in the future add the definition of tracepoints as well. 

Despite the discussion of future work, the attendees had no problems with Ryhl's current design. It seems likely that static tracepoints will soon be usable with Rust code in the kernel, which will enable vendor integration with drivers written in Rust. Dynamic tracepoints and other debugging features will take some more time.
