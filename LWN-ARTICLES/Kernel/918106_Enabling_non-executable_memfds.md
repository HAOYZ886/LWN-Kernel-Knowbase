---
title: "Enabling non-executable memfds"
url: https://lwn.net/Articles/918106/
date: "December 19, 2022"
category: "Memfd; Releases-6.3"
author: "By Jonathan Corbet December 19, 2022"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
December 19, 2022

The memfd interface is a bit of a strange and Linux-specific beast; it was initially [created](<https://lwn.net/Articles/593918/>) to support the secure passing of data between cooperating processes on a single system. It has since gained other roles, but it may still come as a surprise to some to learn that memory regions created for memfds, unlike almost any other data area, have the execute permission bit set. That can facilitate attacks; [this patch set](<https://lwn.net/ml/linux-kernel/20221209160453.3246150-1-jeffxu@google.com/>) from Jeff Xu proposes an addition to the memfd API to close that hole. 

A memfd is created with a call to [`memfd_create()`](<https://man7.org/linux/man-pages/man2/memfd_create.2.html>), which will return a file descriptor referring to the region. That descriptor can be treated as an ordinary file, in that it can be written to or read from; it can also be mapped into a process's address space. Normally the first step will be to call [`ftruncate()`](<https://man7.org/linux/man-pages/man2/ftruncate.2.html>) to set the size of the region; after that it can be populated with data and passed to another process. One interesting characteristic of memfds is that they can be "sealed" with a call to [`fcntl()`](<https://man7.org/linux/man-pages/man2/fcntl.2.html>), an operation that disallows any further changes to the stored data. Sealing allows a recipient to know that the contents of a memfd will not change in unexpected ways in the middle of an operation. 

As it happens, the virtual file that underlies a memfd is created with the execute permission bits set; that allows the memory itself to be mapped as executable. The result is a combination of permissions — both write and execute permission enabled — that developers in both the kernel and user space are increasingly going out of their way to avoid. A memory area that is both writable and executable gives attackers a relatively easy way to inject their own code into a target process. And, indeed, Xu notes in the patch cover letter that memfd areas have been used in just that way to attack ChromeOS systems. 

One might be tempted to respond by just removing the execute permission from the underlying memfd file unconditionally. But at this point that would be an ABI change, and there is at least one known (legitimate) user of executable memfds. The [runc](<https://github.com/opencontainers/runc#readme>) container runtime uses an executable memfd to load the image of the container it is about to run; that feature was [added in response to another vulnerability](<https://lwn.net/Articles/781013/>) in 2019. So the ability to have an executable memfd must remain. 

Executable memfds do not necessarily have to be the default, though, and processes can definitely be given the ability to make a non-executable memfd. Xu's patch set thus modifies the memfd API in that direction by adding a pair of new flags for `memfd_create()`: 

  * `MFD_EXEC` explicitly asks `memfd_create()` to create a memfd with execute permission set. That simply reinforces the current default, but the default can be changed as described below. 
  * `MFD_NOEXEC_SEAL`, instead, creates a memfd without execute permission, and applies a seal that prevents that setting from ever being changed. A memfd created with this flag will thus never be executable no matter how hard a user-space attacker might try. 

There is a new `fcntl()` operation, `F_SEAL_EXEC`, that seals the execute permission and prevents it from being changed thereafter. As with all sealing operations, this change cannot be undone afterward. 

The patches also add a new sysctl knob, called `vm.memfd_noexec`, that is local to the current PID namespace; it controls what the kernel does when the affected process creates a memfd without specifying either of the two new flags. Setting that knob to zero causes `memfd_create()` to behave as if `MFD_EXEC` were set — the current behavior. Setting it to one, instead, causes `MFD_NOEXEC_SEAL` to be set, essentially turning off execute permission by default. A value of two will cause any `memfd_create()` call that does not explicitly provide `MFD_NOEXEC_SEAL` to fail, disabling executable memfds entirely. The default, naturally, must be zero to avoid breaking any existing applications. 

The new code emits a warning to the kernel log if neither of the two flags is set when a memfd is created, in the hope of causing applications to be updated. Peter Xu [observed](<https://lwn.net/ml/linux-kernel/Y5yS8wCnuYGLHMj4@x1n/>) that this could fill the system log with a lot of warnings; after some discussion, it was agreed to emit the warning only once per boot. As a result, it could take several boot cycles to discover all of the applications that need to be fixed on a given system, but that was deemed preferable to unlimited logging. 

Finally, Paul Moore has [questioned](<https://lwn.net/ml/linux-kernel/CAHC9VhRBMTQvnBdSwMbkOsk9eemYfvCmj9TRgxtMeuex4KLCPA@mail.gmail.com/>) the addition of a security-module hook for `memfd_create()` since there is no corresponding change to a security module to actually use that hook. As a result, it's possible that the hook might be taken out until somebody wants to write a policy for this system call. Otherwise, the patch series appears to be ready for merging.
