---
title: Monitoring the internal kernel ABI
url: https://lwn.net/Articles/800452/
date: "September 25, 2019"
category: "Android-Generic kernel image"
author: "By Jake Edge September 25, 2019 LPC"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jake Edge**  
September 25, 2019

* * *

[LPC](<https://lwn.net/Archives/ConferenceByYear/#2019-Linux_Plumbers_Conference>)

As part of the [Distribution Kernels microconference](<https://linuxplumbersconf.org/event/4/sessions/48/#20190909>) at Linux Plumbers Conference 2019, Matthias Männich described how the Android project monitors changes to the internal kernel ABI. As Android kernels evolve, typically by adding features and bug fixes from more recent kernel versions, the project wants to ensure that the ABI remains the same so that out-of-tree modules will still function. While the talk was somewhat Android-specific, the techniques and tools used could be applied to other distributions with similar needs (e.g. enterprise distributions). 

Männich is on the Google Android kernel team, but is relatively new to the kernel; his background is in build systems and the like. He stressed that he is not talking about the user-space ABI of the kernel, but the ABI and API that the kernel exposes to modules. The idea is to have a stable ABI over the life of an Android kernel. He knows that other distributions have been doing this "for ages", but the Android kernel and build system are different so it made sense to look at other approaches to this problem. 

#### Out of tree

It is sometimes impossible to have everything in-tree, he said, which is part of the motivation for this work. Stabilizing the ABI will also decouple the development of the kernel and modules for it. The hope is to reduce the fragmentation in the Android space by reducing the number of different kernel versions out there while providing a single ABI/API for the module ecosystem. 

[ ![\[Matthias Männich\]](https://static.lwn.net/images/2019/lpc-mannich-sm.jpg) ](<https://lwn.net/Articles/800484/>)

Starting as part of Android 8, [Project Treble](<https://lwn.net/Articles/765467/>) decoupled the vendor-specific parts of Android from the rest of the stack. But that resulted in a big conglomeration of vendor drivers and kernel common code, so it did not really fully decouple the two. Since then, the kernel piece has been separated into the generic kernel image (GKI), along with GKI modules that are common, and the hardware-specific drivers that access the GKI via a stable ABI/API. 

The stable interfaces are not something that is wanted upstream, he said. Maintaining stable interfaces will not be done for the mainline; the intent is to do it for trees based on the stable long-term support (LTS) branches. Dhaval Giani asked if the plan was to have the same interface for, say, both 4.9.x and 4.14.x, but Männich said that the intent was only for it to apply to a single LTS branch. So, for example, all Android kernels in the 4.19.x series would be compatible, but not with those in a 5.x kernel. 

K. Y. Srinivasan asked if Google had given up on the idea of forcing everyone to put their code into the tree. Männich said that the company encouraged that. Greg Kroah-Hartman said that he "would love for everybody to be in the tree"; "talk to Qualcomm, please". The Android project is unfortunately working with vendors that are not in the tree, he said; he is working on that problem independently, "but we also have to deal with the real world". Tim Bird wondered if this plan would impose any constraints on the kinds of changes that would be accepted into the LTS branches, but Kroah-Hartman said that it would not. 

In order to make this work, Android will need to find a kernel configuration that works for all of the vendors, Männich said. Android is still one step shy of having reproducible builds as it is still working on hermetic kernel builds, where all of the toolchains and dependencies, including utilities like `uname`, are packaged and used separately from the underlying system where the kernel is being built. 

To reduce the scope of the problem, it is important to have ways to define what is and is not part of the ABI, he said. There will be whitelists and blacklists to facilitate that. There may be other mechanisms as well. 

Currently, Android is only targeting the android-4.19 and android-5.x series—x has not yet been decided—for the stable ABI. There will be one GKI configuration for the kernel, though it may be somewhat different for each architecture that is supported. It only targets Clang builds in a hermetic environment, so the compiler and other tools cannot change over the life of the Android kernel. 

In terms of the scope, the stable ABI only applies to the observable ABI. Instead of looking at the code, the project looks at the binary of the kernel to determine what the ABI is. The developers are working on whitelists and he is hopeful that [symbol namespaces](<https://lwn.net/Articles/760045/>) get merged so that parts of the stable ABI can be defined in terms of which namespaces are supported. 

An attendee wondered if other distributions actually cared that much about the stable ABI problem. Several attendees answered that some did because they had customers who care. In some cases, like a popular desktop graphics driver, the source is not available to just rebuild the module for a new kernel, Laura Abbott said. Developers of those out-of-tree drivers can and do update the drivers, but if a distribution wants to ensure the drivers simply keep on working, enforcing a stable ABI would do that, she said. 

#### libabigail

Android uses [libabigail](<https://sourceware.org/libabigail/>) to analyze the kernel ABI. Libabigail is both a library and a set of tools; Android mostly just uses the tools to extract and serialize/deserialize the ABI description from the kernel and module binaries. Originally, libabigail only used the ELF and DWARF information, but more recently has added support for the kernel by looking at `ksymtab` rather than the ELF symbol table. It will generate an in-memory data structure that describes the ABI that it finds; that data structure can be serialized to XML, which can then be compared to previous or future versions of the ABI. 

Bird wondered if this tool should be added to the upstream kernel Makefile. Kroah-Hartman and Männich agreed that it would add a kernel dependency on an external tool, which is probably not desirable. It is easy to simply invoke the tool on the kernel build tree after it is built, Männich said. 

Giani asked whether the entire observable ABI needs to match between versions. That is where suppression and whitelists come into play, Männich said. Giani suggested that a full whitelist approach might be the way to go since the Android project knows all of the drivers and hardware-specific pieces that it wants to support. Otherwise, it risks growing the supported ABI to an unnecessarily large size. 

Männich said that the configuration of the Android kernel is not terribly large. It is much smaller than a standard distribution configuration. In addition, he is hoping to see symbol namespaces, which will make it even easier to pick and choose pieces to use. The problem with a whitelist-only approach is that certain parts of the ABI may obviously not be of interest, for example the filesystem interfaces, but they may define structures and types that are used elsewhere, another Android team member said. So the process has been to try to remove pieces to bring the size of the stable ABI down. 

Ben Hutchings asked about ABI changes that are still backward compatible; how are those handled? Männich said that some of that is still a work in progress. Libabigail maintainer Dodji Seketeli said that there are suppressions available that he likened to those for [Valgrind](<http://www.valgrind.org>). The suppressions can indicate changes that are known not to be problematic from an ABI standpoint. 

Sasha Levin asked about kernel changes that do not manifest as ABI changes, such as locking semantics; can those be represented and tracked? Männich said that there are some things that cannot currently be handled, but that they are being worked on; he pointed to an example of an untagged `enum` value being returned as an integer from a function. If the enum values are rearranged, it changes the ABI but is not flagged by libabigail. Seketeli said that all types could be added to the ABI that the tool is tracking, not just those that appear in function signatures, but that they are not right now to keep memory usage down. 

In general, things like locking semantics don't change in an LTS branch, the other Android project member said. If you care about locking semantics, you have to care about all of the ABI semantics; there will likely sometimes be problems in that area, but the project will have to find them on its own as the tooling is not going to help, he said. 

As time for the session ran out, Männich quickly went over how the ABI tooling is integrated into the Android process. Basically everyone who builds an Android kernel will get the toolchain and tools, including libabigail, as part of the "`repo sync`" command to update their tree. The ABI generation and a diff against the baseline ABI will be run as part of the overall build process; any changes to the ABI will then bubble up to the [Gerrit](<https://www.gerritcodereview.com>) code review tool that Android uses. The tools are pretty generic, so they should be easily integrated into other workflows. 

[I would like to thank LWN's travel sponsor, the Linux Foundation, for travel assistance to Lisbon for LPC.]
