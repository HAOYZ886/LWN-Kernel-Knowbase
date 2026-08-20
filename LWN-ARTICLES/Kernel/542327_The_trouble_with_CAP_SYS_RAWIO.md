---
title: The trouble with CAP_SYS_RAWIO
url: https://lwn.net/Articles/542327/
date: "March 13, 2013"
category: Capabilities
author: "By Michael Kerrisk March 13, 2013"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Michael Kerrisk**  
March 13, 2013

A February linux-kernel mailing list discussion of a patch that extends the use of the `CAP_COMPROMISE_KERNEL` capability soon evolved into a discussion of the specific uses (or abuses) of the `CAP_SYS_RAWIO` capability within the kernel. However, in reality, the discussion once again exposes some general difficulties in the Linux capabilities implementation--difficulties that seem to have no easy solution. 

The discussion began when Kees Cook [submitted a patch](<https://lwn.net/Articles/542328/>) to guard writes to [model-specific registers](<http://en.wikipedia.org/wiki/Model-specific_register>) (MSRs) with a check to see if the caller has the `CAP_COMPROMISE_KERNEL` capability. MSRs are x86-specific control registers that are used for tasks such as debugging, tracing, and performance monitoring; those registers are accessible via the [`/dev/cpu/CPUNUM/msr`](<http://man7.org/linux/man-pages/man4/msr.4.html>) interface. [`CAP_COMPROMISE_KERNEL`](<https://lwn.net/Articles/523367/>) (formerly known as [`CAP_SECURE_FIRMWARE`](<https://lwn.net/Articles/514985/>)) is a new capability designed for use in conjunction with UEFI secure boot, which is a mechanism to ensure that the kernel is booted from an on-disk representation that has not been modified. 

If a process has the `CAP_COMPROMISE_KERNEL` capability, it can perform operations that are not allowed in a secure-boot environment; without that capability, such operations are denied. The idea is that if the kernel detects that it has been booted via the UEFI secure-boot mechanism, then this capability is disabled for all processes. In turn, the lack of that capability [is intended](<https://lwn.net/Articles/534637/>) to prevent operations that can modify the running kernel. `CAP_COMPROMISE_KERNEL` is not yet part of the mainline kernel, but already exists as [a patch in the Fedora distribution](<http://pkgs.fedoraproject.org/cgit/kernel.git/tree/secure-boot-20130218.patch>) and Matthew Garrett is working towards its inclusion in the mainline kernel. 

H. Peter Anvin [wondered](<https://lwn.net/Articles/542330/>) whether `CAP_SYS_RAWIO` did not already suffice for Kees's purpose. In response, Kees [argued](<https://lwn.net/Articles/542334/>) that `CAP_SYS_RAWIO` is for governing _reads_ : ""writing needs a much stronger check"". Kees [went on](<https://lwn.net/Articles/542335/>) to elaborate: 

there's a reasonable distinction between systems that expect to strictly enforce user-space/kernel-space separation (`CAP_COMPROMISE_KERNEL`) and things that are fiddling with hardware (`CAP_SYS_RAWIO`). 

This in turn led to a short discussion about whether a capability was the right way to achieve the goal of restricting certain operations in a secure-boot environment. Kees [was inclined](<https://lwn.net/Articles/542455/>) to think it probably was the right approach, but deferred to Matthew Garrett, implementer of much of the secure-boot work on Fedora. Matthew thought that a capability approach seemed the best fit, but [noted](<https://lwn.net/Articles/542456/>): 

I'm not wed to [a capability approach] in the slightest, and in fact it causes problems for some userspace (anything that drops all capabilities suddenly finds itself unable to do something that it expects to be able to do), so if anyone has any suggestions for a better approach…

In the current mainline kernel, the `CAP_SYS_RAWIO` capability is checked in the `msr_open()` function: if the caller has that capability, then it can open the MSR device and perform reads and writes on it. The purpose of Kees's patch is to add a `CAP_COMPROMISE_KERNEL` check on each write to the device, so that in a secure-boot environment the MSR devices are readable, but not writeable. The problem that Matthew alludes to is that this approach has the potential to break user space because, formerly, there was no capability check on MSR writes. An application that worked prior to the introduction of `CAP_COMPROMISE_KERNEL` can now fail in the following scenario: 

  * The application has a full set of privileges. 

  * The application opens an MSR device (requires `CAP_SYS_RAWIO`). 

  * The application drops all privileges, including `CAP_SYS_RAWIO` and `CAP_COMPROMISE_KERNEL`. 

  * The application performs a write on the previously opened MSR device (requires `CAP_COMPROMISE_KERNEL`). 

The last of the above steps would formerly have succeeded, but, with the addition of the `CAP_COMPROMISE_KERNEL` check, it now fails. In a subsequent reply, Matthew noted that [QEMU](<http://wiki.qemu.org/>) was one program [that was broken](<https://bugzilla.redhat.com/show_bug.cgi?id=908888>) by a scenario similar to the above. Josh Boyer [noted](<https://lwn.net/Articles/542464/>) that Fedora has had a few reports of applications breaking on non-secure-boot systems because of scenarios like this. He highlighted why such breakages are so surprising to users and why the problem is seemingly unavoidable: 

… the general problem is people think dropping all caps blindly is making their apps safer. Then they find they can't do things they could do before the new cap was added…

Really though, the main issue is that you cannot introduce new caps to enforce finer grained access without breaking something. 

Shortly afterward, Peter stepped back [to ask](<https://lwn.net/Articles/542468/>) a question about the bigger picture: why should `CAP_SYS_RAWIO` be allowed on a secure-boot system? In other words, rather than adding a new `CAP_COMPROMISE_KERNEL` capability that is disabled in secure-boot environments, why not just disable `CAP_SYS_RAWIO` in such environments, since it is the possession of that capability that permits compromising a booted kernel? 

That led Matthew [to point out](<https://lwn.net/Articles/542516/>) a major problem with `CAP_SYS_RAWIO`: 

CAP_SYS_RAWIO seems to have ended up being a catchall of "Maybe someone who isn't entirely root should be able to do this", and not everything it covers is equivalent to being able to compromise the running kernel. I wouldn't argue with the idea that maybe we should just reappraise most of the current uses of CAP_SYS_RAWIO, but removing capability checks from places that currently have them seems like an invitation for userspace breakage. 

To see what Matthew is talking about, we need to look at a little history. Back in January 1999, when capabilities first appeared with the release of Linux 2.2, `CAP_SYS_RAWIO` was a single-purpose capability. It was used in just a single C file in the kernel source, where it governed access to two system calls: `iopl()` and `ioperm()`. Those system calls permit access to I/O ports, allowing uncontrolled access to devices (and providing various ways to modify the state of the running kernel); hence the requirement for a capability in order to employ the calls. 

The problem was that `CAP_SYS_RAWIO` rapidly grew to cover a range of other uses. By the time of Linux 2.4.0, there were 37 uses across 24 of the kernel's C source files, and [looking at the 3.9-rc2 kernel](<http://blog.man7.org/2013/03/revisiting-kernel-capability-usage.html>), there are 69 uses in 43 source files. By either measure, `CAP_SYS_RAWIO` is now the third most commonly used capability inside the kernel source (after `CAP_SYS_ADMIN` and `CAP_NET_ADMIN`). 

`CAP_SYS_RAWIO` seems to have encountered [a fate similar to `CAP_SYS_ADMIN`](<https://lwn.net/Articles/486306/>), albeit on a smaller scale. It has expanded well beyond its original narrow use. In particular, Matthew [noted](<https://lwn.net/Articles/542570/>): 

Not having CAP_SYS_RAWIO blocks various SCSI commands, for instance. These might result in the ability to write individual blocks or destroy the device firmware, but do any of them permit modifying the running kernel? 

Peter had [some choice words](<https://lwn.net/Articles/542571/>) to describe the abuse of `CAP_SYS_RAWIO` to protect operations on SCSI devices. The problem, of course, is that in order to perform relatively harmless SCSI operations, an application requires the same capability that can trivially be used to damage the integrity of a secure-boot system. And that, as Matthew [went on to point out](<https://lwn.net/Articles/542574/>), is the point of `CAP_COMPROMISE_KERNEL`: to disable the truly dangerous operations (such as MSR writes) that `CAP_SYS_RAWIO` permits, while still allowing the less dangerous operations (such as the SCSI device operations). 

All of this leads to a conundrum that was [nicely summarized](<https://lwn.net/Articles/542575/>) by Matthew. On the one hand, `CAP_COMPROMISE_KERNEL` is needed to address the problem that `CAP_SYS_RAWIO` has become too diffuse in its meaning. On the other hand, the addition of `CAP_COMPROMISE_KERNEL` checks in places where there were previously no capability checks in the kernel means that applications that drop all capabilities will break. There is no easy way out of this difficulty. As Peter [noted](<https://lwn.net/Articles/542576/>): ""We thus have a bunch of unpalatable choices, **all of which are wrong**"". 

Some possible resolutions of the conundrum were [mentioned](<https://lwn.net/Articles/542464/>) by Josh Boyer earlier in the thread: `CAP_COMPROMISE_KERNEL` could be treated as a "hidden" capability whose state could be modified only internally by the kernel. Alternatively, `CAP_COMPROMISE_KERNEL` might be specially treated, so that it can be dropped only by a `capset()` call that operates on that capability alone; in other words, if a `capset()` call specified dropping multiple capabilities, including `CAP_COMPROMISE_KERNEL`, the state of the other capabilities would be changed, but not the state of `CAP_COMPROMISE_KERNEL`. The problem with these approaches is that they special-case the treatment of `CAP_COMPROMISE_KERNEL` in a surprising way (and surprises in security-related APIs have a way of coming back to bite in the future). Furthermore, it may well be the case that analogous problems are encountered in the future with other capabilities; handling each of these as a special case would further add to the complexity of the capabilities API. 

The discussion in the thread touched on a number of other difficulties with capabilities. Part of the solution to the problem of the overly broad effect of `CAP_SYS_RAWIO` (and `CAP_SYS_ADMIN`) might be to split the capability into smaller pieces--replace one capability with several new capabilities that each govern a subset of the operations governed by the old capability. Each privileged operation in the kernel would then check to see whether the caller had either the old or the new privilege. This would allow old binaries to continue to work while allowing new binaries to employ the new, tighter capability. The risk with this approach is, as Casey Schaufler [noted](<https://lwn.net/Articles/542586/>), the possibility of an explosion in the number of capabilities, which would further complicate administering capabilities for applications. Furthermore, splitting capabilities in this manner doesn't solve the particular problem that the `CAP_COMPROMISE_KERNEL` patches attempt to solve for `CAP_SYS_RAWIO`. 

Another general problem [touched on](<https://lwn.net/Articles/542587/>) by Casey is that capabilities still have not seen wide adoption as a replacement for set-user-ID and set-group-ID programs. But, as Peter [noted](<https://lwn.net/Articles/542588/>), that may well be 

in large part because a bunch of the capabilities are so close to equivalent to "superuser" that the distinction is meaningless... so why go through the hassle? 

With 502 uses in the 3.9-rc2 kernel, `CAP_SYS_ADMIN` is the most egregious example of this problem. That problem itself would appear to spring from the Linux kernel development model: the decisions about which capabilities should govern new kernel features typically are made by individual developer in a largely decentralized and uncoordinated manner. Without having a coordinated big picture, many developers have adopted the seemingly safe choice, `CAP_SYS_ADMIN`. A related problem is that [it turns out](<http://forums.grsecurity.net/viewtopic.php?f=7&t=2522>) that a number of capabilities allow escalation to full root privileges in certain circumstances. To some degree, this is probably unavoidable, and it doesn't diminish the fact that a well-designed capabilities scheme can be used to reduce the attack surface of applications. 

One approach that might help solve the problem of overly broad capabilities is hierarchical capabilities. The idea, [mentioned](<https://lwn.net/Articles/542594/>) by Peter, is to split some capabilities in a fashion similar to the way that the root privilege was split into capabilities. Thus, for instance, `CAP_SYS_RAWIO` could become a hierarchical capability with sub-capabilities called (say) `CAP_DANGEROUS` and `CAP_MOSTLY_HARMLESS`. A process that gained or lost `CAP_SYS_RAWIO` would implicitly gain or lose both `CAP_DANGEROUS` and `CAP_MOSTLY_HARMLESS`, in the same way that transitions to and from an effective user ID of 0 grant and drop all capabilities. In addition, sub-capabilities could be raised and dropped independently of their "siblings" at the same hierarchical level. However, sub-capabilities are not a concept that currently exists in the kernel, and it's not clear whether the existing capabilities API could be tweaked in such a way that they could be implemented sanely. Digging deeper into that topic remains an open challenge. 

The `CAP_SYS_RAWIO` discussion touched on a long list of difficulties in the current Linux capabilities implementation: capabilities whose range is too broad, the difficulties of splitting capabilities while maintaining binary compatibility (and, conversely, the administrative difficulties associated with defining too large a set of capabilities), the as-yet poor adoption of binaries with file capabilities vis-a-vis traditional set-user-ID binaries, and the (possible) need for an API for hierarchical capabilities. It would seem that capabilities still have a way to go before they can deliver on the promise of providing a manageable mechanism for providing discrete, non-elevatable privileges to applications.
