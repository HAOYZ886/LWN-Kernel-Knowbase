---
title: Loading modules from file descriptors
url: https://lwn.net/Articles/519010/
date: "October 10, 2012"
category: Modules
author: "By Michael Kerrisk October 10, 2012"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Michael Kerrisk**  
October 10, 2012

Loadable kernel modules provide a mechanism to dynamically modify the functionality of a running system, by allowing code to be loaded and unloaded from the kernel. Loading code into the kernel via a module has a number of advantages over building a completely new monolithic kernel from modified source code. The first of these is that loading a kernel module does not require a system reboot. This means that new kernel functionally can be added without disturbing users and applications. 

From a developer perspective, implementing new kernel functionality via modules is faster: a slow "compile kernel, reboot, test" sequence in each development iteration is instead replaced by a much faster "compile module, load module, test" sequence. Employing modules can also save memory, since code in a module can be loaded into memory only when it is actually needed. Device drivers are often implemented as loadable modules for this reason. 

From a security perspective, loadable modules also have a potential downside: since a module has full access to kernel memory, it can compromise the integrity of a system. Although modules can be loaded only by privileged users, there are still potential security risks, since a system administrator may be unable to directly verify the authenticity and origin of a particular kernel module. Providing module-related infrastructure to support administrators in that task is the subject of ongoing effort, with one of the most notable pieces being the work to support [module signing](<https://lwn.net/Articles/515007/>). 

Kees Cook has recently posted a series of patches that tackle another facet of the module-verification problem. These patches add a new system call for loading kernel modules. To understand why the new system call is useful, we need to start by looking at the existing interface for loading kernel modules. 

The Linux interface for loading kernel modules has had (since kernel 2.6.0) the following form: 

```
int init_module(void *module_image, unsigned long len,
                        const char *param_values);
```

The caller supplies the ELF image of the to-be-loaded module via the memory buffer pointed to by `module_image`; `len` specifies the size of that buffer. (The `param_values` argument is a string that can be used to specify initial values for the module's parameters.) 

The main users of `init_module()` are the `insmod` and `modprobe` commands. However any privileged user-space application (i.e., one with the `CAP_SYS_MODULE` capability) can load a module in the same way that these commands do, via a three-step process: opening a file that contains a suitably built ELF image, reading or `mmap()`ing the file's contents into memory, and then calling `init_module()`. 

However, this call sequence is the source of an itch for Kees. Because the step of obtaining a file descriptor for the image file is separated from the module-loading step, the operating system loses the ability to make deductions about the trustworthiness of the module based on its origin in the filesystem. As Kees [said](<https://lwn.net/Articles/514168/>): 

being able to reason about the origin of a kernel module would be valuable in situations where an OS already trusts a specific file system, file, etc, due to things like security labels or an existing root of trust to a partition through things like [dm-verity](<https://lwn.net/Articles/459420/>). 

His solution is fairly straightforward: remove the middle of the three steps posted above. Instead, the application will open the file and pass the returned file descriptor directly to the kernel as part of a new module-loading system call; the kernel then performs the task of reading the module image from the file as a precursor to loading the module. 

Although the concept of the solution is simple, it has been through a few iterations, with the most notable changes being to details of the user-space interface. Kees's initial proposal was to hack the existing `init_module()` interface, so that if `NULL` is passed in the `module_image` argument, the kernel would interpret the `len` argument as a file descriptor. Rusty Russell, the kernel modules subsystem maintainer, somewhat bluntly [suggested](<https://lwn.net/Articles/519006/>) that a new system call would be a better approach, and on the [next revision](<https://lwn.net/Articles/519008>) of the patch, H. Peter Anvin [pointed out](<https://lwn.net/Articles/519004/>) that the system call would be better named according to existing conventions, where the file descriptor analog of an existing system call simply uses the same name as that system call, but with an "f" prefix. Thus, Kees has arrived at the [currently proposed](<https://lwn.net/Articles/519009>) interface: 

```
int finit_module(int fd, const char *param_values);
```

In the most recent patch, Kees, who works for Google on Chrome OS, has also further elaborated on the motivations for adding this system call. Specifically, in order to ensure the integrity of a user's system, the Chrome OS developers would like to be able to enforce the restriction that kernel modules are loaded only from the system's read-only, cryptographically verified root filesystem. Since the developers already trust the contents of the root filesystem, employing [module signatures](<https://lwn.net/Articles/515007/>) to verify the contents of a kernel module would require the addition of an unnecessary set of keys to the kernel and would also slow down module loading. All that Chrome OS requires is a light-weight mechanism for verifying that the module image originates from that filesystem, and the new system call provides just that facility. 

Kees pointed out that the new system call also has potential for wider use. For example, Linux Security Modules (LSMs) could use it to examine digital signatures contained in the module file's extended attributes (the file descriptor provides the kernel with the route to access the extended attributes). During discussion of the patches, interest in the new system call was confirmed by the maintainers of the [IMA](<https://lwn.net/Articles/137306/>) and [AppArmor](<https://lwn.net/Articles/353837/>) kernel subsystems. 

At this stage, there appear to be few roadblocks to getting this system call into the kernel. The only question is when it will arrive. Kees would very much like to see the patches go into the currently open 3.7 merge window, but for [various reasons](<https://lwn.net/Articles/519027/>), it appears probable that they will only be merged in Linux 3.8. 

_Update, January 2013:_ `finit_module()` was indeed merged in Linux 3.8, but with a changed API that added a `flags` argument that can be used to modify the behavior of the system call. Details can be found in the manual page.
