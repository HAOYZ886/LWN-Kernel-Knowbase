---
title: Android kernel notes from LPC 2020
url: https://lwn.net/Articles/830979/
date: "September 10, 2020"
category: Android
author: "By Jonathan Corbet September 10, 2020 LPC"
---

By **Jonathan Corbet**  
September 10, 2020

* * *

[LPC](<https://lwn.net/Archives/ConferenceByYear/#2020-Linux_Plumbers_Conference>)

In its early days, the Android project experienced a high-profile disconnect with the kernel community. That situation has since improved considerably, but there are still differences between Android kernels and the mainline. As a result, it is not possible to run Android on a vanilla kernel. That situation continues to improve, though; much evidence to that effect was on display during the Android microconference at the 2020 [Linux Plumbers Conference](<https://linuxplumbersconf.org/>). Several sessions there showed the progress that is being made toward unifying the Android and mainline kernels — and the places where there is still some work to be done. 

#### The generic kernel image

Todd Kjos started things off by introducing the Android Generic Kernel Image (GKI) effort, which is aimed at reducing Android's kernel-fragmentation problem in general. It is the next step for the Android Common Kernel, which is based on the mainline long-term support (LTS) releases with a number of patches added on top. These patches vary from Android-specific, out-of-tree features to fixes cherry-picked from mainline releases. The end result is that the Android Common Kernel diverges somewhat from the LTS releases on which it is based. 

From there, things get worse. Vendors pick up this kernel and apply their own changes — often significant, core-kernel changes — to create a vendor kernel. The original-equipment manufacturers begin with that kernel when [![\[Todd Kjos\]](https://static.lwn.net/images/conf/2020/lpc/ToddKjos-sm.png)](<https://lwn.net/Articles/830981/>) creating a device based on the vendor's chips, but then add changes of their own to create the OEM kernel that is shipped with a device to the consumer. The end result of all this patching is that every device has its own kernel, meaning that there are thousands of different "Android" kernels in use. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

There are a lot of costs to this arrangement, Kjos said. Fragmentation makes it harder to ensure that all devices are running current kernels — or even that they get security updates. New platform releases require a new kernel, which raises the cost of upgrading an existing device to a new Android version. Fixes applied by vendors and OEMs often do not make it back into the mainline, making things worse for everybody. 

The Android developers would like to fix this fragmentation problem; the path toward that goal involves providing a single generic kernel in binary form (the GKI) that all devices would use. Any vendor-specific or device-specific code that is not in the mainline kernel will need to be shipped in the form of kernel modules to be loaded into the GKI. That means that Android is explicitly encouraging vendor modules, Kjos said; the result is a cleaner kernel without the sorts of core-kernel modifications that ship on many devices now. 

This policy has already resulted in more vendors actively working to upstream their code. That code often does not take the form that mainline developers would like to see; some of it is just patches exporting symbols. That has created some tension in the development community, he said. 

He concluded by saying that the Android 11 release requires all devices to ship with kernels based on the Android Common Kernel; Android 12 will require shipping with the GKI instead. Tim Bird asked how vendors plan to cope when a patch they need isn't integrated into the mainline or the Android Common Kernel; Kjos answered that the current plan is to add vendor hooks via tracepoints. The details, though, have not yet been worked out. 

#### ABI enforcement

Later, Matthias Männich talked about GKI ABI enforcement, the purpose of which is to ensure a stable ABI for modules so that GKI updates do not end up breaking devices in the field. This is not a simple task; the kernel ABI is large, and it is hard to catch changes in every part of it. He emphasized that this work is in no way trying to stabilize the [![\[Matthias Männich\]](https://static.lwn.net/images/conf/2020/lpc/MatthiasMannich-sm.png)](<https://lwn.net/Articles/830982/>) _mainline_ kernel ABI, or even the ABI for LTS kernels. It is only intended to keep the kernel ABI stable within a specific Android version. 

While ABI changes are not welcome in GKI updates, configuration changes are allowed as long as they don't change the interface as seen by modules. The kernel and modules are all built with a single toolchain using a "hermetic build" process wherein all needed libraries are provided independently of the system the kernel is built on. Compiler updates are carefully examined to ensure that they will not result in any ABI changes; Android would rather not upgrade than risk problems, he said. 

Within the ABI itself, the goal is to keep everything that is observable stable. That task is obviously easier if the set of observable aspects is minimized; [kernel symbol namespaces](<https://lwn.net/Articles/760045/>) help in that regard. They also help to prevent kernel symbols from being used accidentally. The kernel-module interface is established by looking at the symbols that are actually used by vendor modules; those naturally have to be exported. Everything that turns out not to be used is trimmed from the GKI, though, making it unavailable. When a vendor needs a new symbol, a request is made to the Android Open Source Project; assuming the request makes sense, the symbol will appear in a subsequent GKI update.
