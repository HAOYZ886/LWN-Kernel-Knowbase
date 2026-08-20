---
title: Some vmsplice() issues
url: https://lwn.net/Articles/592061/
date: "March 26, 2014"
category: "System calls-vmsplice; splice"
author: "By Jonathan Corbet March 26, 2014 2014 LSFMM Summit"
---

By **Jonathan Corbet**  
March 26, 2014

* * *

[2014 LSFMM Summit](<https://lwn.net/Articles/LSFMM2014/>)

Pavel Emelyanov works with the checkpoint-restart in user space (CRIU) project. One of the use cases for CRIU is live migration of processes from one host to another; that involves the moving of a lot of memory to and from sockets. The `vmsplice()` interface seems like an ideal tool for doing that work without unnecessarily copying the data. But in the process of using `vmsplice()` for this purpose, Pavel has run into a number of issues. In the final plenary session at the 2014 Linux Storage, Filesystem, and Memory Management workshop, Pavel discussed the problems he has encountered and their possible solutions. 

One problem is that using a pipe to move pages of memory — part of the process of using `vmsplice()` — requires opening two separate file descriptors. CRUI needs to open a lot of pipes, so it tends to run into the limit on the total number of open file descriptors. Al Viro described a possible workaround: find one of the pipe file descriptors under `/proc`, open it as a read/write file descriptor, then close the two original descriptors. That will cut the number of required file descriptors in half. 

`vmsplice()`, when used with the `SPLICE_F_GIFT` flag, is meant to hand the indicated pages of data directly to the kernel without copying the data. But, Pavel said, it often ends up copying those pages anyway, even though it seems the copying should not be necessary. Some digging through the commit logs suggests that things were done this way to avoid surprising filesystems with pages of data coming from an unexpected direction. The filesystem developers seemed to agree that the amount of work required to handle such pages would be quite small, so perhaps this behavior could be changed. An action item was taken to try to query Nick Piggin (the original author of this code, who has since disappeared from the kernel community) about whether there are any other subtle issues that might prevent greater use of zero-copy transfers. 

Pavel's next problem is that pages sent to files with `vmsplice()` go into the page cache, but he would rather have them bypass the page cache and be written directly to the target file. It was pointed out that splicing to a file descriptor opened with `O_DIRECT` should work properly; at that point, the rest of the problem description came out. An `O_DIRECT` file descriptor does indeed work, but writes are synchronous, slowing things down. Pavel would rather there were a way to do asynchronous `O_DIRECT` writes via `vmsplice()`. Al allowed that it might be possible to make this work, but the job "might not be fun." 

The final problem had to do with how to send pages out of another process's address space without actually copying them. James Bottomley suggested that some of the machinery behind the `fork()` system call could be used. The process would not actually be forked, but a copy of its address space would be made so that the migration process could get to its pages directly. The implementation of this functionality could be tricky but, if it could be done, it might make process migration significantly more efficient. 

[Your editor would like to thank the Linux Foundation for supporting his travel to the Summit.]
