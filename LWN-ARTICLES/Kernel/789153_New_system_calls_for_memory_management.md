---
title: New system calls for memory management
url: https://lwn.net/Articles/789153/
date: "May 24, 2019"
category: Memory management
author: "By Jonathan Corbet May 24, 2019"
---

By **Jonathan Corbet**  
May 24, 2019

Several new system calls have been proposed for addition to the kernel in a near-future release. A few of those, in particular, focus on memory-management tasks. Read on for a look at `process_vm_mmap()` (for zero-copy data transfer between processes), and two new APIs for advising the kernel about memory use in a different process. 

#### process_vm_mmap()

There are many use cases for quickly moving data from one process to another; message-passing applications are one example, but far from the only one. Since [the 3.2 development cycle](<https://lwn.net/Articles/465066/>), there has been a pair of specialized, little-known [system calls](<http://man7.org/linux/man-pages/man2/process_vm_readv.2.html>) intended for this purpose: 

```
ssize_t process_vm_readv(pid_t pid, const struct iovec  *lvec, 
    			     unsigned long liovcnt, const struct iovec *rvec,
    		 	     unsigned long riovcnt, unsigned long flags);
    
        ssize_t process_vm_writev(pid_t pid, const struct iovec  *lvec, 
    			      unsigned long liovcnt, const struct iovec *rvec,
    		 	      unsigned long riovcnt, unsigned long flags);
```

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Both calls copy data between the local address space (as described by the `lvec` array) and the remote space (described by `rvec`); they do so without moving the data through kernel space. For certain kinds of traffic they are quite efficient, but there are exceptions, especially as the amount of copied data gets large. 

[The cover letter](<https://lwn.net/ml/linux-kernel/155836064844.2441.10911127801797083064.stgit@localhost.localdomain/>) for a patch set from Kirill Tkhai describes the problems some have encountered with these system calls: they have to actually pass over and access all of the data while copying it. If the data of interest happens to be swapped out, it will be brought back into RAM. The same is true for the destination; additionally, if the destination side does not have pages allocated in the given address range, more memory will have to be allocated to hold the copy. Then, all of the data passes through the CPU, thus wiping out the (presumably more useful) data already there. This leads to problems like: 

We observe similar problem during online migration of big enough containers, when after doubling of container's size, the time increases 100 times. The system resides under high IO and throwing out of useful caches. 

Tkhai's solution is to introduce a new system call that avoids the copying: 

```
int process_vm_mmap(pid_t pid, unsigned long src_addr, unsigned long len,
    			unsigned long dst_addr, unsigned long flags);
```

This call is much like `mmap()`, in that it creates a new memory mapping in the calling process's address space; that mapping (possibly) starts at `dst_addr` and is `len` bytes long. It will be populated by the contents of the memory range starting at `src_addr` in the process identified by `pid`. There are a couple of flags defined: `PVMMAP_FIXED` to specify an exact address for the mapping and `PVMMAP_FIXED_NOREPLACE` to prevent a fixed mapping from replacing an existing mapping at the destination address. 

The end result of the call looks much like what would happen with `process_vm_readv()`, but with a significant difference. Rather than copying the data into new pages, this system call copies the source process's page-table entries, essentially creating a shared mapping of the data. Avoiding the need to copy the data and possibly allocate new memory for it speeds things considerably; this call will also avoid swapping in memory that has been pushed out of RAM. 

The response to this patch set has been guarded at best. Andy Lutomirski [didn't think the new system call would help](<https://lwn.net/ml/linux-kernel/CALCETrU221N6uPmdaj4bRDDsf+Oc5tEfPERuyV24wsYKHn+spA@mail.gmail.com/>) to solve the real problems and called the API ""quite dangerous and complex"". Some of his concerns were addressed in the following conversation, but he is still unconvinced that the problem can't be solved with `[splice()](<http://man7.org/linux/man-pages/man2/splice.2.html>)`. Kirill Shutemov [worried](<https://lwn.net/ml/linux-kernel/20190522152254.5cyxhjizuwuojlix@box/>) that this functionality might not play well with the kernel's reverse-mapping code and that it would ""introduce hard-to-debug bugs"". This discussion is still ongoing; `process_vm_mmap()` might eventually find its way into the kernel, but there will need to be a lot of questions answered first. 

#### Remote madvise()

There are times when one process would like to call [`madvise()`](<http://man7.org/linux/man-pages/man2/madvise.2.html>) to change the kernel's handling of another process's memory. In the case [described](<https://lwn.net/ml/linux-kernel/20190516094234.9116-1-oleksandr@redhat.com/>) by Oleksandr Natalenko, it is desirable to get a process to use kernel same-page merging (KSM) to improve memory utilization. KSM is an opt-in feature that is requested with `madvise()`; if the process in question doesn't happen to make that call, there is no easy way to cause it to happen externally. 

Natalenko's solution is to add a new file (called `madvise`) to each process's `/proc` directory. Writing `merge` to that file will have the same effect as an `madvise(MADV_MERGEABLE)` call covering the entire process address space; writing `unmerge` will turn off KSM. Possible future enhancements include the ability to affect only a portion of the target's address space and supporting other `madvise()` operations. 

The reaction to this patch set has not been entirely enthusiastic either. Alexey Dobriyan [would rather see](<https://lwn.net/ml/linux-kernel/20190516172452.GA2106@avx2/>) a new system call added for this purpose. Michal Hocko [agreed](<https://lwn.net/ml/linux-kernel/20190515145151.GG16651%40dhcp22.suse.cz/>), suggesting that the "remote `madvise()`" idea [briefly discussed](<https://lwn.net/Articles/787217/>) at this year's [Linux Storage, Filesystem, and Memory-Management Summit](<https://lwn.net/Articles/lsfmm2019/>) might be a better path to pursue. 

#### process_madvise()

As it happens, Minchan Kim has come along with [an implementation](<https://lwn.net/ml/linux-kernel/20190520035254.57579-1-minchan@kernel.org/>) of the remote `madvise()` idea. This patch set [introduces a system call](<https://lwn.net/ml/linux-kernel/20190520035254.57579-6-minchan@kernel.org/>) that looks like this: 

```
int process_madvise(int pidfd, void *addr, size_t length, int advice);
```

The result of this call is as if the process identified by `pidfd` (which is a pidfd file descriptor, rather than a process ID) called `madvise()` on the memory range identified by `addr` and `length` with the given `advice`. This API is relatively straightforward and easy to understand; it also only survived until [the next patch in the series](<https://lwn.net/ml/linux-kernel/20190520035254.57579-7-minchan@kernel.org/>), which rather complicates things: 

```
struct pr_madvise_param {
        	int size;
        	const struct iovec *vec;
        }
    
        int process_madvise(int pidfd, ssize_t nr_elem,
    		    	int *behavior,
    		    	struct pr_madvise_param *results,
    		    	struct pr_madvise_param *ranges,
    		    	unsigned long flags);
```

The purpose of this change was to allow a single `process_madvise()` call to make changes to many parts of the target process's address space. In particular, the `behavior`, `results`, and `ranges` arrays are each `nr_elem` elements long. For each entry, `behavior` is the set of `madvise()` flags to apply, `ranges` is a set of memory ranges held in the `vec` array, and `results` is an array of destinations for the results of the call on each range. 

The patch set also adds a couple of new `madvise()` operations. `MADV_COOL` would cause the indicated pages to be moved to the head of the inactive list, causing them to be reclaimed in the near future (and, in particular, ahead of any pages still on the active list) if the system is under memory pressure. `MADV_COLD`, instead, moves the pages to the _tail_ of the inactive list, possibly causing them to be reclaimed immediately. Both of these features, evidently, are something that the Android runtime system could benefit from. 

The reaction to this proposal was warmer; when most of the comments are related to naming, chances are that the more fundamental issues have been taken care of. Christian Brauner, who has done most of the pidfd work, [requested](<https://lwn.net/ml/linux-kernel/20190521084158.s5wwjgewexjzrsm6@brauner.io/>) that any system call using pidfds start with "`pidfd_`"; he would thus like this new call to be named `pidfd_madvise()`. That opinion is not universally shared, though, so it's not clear that the name will actually change. There were more substantive objections to `MADV_COOL` and `MADV_COLD`, but less consensus on what the new names should be. 

Hocko [questioned](<https://lwn.net/ml/linux-kernel/20190521062421.GD32329@dhcp22.suse.cz/>) the need for the multi-operation API, noting that `madvise()` operations are not normally expected (or needed) to be fast. Kim said he would come back with benchmark numbers to justify that API in a future posting. 

Of the three interfaces described here, `process_madvise()` (or whatever it ends up being named) seems like the most likely to proceed. There appears to be a clear need for the ability to have one process change how another process's memory is handled. All that is left is to hammer out the details of how it should actually work.
