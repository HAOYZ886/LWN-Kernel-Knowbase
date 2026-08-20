---
title: "Briefly: patch quality, CKRM, likely(), and vmsplice()"
url: https://lwn.net/Articles/182369/
date: "May 3, 2006"
category: "Class-based resource management; Development model-Kernel quality; System calls-vmsplice; likely"
author: 
---

A number of issues have been discussed in recent times that, while too short for a full article, are nonetheless worthy of mention. Here's a few of them. 

### Development process

The [2.6.17-rc2-mm1](<http://lwn.net/Articles/181696/>) release included, along with the usual huge pile of patches, a complaint from Andrew Morton: 

It took six hours work to get this release building and linking in just a basic fashion on eight-odd architectures. It's getting out of control.... 

Could patch submitters _please_ be a lot more careful about getting the Kconfig correct, testing various Kconfig combinations (yes sometimes people will want to disable your lovely new feature) and just generally think about these things a bit harder? It isn't rocket science. 

Andrew, it seems, is getting too many submissions which lack basic testing. Occasionally things simply don't compile. More often, patches create problems when their particular configuration options are disabled, or for architectures not tested by the original developer. Andrew ends up fixing those problems, and that takes a fair amount of his time. The [bigger issue](<https://lwn.net/Articles/182370/>) is elsewhere, however: 

My main reason for the big whine is that this defect rate indicates that people just aren't being sufficiently careful in their work. If so many silly trivial things are slipping through, then what does this tell us about the big things, ie: runtime bugs? 

There has been some discussion of how the situation could be improved. Ideas include better automated kernel build farms which would allow any developer to get wider build testing and [a checklist](<https://lwn.net/Articles/182371/>) to be gone over before patches are sent for review. But what is really needed is for developers to simply take a little more care in the preparation of their patches. 

### CKRM rebranded

The CKRM resource management patches have been [received unenthusiatically](<http://lwn.net/Articles/145135/>) by the development community in the past. To many, CKRM looks like a large body of complex code, with hooks distributed throughout the kernel, providing functionality which is of interest to relatively few users. So the CKRM proposals have not gotten very far, and the development team has been quiet recently. 

What the developers have been doing, however, is reworking the CKRM patches in an attempt to make them more palatable. The result is now known as [Resource Groups](<http://lwn.net/Articles/181857/>), and it is, once again, being pushed for inclusion into the kernel. The Resource Group code has been put on a diet, with many features removed and others shoved out to user space. Duplicated code has been taken out, and a major effort has been made to use kernel library primitives wherever possible. 

Andrew Morton had [a reasonable positive reaction](<https://lwn.net/Articles/182376/>) to the new code submission, saying ""...the overall code quality is probably the best I've seen for an initial submission of this magnitude"". He was [more worried](<https://lwn.net/Articles/182378/>) about a proposed memory controller, however, which looks to duplicate much of the memory management subsystem. There have not been a whole lot of comments from elsewhere in the community, however. 

### Not so unlikely after all

The kernel provides a couple of macros, called `likely()` and `unlikely()`, which are intended to provide hints to the compiler regarding which way a test in an `if` statement might go. The processor can then use that hint, at run time, to direct its branch prediction and speculative execution optimizations. These macros are used fairly heavily throughout the kernel to reflect what the programmer thinks will happen. 

A well-known fact of life is that programmers can have a very hard time guessing which parts of their code will actually consume the most processor time. It turns out that they aren't always very good at choosing the likely branches in their code either. To drive this point home, Daniel Walker has put together [a patch](<http://lwn.net/Articles/181694/>) which does a run-time profile of `likely()` and `unlikely()` declarations. With the resulting output, it is possible to see which of those declarations are, in reality, incorrect and slowing down the kernel. 

Using this output, Hua Zhong and others have been writing patches to fix the worst offenders; some of them have already found their way into the mainline. In at least one case, the results have made it clear to the developers that things are not working as they were expected to, and other fixes are in the works. 

One `unlikely()` which remains unfixed, however, is in `kfree()`. Passing a `NULL` pointer to `kfree()` is entirely legal, and there has been a long series of janitorial patches removing tests which checked pointers for `NULL` before freeing them. `kfree()` itself is coded with a hint that a `NULL` pointer is unlikely, but it turns out that, in real life, [over half of the calls to `kfree()` pass `NULL` pointers](<https://lwn.net/Articles/182398/>). There is resistance to changing the hint, however; the preference seems to be to fix the (assumed) small number of high-bandwidth callers which are at the root of the problem. 

### vmsplice()

Last week, your editor astutely caught the last-minute merging of the [`vmsplice()` system call](<http://lwn.net/Articles/181169/>) into 2.6.17-rc3. Rather less astutely, however, your editor missed the fact that the prototype for `vmsplice()` had changed since it was posted on the linux-kernel mailing list. The current prototype for `vmsplice()` is: 

```
long vmsplice(int fd, const struct iovec *iov, 
                      unsigned long nr_segs, unsigned int flags);
```

The use of the `iovec` structure allows `vmsplice()` to be used for scatter/gather operations. 

Since then, `vmsplice()` has picked up a new flag: `SPLICE_F_GIFT`. If that flag is set, the calling process is offering the pages to the kernel as a "gift." If conditions allow, the kernel can simply remove the page from the process's address space and dump it into, for example, the page cache. With this flag, an application can generate data in memory, then send it on to its destination without copying in the kernel.
