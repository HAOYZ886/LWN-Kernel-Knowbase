---
title: Locking down module parameters
url: https://lwn.net/Articles/708179/
date: "December 7, 2016"
category: "Security-UEFI secure boot"
author: "By Jake Edge December 7, 2016"
---

By **Jake Edge**  
December 7, 2016

Support for UEFI secure boot has been available in most mainstream Linux distributions for several years now, but there are still some wrinkles to work out. Some distributors are concerned about various ways for the root user to alter the kernel in ways that would allow the secure-boot assurances to be circumvented. For years, there have been efforts to "lock down" the kernel so that the known ways of evading secure boot can be disabled by distributions and others. The latest piece of that puzzle is a proposal to annotate kernel module parameters such that some can be disallowed when a secure-boot kernel is running. 

Back in November, David Howells [posted](<https://lwn.net/Articles/706637/>) the latest version of the "kernel lockdown" patches, which he has picked up and expanded from those [pushed by Matthew Garrett](<https://lwn.net/Articles/514985/>) back in 2012 and 2013. The patch set restricted a lot of different functionality that would allow user space to modify the running kernel image (which, in turn, allows user space to circumvent secure boot). The restrictions disallow things like loading unsigned kernel modules, writing to `/dev/mem` or `/dev/kmem`, using `kexec_load()` to run a different kernel, various ways to directly access the hardware, and so on. 

Previous iterations of the patch set have run aground at least partly due to the `kexec_load()` restrictions, because some kernel developers did not want to see the useful kexec facility completely disabled. An alternative system call (`kexec_file_load()`) was added that will [only allow booting signed kernels](<https://lwn.net/Articles/603116/>), which neatly solved things for both sides. This time, the main objection came from Alan Cox, who [thought](<https://lwn.net/Articles/708256/>) there was a fundamental piece missing: 

You need to filter or lock down kernel module options because a lot of modules let you set the I/O port or similar (eg mmio) which means you can hack the entire machine with say the 8250 driver just by using it with an mmio of the right location to patch the secure state to zero just by getting the ability to write to the modules conf file. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Without that at least fixed I don't see the point in merging this. Either we don't do it (which given the level of security the current Linux kernel provides, and also all the golden key messups from elsewhere might be the honest approach), or at least try and do the job right. 

Essentially, Cox is arguing that changing certain kernel module parameters before loading a (signed) module is yet another avenue for modifying the kernel image. His objection led Howells to quickly post an [RFC patch of sorts](<https://lwn.net/Articles/708271/>) that would restrict certain operations in drivers when `kernel_is_locked_down()` is true. While that was on the right track, Cox [said](<https://lwn.net/Articles/708272/>), he would rather see a whitelist-based approach, rather than the blacklist-based one that Howells proposed. 

All of that led to the [module parameter annotation patch set](<https://lwn.net/Articles/707851/>) that Howells posted on December 1\. The idea is that all module parameters will be annotated to describe what kind of hardware resource they control (if any). That information can be used in a subsequent patch to restrict which can be used in a locked-down kernel. 

The change was made by altering the `module_param*()` macros, which are helpers for modules that need to take parameters at load time. An argument was added for the "hardware type" and the macros were renamed to `module_param_hw*()`. As can be seen in the [first patch](<https://lwn.net/Articles/708274/>) of the series, the types are I/O port, I/O memory address, interrupt number, DMA channel, DMA address, or other. The change made in the [second patch](<https://lwn.net/Articles/708371/>) demonstrates the idea: 

```
-module_param(mmio_address, ulong, 0);
        +module_param_hw(mmio_address, ulong, iomem, 0);
```

The other 37 patches in the series annotate various module parameters throughout the tree (mostly, of course, in `drivers/`). But Greg Kroah-Hartman was [not particularly impressed](<https://lwn.net/Articles/708276/>) (""ick ick ick"") with the idea. He suggested that the secure boot patch set (i.e. kernel lockdown) was not going anywhere, so there was no need for the annotations. Furthermore, he was skeptical that stopping root users from setting these module parameters was really going to help stop secure-boot abuses. 

Garrett [noted](<https://lwn.net/Articles/708321/>) that the patch set was currently being carried by ""basically every single mainstream Linux distribution"", however. This costs time and effort by the distributions to rebase the patches on newer kernels. Beyond that, at least some of the module parameters can be used to route around secure boot: 

If root can tell a driver to probe for hardware at a specific address, and that driver will then blindly do so, root is trivially able to modify arbitrary kernel memory and disable arbitrary security features. IRQ or io port attacks are much more difficult to take advantage of, but I could imagine that some of them are still plausible. 

Given that distributions ship the lockdown patches, and that Cox has said that some way to disable module parameters should be part of that, Garrett argued that the annotations should be merged unless there were technical objections to the implementation. Kroah-Hartman was [not buying that argument](<https://lwn.net/Articles/708324/>), though. Distributions are _not_ shipping the annotations and the annotations patches don't actually disable anything, they just make it possible to do so, he said. He also suggested simply marking all module parameters (or any that touch the hardware) as "bad", rather than trying to pick and choose which were usable for a locked-down kernel. 

But the annotations have other uses, Cox [said](<https://lwn.net/Articles/708325/>). Locking down raw I/O access, even for systems that are not running under the secure-boot restrictions, is valuable. Right now, certain unrelated capabilities can be used to effectively get the `CAP_SYS_RAWIO` capability by loading modules with crafted parameters, but the kernel could eliminate that possibility by using the annotations. 

Howells also [wondered](<https://lwn.net/Articles/708327/>) about the seemingly contradictory ordering requirements from Kroah-Hartman and Cox: ""for Alan, I have to fix the module parameter hole first; for you, I have to do the secure boot support first"". Like Cox, though, Howells thinks the annotations have value in their own right: ""However, annotation [of] module parameters to indicate hardware resource configuration seems potentially useful in its own right - and lets the policy be decided later."" 

There does seem to be something of a rock and a hard place problem here. The kernel lockdown patches are not particularly popular with quite a few kernel developers (including Linus Torvalds, which makes things that much harder), but they are shipping. That would seem to indicate that they belong upstream or, at least, that _something_ implementing that functionality belongs upstream. 

The annotations, themselves, are relatively harmless (other than providing a bit of churn) and will allow Cox's module parameter issue to be addressed. That will lead to a more secure kernel, overall, with or without secure boot. Once the relevant maintainers have reviewed the patches (and those reviews are starting to trickle in), it would seem that the patches should be merged (though Torvalds would need to override Kroah-Hartman's NAK). The 4.10 merge window will be opening soon; it's likely too late for the annotation patches to make that cut but, with luck, they could make it for 4.11. The larger (and more invasive in the eyes of some) lockdown patch set would seem to have surmounted the known technical objections at that point; whether that paves the way for those to be merged remains to be seen.
