---
title: Filesystem sandboxing with eBPF
url: https://lwn.net/Articles/803890/
date: "November 6, 2019"
category: BPF; Filesystems
author: "By Jake Edge November 6, 2019 OSS EU"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jake Edge**  
November 6, 2019

* * *

[OSS EU](<https://lwn.net/Archives/ConferenceByYear/#2019-Open_Source_Summit_Europe>)

Running untrusted code in a safe manner is generally the goal of sandboxing efforts. The sandbox technique presented by Georgia Tech PhD student Ashish Bijlani at [Open Source Summit Europe 2019](<https://events19.linuxfoundation.org/events/open-source-summit-europe-2019/>) is no exception. He has used something of a novel scheme to allow unprivileged code to implement the sandbox policies using BPF; the policies are then enforced by the kernel. 

#### Background

There are lots of use cases for running untrusted third-party code without risking the contents of files on the system. Two that he mentioned were web-browser plugins obtained from potentially dodgy internet sites and machine-learning code that one might like to evaluate. There is a spectrum of code that can be run, from known-good code to known-bad code; in between those is unknown, untrusted code. Known-good code can be whitelisted and known-bad code can be blacklisted, sandboxing is a technique used for that code in the middle. A sandbox is simply an isolated and controlled execution environment, he said. 

[ ![\[Ashish Bijlani\]](https://static.lwn.net/images/2019/osseu-bijlani-sm.jpg) ](<https://lwn.net/Articles/804044/>)

Bijlani is focused on a specific type of sandbox: a filesystem sandbox. The idea is to restrict access to sensitive data when running these untrusted programs. The rules would need to be dynamic as the restrictions might need to change based on the program being run. Some examples he gave were to restrict access to the `~/.ssh/id_rsa*` files or to only allow access to files of a specific type (e.g. only `*.pdf` for a PDF reader). 

He went through some of the existing solutions to show why they did not solve his problem, comparing them on five attributes: allowing dynamic policies, usable by unprivileged users, providing fine-grained control, meeting the security needs for running untrusted code, and avoiding excessive performance overhead. Unix discretionary access control (DAC)—file permissions, essentially—is available to unprivileged users, but fails most of the other measures. Most importantly, it does not suffice to keep untrusted code from accessing files owned by the user running the code. SELinux mandatory access control (MAC) does check most of the boxes (as can be seen in the talk [slides [PDF]](<https://static.sched.com/hosted_files/osseu19/20/OSSEUSandFS.pdf>)), but is not available to unprivileged users. 

Namespaces (or `chroot()`) can be used to isolate filesystems and parts of filesystems, but cannot enforce security policies, he said. Using `LD_PRELOAD` to intercept calls to filesystem operations (e.g. `open()` or `write()`) is a way for unprivileged users to enforce dynamic policies, but it can be bypassed fairly easily. System calls can be invoked directly, rather than going through the library calls, or files can be mapped with `mmap()`, which will allow I/O to the files without making system calls. Similarly, `ptrace()` can be used, but it suffers from [time-of-check-to-time-of-use](<https://en.wikipedia.org/wiki/Time-of-check_to_time-of-use>) (TOCTTOU) races, which would allow the security protections to be bypassed. 

`ptrace()` also suffers from high performance overhead (roughly 50%), as does the final option that Bijlani outlined: [Filesystem in Userspace](<https://en.wikipedia.org/wiki/Filesystem_in_Userspace>) (FUSE). A FUSE filesystem would check all of his boxes, but it suffers from nearly 80% performance overhead. He was looking for a solution that would only add 5-10% overhead, he said. 

That is what he has created with [SandFS](<https://sandfs.github.io>). It is a stackable filesystem that can enforce unprivileged-user-specified policies on filesystem access. A user would invoke it this way: 

```
$ sandfs -s sandfs.o -d /home/user /bin/bash
```

The `sandfs` binary is unprivileged; it can be run by anyone. The example above would run `bash` within a sandbox for accesses to the `/home/user` directory. The sandbox is defined by `sandfs.o`, which is written in C and compiled by LLVM into BPF bytecode. 

He talked a bit about BPF and how it can be used, calling BPF "a key enabling technology" for SandFS. BPF maps provide a mechanism to communicate between user space and BPF programs running in the kernel; they also have a major role to play for SandFS. More details on BPF can be found in [this LWN article](<https://lwn.net/Articles/740157/>). 

#### Architecture

He then turned to the architecture of SandFS; there are a few different components to it, starting with the SandFS daemon and SandFS library in user space. The daemon is what the `sandfs` binary talks to and the library is available for those developing their own security policies. There is also a modified version of [Wrapfs](<http://wrapfs.filesystems.org/>) that is used to intercept the filesystem operations for the mounted filesystem. A set of SandFS BPF handlers are available in the kernel to implement the security checking for each of the filesystem operations that are intercepted by SandFS itself, which is the filesystem based on Wrapfs. 

The basic operation is that the `sandfs` binary sends the BPF code to the daemon, which loads it into the kernel. If the BPF verifier does not find a problem with the code, the next step is to mount SandFS on the directory specified (`/home/user` in the example). Any filesystem operations will be intercepted by SandFS, which will call out to the BPF programs loaded from user space in order to get access decisions. SandFS itself does not perform I/O, it simply passes any operations that were allowed by the policies down to the lower-level filesystem (e.g. ext4 or XFS). 

The policies can consult BPF maps, which can be written from user space; that allows for dynamic policies. The BPF programs passed in from user space in may look things up in the maps, such as path names, to determine whether to allow access or not; it is even possible to alter parameters to the filesystem operations based on the policies (e.g. to make all `open()` calls read-only). SandFS handles kernel objects, rather than parameters directly passed by user space, so it avoids any TOCTTOU problems. 

In the talk, he gave two example of BPF programs that could be used to restrict access. The first would consult the BPF map for the path being used as part of the `lookup()` filesystem operation; if it found the path in the map, it would return `-EACCES`, thus providing a way for user space to restrict access to any part of the sandboxed directory. The second would look at the mode specified in `open()` operations, rejecting those with `O_CREAT` and changing the mode to `O_RDONLY` for the rest. 

He then showed some performance numbers for a few different types of operations, comparing the time taken for them on ext4 versus SandFS. Creating a `.tar.gz` file of the 4.17 kernel showed the lowest overhead (4.57%, 61.05s vs. 63.84s). Decompressing and expanding the tar file had the most overhead (9.75%, 5.13s vs. 5.63s), while compiling the kernel (`make ‑j 4 tinyconfig`) came in at 9.28% (27.15s vs. 29.67s). 

The SandFS framework could be used in a number of different ways, Bijlani said. It could restrict access to private user data such as SSH keys. It could also be used to compartmentalize certain operations of a complex application, such as a web browser; handling file and media formats could be put into separate sandboxed processes. Also, container-management systems could stack multiple layers of SandFS checks to harden the filesystem access from their containers. 

He wrapped up the talk by noting the the SandFS code is [available on GitHub](<https://github.com/sandfs/sandfs.github.io>). He has written an [academic paper](<https://dl.acm.org/citation.cfm?id=3265734>) on it as well. In addition, he pointed to some related work that he presented at OSS North America in 2018 ([slides [PDF]](<https://events19.linuxfoundation.org/wp-content/uploads/2017/11/When-eBPF-Meets-FUSE-Improving-Performance-of-User-File-Systems-Ashish-Bijlani-Georgia-Tech.pdf>)) and at the 2018 Linux Plumbers Conference ([YouTube video](<https://www.youtube.com/watch?v=XmoJCHNEp2w>)). 

[I would like to thank LWN's travel sponsor, the Linux Foundation, for travel assistance to attend Open Source Summit Europe in Lyon, France.]
