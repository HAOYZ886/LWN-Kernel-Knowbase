---
title: The FUSE BPF filesystem
url: https://lwn.net/Articles/937433/
date: "July 11, 2023"
category: "BPF-Filesystems; Filesystems-In user space"
author: "By Jake Edge July 11, 2023 LSFMM+BPF"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
July 11, 2023

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2023>)

The [Filesystem in Userspace](<https://www.kernel.org/doc/html/next/filesystems/fuse.html>) (FUSE) framework can be used to create a "stacked" filesystem, where the FUSE piece adds specialized functionality (e.g. reporting different file metadata) atop an underlying kernel filesystem. The performance of such filesystems leaves a lot to be desired, however, so the [FUSE BPF filesystem](<https://lwn.net/ml/linux-kernel/20230418014037.2412394-1-drosen@google.com/>) has been proposed to try to improve the performance to be close to that of the underlying native filesystem. It came up in the context of a [session on FUSE passthrough](<https://lwn.net/Articles/932060/>) earlier in the [2023 Linux Storage, Filesystem, Memory-Management and BPF Summit](<https://lwn.net/Articles/lsfmmbpf2023>), but the details of FUSE BPF were more fully described by Daniel Rosenberg in a combined filesystem and BPF session on the final day of the summit. 

Rosenberg said that he wanted to introduce the filesystem, describe its current status, and discuss some of the open questions with regard to future plans for it. The goal is for a stacked FUSE filesystem to come as close to the native filesystem's performance as the FUSE BPF developers can get. In addition, they want to keep "all of the nice ease-of-use of FUSE", with its "defined entry points"; the idea is to keep the interface "similar to what you would see from the FUSE daemon". 

[ ![\[Daniel Rosenberg\]](https://static.lwn.net/images/2023/lsfmb-rosenberg-sm.png) ](<https://lwn.net/Articles/937828/>)

He put up a diagram showing what classic FUSE stacked filesystem does, which requires a "transition to and from user space several times" for each operation. The application makes a system call that reaches the FUSE driver in the kernel, which calls back out to the user-space FUSE daemon (for handling the specialized functionality). The FUSE daemon then makes another system call to the lower filesystem. The difference for FUSE BPF is that the in-kernel FUSE driver can use BPF to perform any filtering or other functionality for each operation in the kernel, then call directly into the VFS layer for the lower filesystem. If the FUSE functionality cannot be performed in BPF for some reason (e.g. consulting a database is required), it can call out to the FUSE daemon for the pre- and post-filtering. 

The pre- and post-filters are what provide the functionality specific to that FUSE filesystem, whatever it is. For Android, which is where FUSE BPF is being used, there are specific directories that are being hidden using the filtering. You could also imagine a filter that changes the data being read in order to hide something from the applications. 

The implementation uses the [BPF operation structures](<https://lwn.net/Articles/811631/>) (`struct_ops`) to replace certain kernel operations with the BPF filters. Those filters mostly access structures that contain the arguments for the operation, though there is a special buffer type to contain variable-length arguments, such as strings or data buffers. The BPF programs have the option of falling back to the normal FUSE path if desired. 

One of the benefits of the operation-structure approach is that the FUSE BPF filesystem only needs to provide code for the operations it wants to intercept. In a "very dumb example that you would never actually want to do", a stacked filesystem that simply adds a character to the end of all file names would only need to implement filters for the lookup and directory-read operations. 

In order to use FUSE BPF, the `struct_ops` program needs to be registered with the system using [`bpftool`](<https://github.com/torvalds/linux/blob/master/tools/bpf/bpftool/Documentation/bpftool-struct_ops.rst>). After that, either at mount or lookup time, the program needs to be associated with the inode or dentry of interest using a file descriptor for the backing file or directory in the lower filesystem. The developers are willing to entertain other ideas of ways to identify the backing file, but a file descriptor was easy for their use case. 

Rosenberg put up a table of benchmarks that used a tmpfs RAM-based filesystem as the lower filesystem, which exaggerates the performance improvements that are seen. The benchmarks show near-native performance, but with a more complex lower filesystem, the performance improvements much smaller, he said. He asked Paul Lawrence, who had run the benchmarks, if he wanted to comment. Lawrence agreed that the tmpfs testing showed a much larger benefit from FUSE BPF versus regular FUSE than would be seen in more realistic testing. 

Rosenberg said that there are some things that they are still working on. One big thing on their to-do list is to perform the operations using the credentials of the FUSE daemon. There are additional FUSE opcodes, including the `ioctl()` opcode, that need to be implemented. Beyond that, some of the pre- and post-filters "are not fully hooked up"; he is waiting to see if some of that needs to change before rolling it out to all of the different opcodes. 

Aleksa Sarai said that io_uring had some similar problems with credentials, so he suggested looking at what those developers did as a model; he thought that it involved creating a thread from the process whose credentials should be used. But Christian Brauner said that, for the 6.4 kernel, he had merged a generic API called "user workers" for doing this kind of credential handling. Rosenberg said that he was "all for using pre-existing stuff". 

Lawrence said that thread and worker-queue switching on Android leads to a huge increase in latency, to the point where it cannot be shipped. They had run into problems with [dm-verity](<https://docs.kernel.org/admin-guide/device-mapper/verity.html>) due to its worker threads, for example. But he thinks the FUSE BPF credential problem can be solved fairly simply by running the I/O in the context of the FUSE daemon. That is the normal FUSE model, so he thinks FUSE BPF should stick with it. 

One attendee asked about the optional pre- and post-filters in user space; if you are going to have to pay the price to call out to user space anyway, does it make sense to just do all the processing there? Rosenberg said that one transition between kernel and user space can still be saved in that case, though there is less of a benefit. But the reason that you are doing the filtering in user space is probably because there is a lot of work that needs to be done in the filter, the attendee said. Lawrence said that in the Android use case, there are just some small pieces that need to be handled with the user-space filters and the vast majority of the file operations will use the BPF filters and stay in the kernel, saving the transition cost. He did acknowledge that there might be a better way to handle that, but that they had found it useful to have the user-space filters for Android. 

Josef Bacik wanted to confirm his understanding of what features FUSE BPF would add. In particular, there are two separate pieces: adding the ability to pass operations directly to the underlying filesystem using the file-descriptor registration and the ability to attach the BPF pre- and post-filters for the operations on the upper filesystem. For filesystem-passthrough purposes, the lookup operation could open the underlying file, then associate that descriptor, and all of the rest of the operations would go directly to the underlying filesystem. Rosenberg agreed with that, noting that there would not be any BPF needed for handling passthrough. 

Another attendee wondered if the only way to do the association was at lookup time; for the [composefs use case](<https://lwn.net/Articles/933616/>), it would be better done when the file is opened. Lawrence said that Meta had already asked that FUSE BPF move away from requiring file-descriptor registration and that a path should be used instead, so that the association can be done without an open file. The plan is to change to using a path relative to a file descriptor (either of which can be null) for the association, which is the usual convention for the `*at()` system calls. 

There was some discussion of allowing association at either lookup or open time, but Lawrence said that they have not looked into that deeply yet. It was fairly straightforward to only allow associating files at lookup time, but it may make sense to broaden that. Brauner said that adding association at open time would really complicate the code; he suggested keeping the implementation simple, at least for now. 

Sarai said that it was important to use the [`openat2()`](<https://man7.org/linux/man-pages/man2/openat2.2.html>) resolve flags when the switch was made to do relative lookups. There are classic problems when resolving paths that can allow malicious programs to access parts of the filesystem that should not be allowed. If the proper resolve flags are used, that should easily eliminate those kinds of escapes. 

Rosenberg said that, currently, there is a problem falling back to regular FUSE because the BPF code uses a FUSE node ID of zero when it creates nodes, but FUSE does not understand that value. There needs to be a way to reserve a block of node IDs for BPF to use, but it is not a problem that they have encountered so it has been deferred. There are also two issues with the `struct_ops`: there is no module support for them, so he has hacked around that, and there are a whole raft of `struct_ops` callbacks needed, which required him to allocate two pages of memory to hold them all. Those both need to be cleaned up. 

They have some plans for upstreaming the code. The patch set is rather large at this point, with more than 30 patches, he said, which he is trying to arrange to make them as easy as possible to review. The current organization has the passthrough patches first, with the BPF pieces coming later in the patch series. 

Bacik suggested that he split the series into two, with the passthrough pieces as the first. When he tried to review the current series, it was difficult to grasp that FUSE BPF is made up of two separate things. He recognizes that it is a single project from their perspective but it would help him and perhaps others to break things up. Rosenberg acknowledged that, but noted that the Android developers do not have a use for the pure passthrough version, though it is a good intermediate point. 

Brauner asked why the passthrough piece was useful at all, but Amir Goldstein emphatically said "I don't understand, it's _so_ useful". Once a file has been looked up or opened, all of its I/O can be passed through directly to the filesystem; he suggested that providing passthrough for directory operations would also be useful, but others were less sure. Miklos Szeredi agreed that file passthrough was useful, while Bacik thought that all of it was, just that it should be broken up for review purposes.
