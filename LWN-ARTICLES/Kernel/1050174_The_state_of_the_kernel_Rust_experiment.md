---
title: The state of the kernel Rust experiment
url: https://lwn.net/Articles/1050174/
date: "December 13, 2025"
category: "Development tools-Rust"
author: "By Jonathan Corbet December 13, 2025 Maintainers Summit"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
December 13, 2025

* * *

[Maintainers Summit](<https://lwn.net/Articles/1049982/>)

The ability to write kernel code in Rust was explicitly added as an experiment — if things did not go well, Rust would be removed again. At the 2025 Maintainers Summit, a session was held to evaluate the state of that experiment, and to decide whether the time had come to declare the result to be a success. The (arguably unsurprising) conclusion was that the experiment is indeed a success, but there were some interesting points made along the way. 

#### Headlines

Miguel Ojeda, who led the session, started with some headlines. The Nova driver for NVIDIA GPUs is coming, with pieces already merged into the mainline, and the [Android binder driver](<https://lwn.net/Articles/953116/>) was merged for 6.18. Even bigger news, he said, is that Android 16 systems running the 6.12 kernel are shipping with the Rust-written [ashmem](<https://rust-for-linux.com/android-%60ashmem%60>) module. So there are millions of real devices running kernels with Rust code now. 

[![\[Miguel Ojeda\]](https://static.lwn.net/images/conf/2025/ossjp/MiguelOjeda-sm.png)](<https://lwn.net/Articles/1050175/>) Meanwhile, the Debian project has, at last, enabled Rust in its kernel builds; that will show up in the upcoming "forky" release. The amount of Rust code in the kernel is ""exploding"", having grown by a factor of five over the last year. There has been an increase in the amount of cooperation between kernel developers and Rust language developers, giving the kernel project significant influence over the development of the language itself. The Rust community, he said, is committed to helping the kernel project. 

The [rust_codegen_gcc](<https://github.com/rust-lang/rustc_codegen_gcc?tab=readme-ov-file#wip-libgccjit-codegen-backend-for-rust>) effort, which grafts the GCC code generator onto the rustc compiler, is progressing. Meanwhile the fully GCC-based [gccrs](<https://github.com/Rust-GCC/gccrs?tab=readme-ov-file#gcc-rust>) project is making good progress. Gccrs is now able to compile the kernel's Rust code (though, evidently, compiling it to _correct_ runnable code is still being worked on). The gccrs developers see building the kernel as one of their top priorities; Ojeda said to expect some interesting news from that project next year. 

With regard to Rust language versions, the current plan is to ensure that the kernel can always be built with the version of Rust that ships in the Debian stable release. The kernel's minimum version would be increased 6-12 months after the corresponding Debian release. The kernel currently specifies a minimum of Rust 1.78, while the current version is (as of the session) 1.92. Debian is shipping 1.85, so Ojeda suggested that the kernel move to that version, which would enable the removal of a number of workarounds. 

Jiri Kosina asked how often the minimum language version would be increased; Ojeda repeated that it would happen after every Debian stable release, though that could eventually change to every other Debian release. It is mostly a matter of what developers need, he said. Linus Torvalds said that he would be happy to increase the minimum version relatively aggressively as long as it doesn't result in developers being shut out. Distributors are updating Rust more aggressively than they have traditionally updated GCC, so requiring a newer version should be less of a problem. 

Arnd Bergmann said that the kernel could have made GCC 8 the minimum supported version a year earlier than it did, except that SUSE's SLES was running behind. Kosina answered that SUSE is getting better and shipping newer versions of the compiler now. Dave Airlie worried that problems could appear once the enterprise distributors start enabling Rust; they could lock in an ancient version for a long time. Thomas Gleixner noted, though, that even Debian is now shipping GCC 14; the situation in general has gotten better. 

#### Still experimental?

Given all this news, Ojeda asked, is it time to reconsider the "experimental" tag? He has been trying to be conservative about asking for that change, but said that, with Android shipping Rust code, the time has come. Airlie suggested making the announcement on April 1 and saying that the experiment had failed. More seriously, he said, removing the "experimental" tag would help people argue for more resources to be directed toward Rust in their companies. 

Bergmann agreed with declaring the experiment over, worrying only that Rust still ""doesn't work on architectures that nobody uses"". So he thought that Rust code needed to be limited to the well-supported architectures for now. Ojeda said that there is currently good support for x86, Arm, Loongarch, RISC-V, and user-mode Linux, so the main architectures are in good shape. Bergmann asked about PowerPC support; Ojeda answered that the PowerPC developers were among the first to send a pull request adding Rust support for their architecture. 

Bergmann persisted, asking about s390 support; Ojeda said that he has looked into it and concluded that it should work, but he doesn't know the current status. Airlie said that IBM would have to solve that problem, and that it will happen. Greg Kroah-Hartman pointed out the Rust upstream supports that architecture. Bergmann asked if problems with big-endian systems were expected; Kroah-Hartman said that some drivers were simply unlikely to run properly on those systems. 

With regard to adding core-kernel dependencies on Rust code, Airlie said that it shouldn't happen for another year or two. Kroah-Hartman said that he had worried about interactions between the core kernel and Rust drivers, but had seen far fewer than he had expected. Drivers in Rust, he said, are indeed proving to be far safer than those written in C. Torvalds said that some people are starting to push for CVE numbers to be assigned to Rust code, proving that it is definitely not experimental; Kroah-Hartman said that no such CVE has yet been issued. 

The DRM (graphics) subsystem has been an early adopter of the Rust language. It was still perhaps surprising, though, when Airlie (the DRM maintainer) said that the subsystem is only ""about a year away"" from disallowing new drivers written in C and requiring the use of Rust. 

Ojeda returned to his initial question: can the "experimental" status be ended? Torvalds said that, after nearly five years, the time had come. Kroah-Hartman cited the increased support from compiler developers as a strong reason to declare victory. Steve Rostedt asked whether function tracing works; the answer from Alice Ryhl was quick to say that it does indeed work, though ""symbol demangling would be nice"". 

Ojeda concluded that the ability to work in Rust has succeeded in bringing in new developers and new maintainers, which had been one of the original goals of the project. It is also inspiring people to do documentation work. There are a lot of people wanting to review Rust code, he said; he is putting together a list of more experienced developers who can help bring the new folks up to speed. 

The session ended with Dan Williams saying that he could not imagine a better person than Ojeda to have led a project like this and offered his congratulations; the room responded with strong applause.
