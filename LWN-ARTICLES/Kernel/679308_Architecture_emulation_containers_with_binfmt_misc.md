---
title: Architecture emulation containers with binfmt_misc
url: https://lwn.net/Articles/679308/
date: "March 9, 2016"
category: Containers; binfmt misc
author: "By Jonathan Corbet March 9, 2016"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
March 9, 2016

Containers bring a lot of advantages in the areas of security and systems administration; they can be used to run lightweight virtual "systems" in isolation from each other and the host system. Since containers lack their own kernel — they run directly on the host-system kernel — one does not ordinarily expect them to be built for an architecture other than the host they are running on. But it turns out that there are use cases for such containers, and that they can be run using the somewhat obscure "`binfmt_misc`" kernel mechanism if a small patch set is applied. 

#### binfmt_misc

When an executable file is provided to one of the `exec*()` system calls, the kernel normally expects to find a native binary for the system it is running on. The kernel has long had a mechanism by which it can recognize other executable-file formats and run them, though. The classic example is the module that looks for a file that begins with "`#!`" — the marker for a shell script. When such a file is recognized, the name of the interpreter for the script will be read from the first line of the file; the interpreter will then be run with the file as its standard input. 

But one can imagine many other possible formats for executable files. These could be binaries built for a different operating system (DOS binaries that could be run with DOSEMU, for example) or byte-code binaries that need to run on a specific machine (such as Java byte code). One could try to code awareness of all these formats into the kernel, but that gets unwieldy after a while. It also lacks flexibility, which is unfortunate; the kernel developers are never going to know about all of the possible executable formats that might be of interest. 

The obvious solution is to allow user space to describe new executable formats to the kernel; that is the role of the `binfmt_misc` mechanism. If this feature is configured into the kernel (as it usually is), a system administrator can add a new executable format by writing a special string to `/proc/sys/fs/binfmt_misc/register`. That string includes: 

  * A way for the kernel to recognize the new format. It can either be a particular file extension, or a "magic number" found near the beginning of the file. 

  * The name of the interpreter that is to be run to execute files with this format. 

  * Some flags that control how the `argv` array is created and, essentially, whether files in this format can be setuid or not. 

The full details of how it all works can be found in [`Documentation/binfmt_misc.txt`](<https://lwn.net/Articles/679310/>) in the kernel source tree. 

It is not hard to see how `binfmt_misc` can be used to run binaries built for a different architecture. It is a simple matter of describing those binaries and naming an emulator ([QEMU](<http://qemu.org/>), for example) that is able to run the binaries. That works well for binaries to be run directly on the host system, but it can be a bit more challenging to run a container that is built for another architecture. 

#### Architecture emulation in containers

The problem, as James Bottomley pointed out in [this brief patch set](<https://lwn.net/Articles/679309/>), is that `binfmt_misc` has to locate and run the interpreter binary at the time that the foreign binary is invoked. This invocation happens within the container, so the interpreter has to be visible in the container as well, but, likely as not, the container is running within a namespace intended to keep it from seeing the rest of the system. As a result, the interpreter must be placed inside the container itself. That complicates what would otherwise be containerized system built entirely for the emulated architecture. It also forces any orchestration system to be aware of the emulation within the container and set things up accordingly, making emulated containers less transparent than they would otherwise be. 

The solution is to add a new mode for `binfmt_misc` wherein the interpreter binary is opened by the kernel when the new format is initially set up. When a binary in that format is encountered, the already-opened interpreter can be run, rather than seeking out and opening the interpreter at that time. This mechanism will work inside a container that otherwise has no access to the interpreter; the kernel already has the interpreter open, so it can run it directly. 

This mode is set up by using the new "`F`" flag when describing the format to `binfmt_misc`. Once the kernel has opened the interpreter file, it will keep it open until the format is removed. That means that updates to the interpreter binary will not take effect unless the format is removed and reestablished. That should not ordinarily be a problem, but it could be a surprise for system administrators who are not aware of this behavior. 

The patch set received a small number of generally favorable reviews. If it is merged, as seems likely, it will make it easier to run containers built for a number of machine architectures on the same host, making Linux containers more flexible in general.
