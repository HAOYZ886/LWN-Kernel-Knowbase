---
title: Automatically detecting kernel interface changes
url: https://lwn.net/Articles/703890/
date: "October 19, 2016"
category: "Development tools-ABI-related"
author: "By Jonathan Corbet October 19, 2016 Kernel
Recipes"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
October 19, 2016

* * *

[Kernel Recipes](<https://lwn.net/Archives/ConferenceByYear/#2016-Kernel_Recipes>)

ABI changes can be painful for anybody charged with the development and maintenance of software; that can be doubly so when the changes happen inadvertently and take people by surprise. There is tooling out there that can search for and report ABI changes. At [Kernel Recipes 2016](<https://kernel-recipes.org/en/2016/>), Dodji Seketeli presented some early work he has done on a tool that would find unexpected kernel ABI changes and asked what might seem like an obvious question: would this functionality be useful to the development community? 

It is worth noting that he was not talking about the sort of ABI change that kernel developers worry about the most: changes to the user-space ABI. Instead, he is focusing on changes to the loadable-module ABI. At first blush, that might seem like it could reduce the level of interest in his work. As was pointed out in the talk, kernel developers are generally unwilling to talk about the module interface as an ABI at all; at best, it's a fluid API with no stability guarantees. This interface is explicitly allowed to change, so the number of developers wanting a tool to flag those changes might be thought to be small. 

Interest in this kind of tool comes mostly from distributors. The enterprise distributors could use it to let binary-driver vendors know when [![\[Dodji Seketeli\]](https://static.lwn.net/images/conf/2016/recipes/DodjiSeketeli-sm.jpg)](<https://lwn.net/Articles/703897/>) something has changed in the module interface. But Ben Hutchings, Debian's kernel maintainer, said that it would be generally useful to avoid making changes to the module interface when patching a stable kernel. 

The [abidiff](<https://sourceware.org/libabigail/manual/abidiff.html>) tool exists to provide just this kind of information. It reads the ELF symbol information from an object file, along with any debug information found there. It uses that information to build an internal representation of the ABI, which can be saved in a special XML file. Given ABI representations from two different objects, abidiff can report on the differences between the two. 

Seketeli showed some example output from abidiff; it can be seen in [his slides](<https://kernel-recipes.org/en/2016/talks/would-an-abi-changes-visualization-tool-be-useful-to-linux-kernel-maintenance/>). The tool is able to detect changes in the types of a function's parameters or its return value. Anything that changes the size of a structure or the layout of its members will also be reported on. The removal of functions is noted, and so on. There are mechanisms for reducing noise by filtering out changes that might not be of interest; for example, changes to structures that do not appear in a specific set of header files can be suppressed. Other tools built on top of abidiff can look for ABI changes in libraries stored in package files. 

But, he pointed out, none of this works with the kernel now. But wouldn't it be nice if we had a tool that could look at a set of kernel modules and exported interfaces, generating a report of what has changed from a previous version? 

Getting there requires a bit of work. The tool would need to understand and handle the special ELF sections used in the kernel build; the `__export_symbol` and `__export_symbol_gpl` sections are particularly relevant. Kernel modules also need to be parsed properly, and an interface description generated from the result. The sheer size of the kernel presents a problem as well; it will force some memory-usage optimizations that have not been necessary thus far. These are the sort of issues he has been working on. 

Thus far, he has added a kernel-specific mode to the [abidw](<https://www.mankier.com/1/abidw>) tool, which generates an XML representation of an ABI from an ELF file for use with abidiff. Some examples of the output can be found in [this page](<https://people.redhat.com/~dseketel/kabidiff/>). Anybody wanting to play with this work can grab a copy of the repository by running: 

```
git clone -b dodji/kabidiff git://sourceware.org/git/libabigail.git
```

The discussion of this work was wide-ranging and energetic; it is hard to report on here. One topic that came up was the possibility of detecting changes in the user-space ABI instead; that is a tool that would be useful for regression testing in general. That, Seketeli allowed, is a rather harder problem. Even just looking at the system-call interface, it can be hard for a tool to understand what a system call's parameters are supposed to represent. 

So a user-space ABI checker is probably not on the immediate horizon. We probably will see a tool that can find changes in the module interface, though, and that will have its own uses. Developers might be surprised to learn how often the changes they make affect the interface used by loadable modules.
