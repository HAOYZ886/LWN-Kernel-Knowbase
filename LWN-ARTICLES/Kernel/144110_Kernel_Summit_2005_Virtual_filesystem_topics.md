---
title: "Kernel Summit 2005: Virtual filesystem topics"
url: https://lwn.net/Articles/144110/
date: "July 19, 2005"
category: "Filesystems-Virtual filesystem layer"
author: 
---

From LWN's [2005 Kernel Summit coverage](<https://lwn.net/Articles/KernelSummit2005/>).   
---  
Monday's final session was a discussion of a number of virtual filesystem topics, led by Suparna Bhattacharya. There was no overriding theme to this session; it was more a collection of outstanding issues. 

The first of these issues is `mm/filemap.c` in the kernel source. This code once used to be readable, but it has turned into a complicated mess. As an example, consider a function like `generic_file_write()`, whose purpose should be obvious from its name. In fact, it is not so generic; `filemap.c` contains: 

```
generic_write_checks()
        generic_file_write()
        generic_file_writev()
        generic_file_direct_write()
        generic_file_buffered_write()
        generic_file_write_nolock()
        generic_file_aio_write()
        generic_file_aio_write_nolock()
```

As the VFS has gotten more complicated, and, in particular, as it has gained support for features like direct I/O, the interfaces have gotten somewhat out of control. Locking, in particular, has become complex, with different I/O modes having different locking regimes. The VFS is now almost unapproachable for many programmers. 

One possibility for simplifying things would be to eliminate the concurrent direct and buffered I/O on the same file. If only one mode of access had to be considered (perhaps enforced by way of a mount or chattr option), some of the code could be simplified. It was quickly determined, however, that the kernel would have to continue to support both modes of access on the same file. Otherwise, for example, how might one back up a file which is currently under direct I/O (assuming that is a smart thing to do in the first place, of course)? Direct I/O is also something which must be done with great care; an application must be aware of what it is doing. So any sort of option which would cause unaware applications to perform direct I/O would lead to certain failure. 

Wim Coekaerts noted that his group has written patches for a number of GNU utilities (such at `tar`) enabling them to perform direct I/O. Those patches have never been accepted, however. 

One way of simplifying the situation, and helping user space as well, would be to provide support for preallocation of blocks in files. Something along the lines of the `[posix_fallocate()](<http://www.opengroup.org/onlinepubs/009695399/functions/posix_fallocate.html>)` function. This idea made sense to most; it just needs somebody to implement it. 

Another helpful change would be to put direct I/O pages into the page cache; then many locking issues simply go away. Of course, the whole point of direct I/O is to avoid the page cache. So the page cache entries would have to point to the existing user-space pages, perhaps by way of some sort of virtual `struct page`. This is a scary idea; there is a great deal of kernel code which assumes that each `page` structure corresponds to a physical page in memory. Changing such a fundamental assumption in a safe way could be a challenge; that said, the task would almost certainly be easier now than it would have been a few years ago. 

The continuing existence of buffer heads has been raised as a problem more than once this day. They come about as a result of mismatches between the filesystem block size and the system's page size; buffer heads are also used in the ext3 journaling code. Buffer heads have been around forever, but they have to live in low memory (which can be scarce on big systems), and they require the existence of separate code paths to deal with them. Nonetheless, getting rid of buffer heads in the near future will be hard, and whatever replaces them may turn out to be just as complex. 

Delayed allocation, multi-block allocation, and extents were also pointed out as desirable features. The question: should they be implemented within individual filesystems, or as generic code in the VFS layer? Linus stated that he tends to be against generic code when it starts to get complex. If things get too twisted, it can be better to have simpler, filesystem-specific implementations. His suggestion was to fix the `filemap.c` mess first; once that has been simplified, one can consider adding other generic capabilities to the VFS layer. 

There are lock ordering issues which will have to be faced at some point. Multi-block allocations naturally call for a lock ordering regime (specific locks first, then more general locks) which is contrary to what is done now. Cluster filesystems will create the need to lock multiple inodes at once; at that point, the order in which those locks is taken is of crucial importance. Lock ordering mistakes can lead to system deadlocks. 

The shared subtree patches were mentioned briefly. Shared subtrees come out of [a suggestion by Al Viro](<http://lwn.net/Articles/119232/>); they are intended as a way to make the same filesystem tree be simultaneously available in multiple parts of the system namespace. This proposal is, among other things, a response to some of the things the reiser4 filesystem is trying to do. Unfortunately, Al Viro was not able to attend the summit, and few developers had looked at this patch. So there was not much discussion.
