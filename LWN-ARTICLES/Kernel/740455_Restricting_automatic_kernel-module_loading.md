---
title: "Restricting automatic kernel-module loading"
url: https://lwn.net/Articles/740455/
date: "December 4, 2017"
category: "Modules; Security-Kernel hardening"
author: "By Jonathan Corbet December 4, 2017"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
December 4, 2017

The kernel's module mechanism allows the building of a kernel with a wide range of hardware and software support without requiring that all of that code actually be loaded into any given running system. The availability of all of those modules in a typical distributor kernel means that a lot of features are available — but also, potentially, a lot of exploitable bugs. There have been numerous cases where the kernel's automatic module loader has been used to bring buggy code into a running system. An attempt to reduce the kernel's exposure to buggy modules shows how difficult some kinds of hardening work can be. 

#### Module autoloading

There are two ways in which a module can be loaded into the kernel without explicit action on the administrator's part. On most contemporary systems, it happens when hardware is discovered, either by a bus driver (on buses that support discovery) or from an external description like a device tree. Discovery causes an event to be sent to user space, where a daemon like `udev` applies whatever policies have been configured and loads the appropriate modules. This mechanism is driven by the available hardware and is relatively hard for an attacker to influence. 

Within the kernel, though, lurks an older mechanism, in the form of the `request_module()` function. When a kernel function determines that a needed module is missing, it can call `request_module()` to send a request to user space to load the module in question. For example, if an application opens a char device with a given major and minor number and no driver exists for those numbers, [the char device code](<http://elixir.free-electrons.com/linux/latest/source/fs/char_dev.c#L659>) will attempt to locate a driver by calling: 

```
request_module("char-major-%d-%d", MAJOR(dev), MINOR(dev));
```

If a driver module has declared an alias with matching numbers, it will be automatically loaded into the kernel to handle the open request. 

There are hundreds of `request_module()` calls in the kernel. Some are quite specific; one will load the `ide-tape` module should the user be unfortunate enough to have such a device. Others are more general; there are many calls in the networking subsystem, for example, to locate modules implementing specific network protocols or packet-filtering mechanisms. While the device-specific calls have been mostly supplanted by the `udev` mechanism, modules for features like network protocols still rely on `request_module()` for user-transparent automatic loading. 

Autoloading makes for convenient system administration, but it can also make for convenient system exploitation. The [DCCP protocol vulnerability](<https://lwn.net/Articles/715811/>) disclosed in February, for example, is not exploitable if the DCCP module is not loaded in the kernel — which is normally the case, since DCCP has few users. But the autoloading mechanism allows any user to force that module to be loaded simply by creating a DCCP socket. Autoloading thus widens the kernel's attack surface to include anything in a module that unprivileged users can cause to be loaded — and there are a _lot_ of modules in a typical distributor kernel. 

#### Tightening the system

Djalal Harouni has been working on a patch set aimed at reducing the exposure from autoloading; the [most recent version](<https://lwn.net/Articles/740059/>) was posted on November 27\. Harouni's work takes inspiration from the hardening found in the grsecurity patch set, but takes no code from there. In this incarnation (it has changed somewhat over time), it adds a new sysctl knob (`/proc/sys/kernel/modules_autoload_mode`) that can be used to restrict the kernel's autoloading mechanism. If this knob is set to zero (the default), autoloading works as it does in current kernels. Setting it to one restricts autoloading to processes with specific capabilities: processes with `CAP_SYS_MODULE` can cause any module to be loaded, while those with `CAP_NET_ADMIN` can autoload any module whose alias starts with `netdev-`. Setting this knob to two disables autoloading entirely. Once this value has been raised above zero, it cannot be lowered during the lifetime of the system. 

The patch set also implements a per-process flag that could be set with the [`prctl()`](<http://man7.org/linux/man-pages/man2/prctl.2.html>) system call. This flag (which takes the same values as the global flag) could restrict autoloading for a specific process and all of its descendants without changing module-loading behavior in the system overall. 

It is safe to say that this patch set will not be merged in its current form for a simple reason: Linus Torvalds strongly disliked it. Disabling autoloading is likely to break a lot of systems, meaning that distributors will be unwilling to enable this option and it will not see much use. ""A security option that people can't use without breaking their system is pointless"", he [said](<https://lwn.net/Articles/740458/>). The discussion got heated at times, but Torvalds is not opposed to the idea of reducing the kernel's exposure to autoloaded vulnerabilities. It was just a matter of finding the right solution. 

The per-process flag looks like it could be a part of that solution. It could be used, for example, to restrict autoloading for code running within a container while leaving the system as a whole unchanged. It is not uncommon to create a process within a container with the `CAP_NET_ADMIN` capability to configure that container's networking while wanting most of the code running in the container to be unable to force module loading. 

But, Torvalds said, a single flag will never be able to properly control all of the situations where autoloading comes into play. Some modules should perhaps always be loadable, while others may need a specific capability. So he suggested retaining the `request_module_cap()` function added by Harouni's patch set (which performs the load only if a specific capability is present) and using it more widely. But he did have a couple of changes to request. 

The first is that `request_module_cap()` shouldn't actually block module loading if the needed capability is absent — at least not initially. Instead, it should log a message. That will allow a study of where module autoloading is actually needed that would, with luck, point out the places where autoloading could be restricted without breaking existing systems. He also [suggested](<https://lwn.net/Articles/740461/>) that the capability check is too simplistic. For example, the "`char-major-`" autoload described above only happens if a process is able to open a device node with the given major and minor numbers. In such cases, a permission test (the ability to open that special file) has already been passed and the module should load unconditionally. So there may need to be other variants of `request_module()` to describe settings where capabilities do not apply. 

Finally, Torvalds had [another idea](<https://lwn.net/Articles/740463/>) related to the idea that the worst bugs tend to lurk in modules that are poorly maintained at best. The DCCP module mentioned above, for example, is known to be little used and nearly unmaintained. If the modules that _are_ well maintained were marked with a special flag, it might be possible to restrict unprivileged autoloading to those modules only. That would prevent the autoloading of some of the cruftier modules while not breaking autoloading in general. This idea does raise one question that nobody asked, though: when a module ceases being maintained, who will maintain it well enough to remove the "well maintained" flag? 

In any case, that flag will probably not be added right away, if [this proposed plan](<https://lwn.net/Articles/740464/>) from Kees Cook holds. He suggested starting with the `request_module_cap()` approach with warnings enabled. The per-process flag would be added for those who can use it, but the global knob to restrict autoloading would not. Eventually it might be possible to get rid of unprivileged module loading, but that will be a goal for the future. The short-term benefit would be better information about how autoloading is actually used and the per-process option for administrators who want to tighten things down now. 

This conversation highlights one of the fundamental tensions that can be found around kernel hardening work. Few people are opposed to a more secure kernel, but things get much more difficult as soon as the hardening work can break existing systems — and that is often the case. Security-oriented developers often get frustrated with the kernel community's resistance to hardening changes with user-visible impacts, while kernel developers have little sympathy for changes that will lead to bug reports and unhappy users. Some of those frustrations surfaced in this discussion, but most of the developers involved were mostly interested in converging on a solution that works for everybody involved.
