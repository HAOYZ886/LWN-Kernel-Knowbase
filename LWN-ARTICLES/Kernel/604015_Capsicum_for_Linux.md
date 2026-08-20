---
title: Capsicum for Linux
url: https://lwn.net/Articles/604015/
date: "July 2, 2014"
category: "Security-Security technologies"
author: "By Jonathan Corbet July 2, 2014"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
July 2, 2014

Capsicum is a process sandboxing framework that was originally developed for FreeBSD; it was [covered here](<https://lwn.net/Articles/482858/>) in early 2012\. [The beginnings of Capsicum support for Linux](<https://lwn.net/Articles/603929/>) have now been posted by David Drysdale for review; that provides a good opportunity to look at how this mechanism might fit into the Linux kernel. This work might be a hard sell in the kernel community, but Capsicum might just provide a sufficiently useful set of features to make the trouble worthwhile. 

Capsicum is built around a concept called "capabilities," which, naturally enough, is entirely different from Linux capabilities (and from POSIX capabilities as well). In the Capsicum world, capabilities are attached to file descriptors to regulate which operations can be performed on those descriptors. So, for example, a file descriptor can only be read if the `CAP_READ` capability is present. Access to `lseek()` is controlled by `CAP_SEEK`, memory mapping has a set of capabilities (`CAP_MMAP_W` to create a writable mapping, for example), truncation is controlled by `CAP_FTRUNCATE`, and so on. There are two special capabilities for `ioctl()` and `fcntl()` that restrict those calls to specific subcommands. 

By default, open file descriptors are unrestricted. The normal mode of operation is that a process will apply restrictions to itself using the new `cap_rights_limit()` system call: 

```
int cap_rights_limit(unsigned int fd, struct cap_rights *new_rights,
        			 unsigned int new_fcntls, int nioctls, unsigned int *new_ioctls);
```

After this call, operations on `fd` will be limited to those listed in `new_rights`. If those rights include `CAP_FCNTL`, then `new_fcntls` limits the set of `fcntl()` commands available. Similarly, if the capabilities on the file descriptor include `CAP_IOCTL`, the `new_ioctls` array (of length `nioctls`) provides the set of allowed `ioctl()` commands. Multiple calls to `cap_rights_limit()` can be made for the same file descriptor, but those calls can only remove capabilities, never add them. 

There is also a `cap_rights_get()` call to query the set of capabilities attached to a given file descriptor. 

Needless to say, restrictions on file descriptors are of limited value if the constrained process can simply open new descriptors on the same objects. To prevent that from happening, Capsicum implements a "capability mode" entered via `cap_enter()`. Once that mode has been entered, access to most global namespaces is curtailed, preventing the opening of new files. A process _can_ still open a file with `openat()` if it has a directory file descriptor (and, of course, the relevant capabilities are present). Such files are constrained to be underneath that directory, though — use of absolute pathnames or "`../`" is not allowed. 

(As an aside, the "can only open files below this directory" functionality was deemed to be sufficiently useful that David pulled it out into a separate patch and made it available independently of Capsicum. [This patch](<https://lwn.net/Articles/604052/>) adds a new `O_BENEATH_ONLY` flag for calls like `openat()`. Once a directory has been opened with this option, the resulting file descriptor can only be used to open files that exist below that directory in the filesystem hierarchy.) 

That said, the patch set as posted does not provide an implementation of `cap_enter()`. Also missing is the entire "process capabilities" mechanism, which represents specific processes as file descriptors so that the relevant system calls (`wait()`, `kill()`, etc.) can be controlled. The patch set is described as being "part 1," so, one assumes, the remaining pieces will come later. 

Within the kernel, system call implementations typically start by converting passed-in file descriptors to `struct file` with calls to `fdget()`. This is the point where David decided to apply the capability checks. When file descriptors are restricted with Capsicum, the normal `file` structure is replaced by a wrapper structure containing the rights information. Every `fdget()` call in the kernel (there are about 100 of them) must be replaced with a call to: 

```
struct file *fdgetr(unsigned int fd, int caps ...);
```

Where `caps` is a variable-length list of capabilities that must be present for the operation to succeed. Callers must also be changed to deal with an "error pointer" return value; `fdget()` in current kernels can return `NULL` but not a specific error value. The result is that the patch set is somewhat invasive; that may be a cause of resistance should the patch set reach a point where it is being seriously proposed for inclusion. 

The patch set currently works by creating a pair of new Linux security module (LSM) hooks to do the actual capability checks. David wonders, though, whether that is the right approach, since Capsicum is not a complete security module. If the kernel implemented stacked security modules, perhaps Capsicum could be run in this mode alongside another, more complete module. But stacking does not look like it will be supported in the kernel anytime soon. So Capsicum may well be better off implemented outside of the LSM framework. 

There is another question that is worth considering here. The kernel's [secure computing (seccomp)](<https://lwn.net/Articles/475043/>) subsystem allows the loading of programs (written for the BPF virtual machine) that can, in theory, implement all of the restrictions found in Capsicum, especially if [the recently proposed BPF changes](<https://lwn.net/Articles/603983/>) are merged. It might not be _easy_ , but it should be possible. Somebody is bound to ask whether the kernel needs another sandbox-creation mechanism with similar capabilities. 

In general, the addition of new security-related subsystems can be a hard sell; many developers see little value for a lot of cost in these subsystems. But there is value in the ability to reduce the damage that can be done by a compromised process, and FreeBSD's use of Capsicum means that some applications have already had the necessary code added. Adding the same API to Linux would allow that work to be reused. So Capsicum seems worth considering, even if it will likely have some obstacles to overcome before merging is a possibility.
