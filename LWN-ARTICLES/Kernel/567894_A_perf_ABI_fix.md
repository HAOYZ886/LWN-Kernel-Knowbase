---
title: A perf ABI fix
url: https://lwn.net/Articles/567894/
date: "September 24, 2013"
category: "Development model-User-space ABI; Performance monitoring"
author: "By Jonathan Corbet September 24, 2013"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
September 24, 2013

It is often said that the kernel developers are committed to avoiding ABI breaks at almost any cost. But ABI problems can, at times, be hard to avoid. Some have argued that the perf events interface is particularly subject to incompatible ABI changes because the `perf` tool is part of the kernel tree itself; since `perf` can evolve with the kernel, there is a possibility that developers might not even notice a break. So the recent discovery of a perf ABI issue is worth looking at as an example of how compatibility problems are handled in that code. 

The `perf_event_open()` system call returns a file descriptor that, among other things, may be used to map a ring buffer into a process's address space with `mmap()`. The first page of that buffer contains various bits of housekeeping information represented by `struct perf_event_mmap_page`, defined in `<uapi/linux/perf_event.h>`. Within that structure (in a 3.11 kernel) one finds this bit of code: 

```
union {
    	__u64	capabilities;
    	__u64	cap_usr_time  : 1,
    		cap_usr_rdpmc : 1,
    		cap_____res   : 62;
        };
```

For the curious, `cap_usr_rdpmc` indicates that the `RDPMC` instruction (which reads the performance monitoring counters directly) is available to user-space code, while `cap_usr_time` indicates that the time stamp counter can be read with `RDTSC`. When these features (described as "capabilities," though they have nothing to do with the security-oriented capabilities implemented by the kernel) are available, code which is monitoring itself can eliminate the kernel middleman and get performance data more efficiently. 

The intent of the above union declaration is clear enough: the developers wanted to be able to deal with the full set of capabilities as a single quantity, or to be able to access the bits individually via the `cap_` fields. One need not look at it for too long, though, to see the error: each of the `cap_` fields is a separate member of the enclosing union, so they will all map to the same bit. This interface, thus, has never worked as intended. But, in a testament to the thoroughness of our code review, it was [merged for 3.4](<http://git.kernel.org/linus/c7206205d00ab375839bd6c7ddb247d600693c09>) and persisted through the 3.11 release. 

Once the problem was noticed, Adrian Hunter quickly posted [the obvious fix](<https://lwn.net/Articles/567925/>), grouping the `cap_` fields into a separate structure. But it didn't take long for Vince Weaver to [find a new problem](<https://lwn.net/Articles/567926/>): code that worked with the broken structure definition no longer does with the fixed version. The fix moved `cap_usr_rdpmc` from bit 0 to bit 1 (while leaving `cap_usr_time` in bit 0), with the result that binaries built for older kernels look for it in the wrong place. If a program is, instead, built with the newer definition, then run on an older kernel, it will, once again, look in the wrong place and come to the wrong conclusion. 

After some discussion, it became clear that it would not be possible to fix this problem in an entirely transparent way or to hide the fix from newer code. At that point, Peter Zijlstra [suggested](<https://lwn.net/Articles/567928/>) that a version number field be used; applications could explicitly check the ABI version and react accordingly. But Ingo Molnar [rejected](<https://lwn.net/Articles/567929/>) that approach as ""really fragile"" and came up with a fix of his own. After a few rounds of discussion, the union [came to look like this](<https://lwn.net/Articles/568139/>): 

```
union {
     	__u64	capabilities;
     	struct {
    	    __u64 cap_bit0			: 1,
    	    	  cap_bit0_is_deprecated	: 1, 
    	    	  cap_user_rdpmc		: 1,
    	    	  cap_user_time			: 1,
    	    	  cap_user_time_zero		: 1,
    	    	  cap_____res			: 59;
     	};
         };
```

In the new ABI, `cap_bit0` is always zero, while `cap_bit0_is_deprecated` is always one. So code that is aware of the shift can test `cap_bit0_is_deprecated` to determine which version of the interface it is using; if it detects a newer kernel, it will know that the various `cap_user_` (changed from `cap_usr_`) fields are valid and can be used. Code built for older kernels will, instead, see all of the old capability bits (both of which mapped onto bit 0) as being set to zero. (For the curious, the new `cap_user_time_zero` field was added in [an independent 3.12 change](<http://git.kernel.org/linus/c73deb6aecda2955716f31572516f09d930ef450>)). 

One could argue that this change still constitutes an ABI break, in that older code may conclude that `RDPMC` is unavailable when it is, in fact, supported by the system it is running on. Such code will not perform as well as it would have with an older kernel. But it will perform correctly, which is the biggest concern here. More annoying to some might be the fact that code written for one version of the interface will fail to compile with the other; it is an _API_ break, even if the ABI continues to work. This will doubtless be irritating for some users or packagers, but it was seen as being better than continuing to allow code to use an interface that was known to be broken. Vince Weaver, who has sometimes been critical of how the perf ABI is managed, [conceded](<https://lwn.net/Articles/567931/>) that ""this seems to be about as reasonable a solution to this problem as we can get"". 

One other important aspect to this change is the fact that the structure itself describes which interpretation should be given to the capability bits. It can be tempting to just make the change and document somewhere that, as of 3.12, code must use the new bits. But that kind of check is easy for developers to overlook or forget, even in this simple situation. If the fix is backported into stable kernels, though, then simple kernel version number checks are no longer good enough. With the special `cap_bit0_is_deprecated` bit, code can figure out the right thing to do regardless of which kernel the fix appears in. 

In the end, it would be hard to complain that the perf developers have failed to respond to ABI concerns in this situation. There will be an API shift in 3.12 (assuming Ingo's patch is merged, which had not happened as of this writing), but all combinations of newer and older kernels and applications will continue to work; this ABI break went in during the 3.12 merge window, but never found its way into a stable kernel release. The key there is early testing; by catching this issue at the beginning of the development cycle, Vince helped to ensure that it would be fixed by the time the stable release happened. The kernel developers do not want to create ABI problems, but extensive user testing of development kernels is a crucial part of the process that keeps ABI breaks from happening.
