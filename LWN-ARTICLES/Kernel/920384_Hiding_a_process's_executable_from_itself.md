---
title: "Hiding a process's executable from itself"
url: https://lwn.net/Articles/920384/
date: "January 23, 2023"
category: "Security-Security technologies; proc"
author: "By Jonathan Corbet January 23, 2023"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
January 23, 2023

Back in 2019, a [high-profile container vulnerability](<https://lwn.net/Articles/781013/>) led to the adoption of some complex workarounds and a frenzy of patching. The immediate problem was fixed, but the incident was severe enough that security-conscious developers have continued to look for ways to prevent similar vulnerabilities in the future. [This patch set](<https://lwn.net/ml/linux-kernel/20230119170718.3129938-1-gscrivan@redhat.com/>) from Giuseppe Scrivano takes a rather simpler approach to the problem. 

The 2019 incident, which came to be known as CVE-2019-5736, involved a sequence of steps that culminated in the overwriting of the [`runc`](<https://github.com/opencontainers/runc>) container-runtime binary from within a container. That binary should not have even been visible within the container, much less writable, but such obstacles look like challenges to a determined attacker. In this case, the attack was able to gain access to this binary via `/proc/self/exe`, which always refers to the binary executable for the current process. 

Specifically, the attack opens the `runc` process's `/proc/self/exe` file, creating a read-only file descriptor — inside the container — for the target binary, which lives outside that container. Once `runc` exits, the attacker is able to reopen that file descriptor for write access; that descriptor can subsequently be used to overwrite the `runc` binary. Since `runc` is run with privilege outside of the container runtime, this becomes a compromise of the host as a whole; see the above-linked article for details. 

This vulnerability was closed by having `runc` copy its binary image into a memfd area and sealing it; control is then be passed to that image before entering the container. Sealing prevents modifying the image, but even if that protection fails, the container is running from an independent copy of the binary that will never be used again, so overwriting it is no longer useful. It is a bit of an elaborate workaround, but it plugged the hole at the time. 

Scrivano is proposing a different solution to the problem: simply disable access to `/proc/self/exe` as a way of blocking image-overwrite attacks of this type. Specifically, it adds a new [`prctl()`](<https://man7.org/linux/man-pages/man2/prctl.2.html>) command called `PR_SET_HIDE_SELF_EXE` that can be used to disable access to `/proc/self/exe`. Once this option has been enabled, any attempt to open that file from within the owning process will fail with an `ENOENT` error — as if the file simply no longer existed at all. Enabling this behavior is a one-way operation; once it has been turned on, it cannot be disabled until the next [`execve()`](<https://man7.org/linux/man-pages/man2/execve.2.html>) call, which will reset this option to the disabled state, is made. 

This behavior is necessarily opt-in; any program that wants to have its executable image protected from access in this way will have to request it specifically. The intent, though, is that this simple call will be able to replace the more complicated workarounds that are needed to prevent this sort of attack today. A `prctl()` is a small price to pay if it eliminates the need to create a new copy of the executable image every time a new container is launched. 

This new option is thus a simple way of blocking this specific attack, but it leads to some related questions. Hiding the container-runtime binary seems like less satisfying solution than ensuring that this binary cannot be modified regardless of whether it is visible within a container. It seems like it closes off one specific path to a compromise without addressing the underlying problem. 

More to the point, perhaps, is the question of just how many operations the kernel developers would like to add to prevent access to specific resources that might possibly be misused. There is, conceivably, no end to the files (under `/proc` and beyond) that might be useful to an attacker who is determined to take over the system. Adding a new `prctl()` option — and the necessary kernel code to implement it — for every such file could lead to a mess in the long term. There comes a point where it might make more sense to use a security module to implement this sort of policy. 

If the development community feels that way, though, it's not saying so — or much of anything else. The patch set has been posted three times and has not received any substantive comments any of those times. There will, presumably, need to be an eventual discussion that decides whether this type of mechanism is the best way of protecting systems against attacks using `/proc/self/exe`. For the moment, though, it would appear that this change is simply waiting for the wider community to take notice of it.
