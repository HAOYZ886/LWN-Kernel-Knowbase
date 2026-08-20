---
title: "GPL-only symbols and ndiswrapper"
url: https://lwn.net/Articles/205644/
date: "October 25, 2006"
category: Copyright issues; NDISwrapper
author: 
---

The "ndiswrapper" module has been featured on this page before. It is a special sort of glue module which allows Windows NDIS drivers to be loaded into a Linux kernel. It can be found on systems using hardware (wireless adapters in particular) which is not well supported by Linux drivers; by gluing in the Windows driver, ndiswrapper allows this hardware to operate. But, since it is a mechanism created to stuff the most proprietary of binary modules into Linux, ndiswrapper was always going to raise some eyebrows. 

One of the many changes that went into the 2.6.16 kernel was an explicit check for the ndiswrapper module. It is, in fact, this explicit: 

```
if (strcmp(mod->name, "ndiswrapper") == 0)
    	add_taint_module(mod, TAINT_PROPRIETARY_MODULE);
```

This test means that any system which has had ndiswrapper loaded will have the "proprietary module" taint flag set. As a result, the kernel developers are highly unlikely to be interested in helping with any problems encountered running that kernel. Since 2.6.16 was released last March, one might well wonder why ndiswrapper author Giridhar Pemmasani is only now getting around to [complaining](<https://lwn.net/Articles/205645/>) about that test. It turns out that the kernel developers have quietly taken things one step further in the 2.6.19-rc kernels. 

The kernel has long exported symbols to modules in two modes. Symbols exported with `EXPORT_SYMBOL` are available to all modules loaded into the kernel, while those exported with `EXPORT_SYMBOL_GPL` are only available to those which declare a GPL-compatible license. This distinction has never been a problem for ndiswrapper, which is licensed under the GPL. So, even after the explicit taint was added, ndiswrapper could load and function normally. 

For 2.6.19, a patch by Florin Malita was merged which changes the calculation for GPL-only symbols slightly. Rather than checking whether a module has a GPL-compatible license, the new code checks whether the module has the "proprietary module" taint bit set. In most cases, the end result is the same. For ndiswrapper, however, the result is that GPL-only symbols, which were accessible in earlier kernels, are now unavailable. And that means that ndiswrapper can no longer be loaded into the kernel. The module's author thinks that this is unfair, since ndiswrapper is, in fact, GPL-licensed code. 

Alan Cox's [response](<https://lwn.net/Articles/205647/>) reads like this: 

EXPORT_SYMBOL_GPL() is used to assert that the symbol is absolutely definitely not a public symbol. EXPORT_SYMBOL exports symbols which might be but even then the GPL derivative work rules apply. When you mark a driver GPL it is permitted to use _GPL symbols, but if it does so it cannot then go and load other non GPL [modules] and expect people not to question its validity. 

The core idea makes sense: the GPL-only restrictions are not worth much if they can be trivially circumvented by loading a glue module. One cannot help but wonder, however, if the wrong target has been chosen in this case. 

The purpose of GPL-only exports is to inhibit the creation of proprietary derived products of the kernel. It is hard to imagine an argument that could demonstrate that a typical NDIS module is, in any way, a derived product of the Linux kernel. These are drivers written for another operating system entirely by people who, likely as not, have never had any sort of contact with Linux source. Unlike certain other types of proprietary modules, NDIS drivers are clearly independent works. One may well balk at the notion of loading such a driver into one's kernel, but it is hard to make a case that copyright law somehow prohibits such an action. 

It also seems a little strange to penalize a module for having the wrong name. There are no explicit checks for, say, the MadWifi module, which also loads a binary-only component. Simply renaming the module would circumvent this check, opening a window which would take the kernel developers some months to close again. One could imagine a determined programmer coming up with a random name every time a module is built, decisively winning that particular battle. The ndiswrapper author seems uninclined to play those games, however; he has, instead, tried to work within the kernel community. The module already takes pains to add a kernel taint itself whenever an NDIS driver is loaded. 

There does not seem to be any particular interest in the kernel community in backing down on this change however. That leaves the ndiswrapper author in a position where he must either rework the code to avoid GPL-only symbols or find some other way of enabling it to load once again. One assumes that some sort of workaround will be found; it may not be an optimal solution, but ndiswrapper does have a significant community which depends on it to make its hardware work under Linux.
