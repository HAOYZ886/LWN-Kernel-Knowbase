---
title: Kconfiglib
url: https://lwn.net/Articles/426013/
date: "February 2, 2011"
category: "Build system-Kernel configuration"
author: "By Jonathan Corbet February 2, 2011"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
February 2, 2011

The kernel configuration system is a complex bit of code in its own right; many people who have no trouble hacking on kernel code find reasons to avoid going into the configuration subsystem. There is value in being able to work with the complicated data structure that is a kernel configuration, though. Ulf Magnusson has recently posted a library, [Kconfiglib](<https://lwn.net/Articles/426006/>), which, he hopes, will make that easier. 

Kconfiglib is a Python library which is able to load, analyze, and output kernel configurations; care has been taken to ensure that any configuration it creates is identical to what comes out of the existing kernel configuration system. With Kconfiglib, it becomes straightforward to write simple tools like "allnoconfig"; it also is possible to ask questions about a given configuration. One possible tool, for example, would answer the "why can't I select `CONFIG_FOO`" question - a useful feature indeed. 

There are currently no Python dependencies in the kernel build system; trying to add one could well run into opposition. But Kconfiglib could find a role in the creation of ancillary tools which are not required to configure and build a kernel as it's always been done. For the curious, there's [a set of examples](<http://dl.dropbox.com/u/10406197/kconfiglib.html>) available.
