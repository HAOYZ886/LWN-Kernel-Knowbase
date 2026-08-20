---
title: Persistent BPF objects
url: https://lwn.net/Articles/664688/
date: "November 18, 2015"
category: BPF
author: "By Jonathan Corbet November 18, 2015"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
November 18, 2015

With the addition of the [`bpf()` system call](<https://lwn.net/Articles/612878/>) in the 3.18 development cycle, user space gained the ability to load extended BPF programs into the kernel and to share data areas (called "maps") with them. The 4.4 kernel will take things further by [making it possible](<https://lwn.net/Articles/660331/>) for unprivileged processes to perform BPF operations. As interest in using BPF increases, though, some of the limitations of the initial design are starting to show through; one of those is the inability to create BPF objects (programs or maps) that outlive the process that creates them. That particular shortcoming will be addressed by another patch set, also merged for 4.4. 

The original thinking behind the lifecycle of BPF objects was that they would be created and used by a single process. Current uses, though, are stretching that model. The network traffic-control subsystem, for example, may want to attach both classification and dispatching BPF programs to a traffic policy; that policy should then live after the creating invocation of `tc` has exited. Tracing applications, too, may involve setting up programs and maps that should persist for a while. 

In pre-4.4 kernels, the only way to make these objects persist is to ensure that some process keeps the file descriptor open. One can create a special daemon that functions as a shelf for file descriptors, then pass BPF objects to it over Unix-domain sockets, but this solution lacks elegance and could be difficult to secure. If there is a true use case for persistent BPF objects, the kernel probably should support them directly. 

One can say that, however, without answering the question of just how the persistence mechanism should work. In this case, it seems that the BPF developers considered just about every possible option. One could use a special FUSE filesystem to hold the file descriptors, but that really looks like a variant on the dedicated daemon idea. One could create a special namespace, the way network sockets or System IPC objects are handled, but the interface is awkward and inaccessible to shell scripts. For a while the developers even played with the idea of creating special devices for persistent BPF objects, but that idea went down on concerns of memory use and inability to play well with namespaces. 

So what we have instead is yet another special kernel virtual filesystem. This one is meant to be mounted at `/sys/fs/bpf`. It is a singleton filesystem, meaning that it can be mounted multiple times within a single namespace and every mount will see the same directory tree. Each mount namespace will, however, get its own version of this filesystem. Within `/sys/fs/bpf`, a suitably privileged user can create and remove directories in the usual ways to set up a suitable directory hierarchy. 

The "files" in this hierarchy, which represent persistent BPF objects, must be managed with the `bpf()` system call, though. The new `BPF_PIN_FD` `bpf()` command can be used to "pin" a file descriptor into the BPF filesystem; it takes a file descriptor corresponding to a BPF object and a path name as arguments. Once the `BPF_PIN_FD` call has succeeded, the associated BPF object will be made persistent and visible in the filesystem at the given path name. To unpin an object, ending its persistence, one need only remove the associated file in the usual way. 

To access the persistent object, one must use another new `bpf()` command called `BPF_GET_FD`. It functions much like an `open()` call, in that it takes a path name and returns a file descriptor corresponding to that path. That file descriptor may then be used with other `bpf()` operations as needed. 

Given that `BPF_GET_FD` looks like `open()`, one might well wonder why programs can't simply call `open()` instead. This was, evidently, a deliberate design decision; [according to Alexei Starovoitov](<https://lwn.net/Articles/664689/>): 

We've considered letting open() of the file return bpf specific anon-inode, but decided to reserve that for other more natural file operations. Therefore BPF_NEW_FD is needed. 

(The `BPF_NEW_FD` command was present in an earlier version of the patch, but is not part of what was merged into 4.4). 

The nature of these "more natural" operations was not laid out. There has been some discussion, though, of exposing BPF maps directly in the filesystem namespace. A map is essentially a key/value store, so one could consider representing it as a directory, with each key showing up as a "file" within it. The true value of this feature is not entirely clear, and it could get awkward when one considers that keys can be arbitrary binary data; they need not follow the rules that apply to file names. So it's perhaps not surprising that this feature is not present in the current patch set. 

For the curious, the developers [included](<http://git.kernel.org/linus/42984d7c1e563bf92e6ca7a0fd89f8e933f2162e>) an example program under `samples/bpf`. Now it is up to distributors to decide whether they want to mount `/sys/fs/bpf` by default, and for application developers to make use of this new capability.
