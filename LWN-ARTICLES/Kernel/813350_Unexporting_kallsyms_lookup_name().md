---
title: Unexporting kallsyms_lookup_name()
url: https://lwn.net/Articles/813350/
date: "February 28, 2020"
category: "Modules-Exported symbols"
author: "By Jonathan Corbet February 28, 2020"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
February 28, 2020

One of the basic rules of kernel-module development is that modules can only access symbols (functions and data structures) that have been explicitly exported. Even then, many symbols are restricted so that only modules with a GPL-compatible license can access them. It turns out, though, that there is a readily available workaround that makes it easy for a module to access any symbol it wants. That workaround seems likely to be removed soon despite some possible inconvenience for some out-of-tree users; the reason why that is happening turns out to be relatively interesting. 

The backdoor in question is [`kallsyms_lookup_name()`](<https://elixir.bootlin.com/linux/latest/source/kernel/kallsyms.c#L162>), which will return the address associated with any symbol in the kernel's symbol table. Modular code that wants to access a symbol ordinarily denied to it can use `kallsyms_lookup_name()` to get the address of its target, then dereference it in the usual ways. This function itself is exported with the GPL-only restriction, which theoretically limits its use to free software. But if a proprietary module somewhere were to falsely claim a free license to get at GPL-only symbols, it would not be the first time. 

Will Deacon has [posted a patch series](<https://lwn.net/ml/linux-kernel/20200221114404.14641-1-will@kernel.org/>) that removes the export for `kallsyms_lookup_name()` (and `kallsyms_on_each_symbol()`, which is also open to abuse). There were some immediate positive responses; few developers are favorably inclined toward module authors trying to get around the export system, after all. There were, however, a couple of concerns expressed. 

One of those is that there is, it seems, a class of out-of-tree users of `kallsyms_lookup_name()` that is generally considered to be legitimate: live-patching systems for the kernel. Irritatingly, kernel bugs often stubbornly refuse to restrict themselves to exported functions, so a live patch must be able to locate (and patch out) any function in the kernel; `kallsyms_lookup_name()` is a convenient way to do that. After some discussion Joe Lawrence [let it be known](<https://lwn.net/ml/linux-kernel/943e7093-2862-53c6-b7f4-96c7d65789b9@redhat.com/>) that the [kpatch](<https://en.wikipedia.org/wiki/Kpatch>) system has all of its needed infrastructure in the mainline kernel, and so has no further need for `kallsyms_lookup_name()`. The [Ksplice](<https://ksplice.oracle.com/>) system, though, evidently still uses it. As Miroslav Benes [observed](<https://lwn.net/ml/linux-kernel/alpine.LSU.2.21.2002251854550.1630@pobox.suse.cz/>), though: ""no one cares about ksplice in upstream now"". So it would appear that live patching will not be an obstacle to the merging of this patch. 

A different sort of concern was [raised](<https://lwn.net/ml/linux-kernel/20200221232746.6eb84111a0d385bed71613ff@kernel.org/>) by Masami Hiramatsu, who noted that there are a number of other ways to find the address associated with a kernel symbol. User space could place some kprobes to extract that information, or a kernel module could, if time and CPU use is not a concern, use `snprintf()` with the "`%pF`" format (which prints the function associated with a given address) to search for the address of interest. He worried that the change would make life harder for casual developers while not really getting in the way of anybody who is determined to abuse the module mechanism. 

In response, Deacon posted [an interesting message](<https://lwn.net/ml/linux-kernel/20200221144853.GA18153@willie-the-truck/>) about what is driving this particular change. Kernel developers are happy to make changes just to make life difficult for developers they see as abusing the system, but that is not quite what is happening here. Instead, it is addressing a support issue at Google. 

Back in 2018, LWN [reported on](<https://lwn.net/Articles/771974/>) work being done to bring the Android kernel closer to the mainline. One of the steps in that direction is moving the kernel itself into the [Android generic system image](<https://source.android.com/setup/build/gsi>) (GSI), an Android build that must boot and run on a device for that device to be considered compliant with the Android requirements. Putting the kernel into the GSI means that hardware vendors can no longer modify it; they will be limited to what they can do by adding kernel modules to the GSI. 

Restricting vendors to supplying kernel modules greatly limits the kind of changes they can make; there will be no more Android devices that replace the CPU scheduler with some vendor's special version, for example. But that only holds if modules are restricted to the exported-symbol interface; if they start to reach into arbitrary parts of the kernel, all bets are off. Deacon doesn't say so, but it seems clear that some vendors are, at a minimum, thinking about doing exactly that. The business-friendly explanation for removing this capability is: ""Monitoring and managing the ABI surface is not feasible if it effectively includes all data and functions via kallsyms_lookup_name()"". 

After seeing this explanation, Hiramatsu [agreed that the patch makes sense](<https://lwn.net/ml/linux-kernel/20200222084438.37a0ff99edbe32acdb666c79@kernel.org/>) and offered a Reviewed-by tag. So this concern, too, seems unlikely to prevent this patch set from being merged. 

It's worth repeating that discouraging module developers from bypassing the export mechanism is generally seen as more than sufficient motivation to merge a change like this. But it is also interesting to see a large company supporting that kind of change as well. By more closely tying the Android kernel to the mainline, Google would appear to be better aligning its own interests with the long-term interests of the development community — on this point, at least. That, hopefully, will lead to better kernels on our devices that also happen to be a lot closer to mainline kernels.
