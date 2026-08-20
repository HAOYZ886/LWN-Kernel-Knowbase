---
title: Finding kernel problems automatically
url: https://lwn.net/Articles/87538/
date: "June 1, 2004"
category: "  user; Debugging; Development tools-Sparse; sparse"
author: 
---

In past years, this page has looked at the work done by the "Stanford checker," which analyzes code in search of various types of programming errors. The checker has found a lot of problems over the years, with the result that a lot of problems have been fixed before they had a chance to bite users of production kernels. 

The only problem with the Stanford checker is that it is not free software; it is, in fact, completely unavailable to the world as a whole. Rather than release the code, the checker group went off and formed [Coverity](<http://coverity.com>) to commercialize the checker software (now called "SWAT" and touted, ominously, as being "patent pending"). Developers at Coverity still occasionally post reports of potential bugs found by SWAT, but, for the most part, their attention seems focused on potential revenue opportunities. 

It is hard to complain about this outcome. Before heading on this course, the Coverity folks uncovered vast numbers of bugs, and all Linux users benefited from that work. They also demonstrated how valuable static code testing tools can be. The community, however, was left in the position of having to actually write its own checker if it wanted one. Fortunately, this is the sort of thing the community can be good at. 

A while back, none other than Linus Torvalds started work on his own tool, which came to be called "sparse." There has recently been a flurry of new activity around sparse, so it seems like a good time to take a look. 

sparse is normally obtained by cloning the BitKeeper repository at `bk://kernel.bkbits.net/torvalds/sparse`. For those who don't use BK, a [checked-out version](<http://www.kernel.org/pub/software/devel/sparse/>) is available (as a bunch of SCCS files) on kernel.org. There is a low-bandwidth [sparse mailing list](<http://vger.kernel.org/vger-lists.html#linux-sparse>) as well. 

Essentially, sparse is a parsing and analysis library for the C language. One could put a number of different backends onto it; for example, a code-generation backend would turn it into a simple compiler. For the purposes of the kernel, however, the backend of interest is the analysis code which looks for various types of errors. The analyzer checks for quite a few different types of errors. Many of these (many sorts of type mismatches, for example) are also found by the compiler, but other tests are unique to sparse. 

The core test done by sparse is still the check for improper use of user-space pointers. A quick look through the kernel will turn up liberal use of a type attribute called `__user`; for example, the `read()` method invoked from system calls is prototyped as: 

```
ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);
```

When the kernel is being compiled, `__user` is defined as the empty string, so `gcc` doesn't see it at all. When sparse is being used, instead, it marks the pointer as (1) being in a separate address space, and (2) not being legal to dereference. sparse will use those flags to catch any mixing of user- and kernel-space pointers, and any attempt to directly dereference user-space pointers. 

These checks have turned up a surprising number of errors. The kernel normally sets up the virtual address space in such a way that direct dereferencing of user-space pointers actually works - most of the time. Using user-space addresses in this way will fail, however, if the user page is not actually resident in memory at the time. More importantly, perhaps, this sort of direct dereferencing bypasses the normal access controls; every such error could, thus, become a security hole. 

Catching such mistakes automatically seems like a good idea. It does require, however, that every variable holding a user-space pointer be marked with the `__user` attribute. Since much of the kernel (including every device driver) deals with user-space pointers, this is not a trivial job. This job is proceeding, however; several dozen patches adding `__user` annotations (and fixing problems found on the way) have been merged for 2.6.7. 

Other checks performed include finding constants which are overly long for their target type, mistakes in embedded assembly language code, empty `switch` statements, assignments in conditionals, and so on. Its output is rather noisy still, but one assumes that will improve over time. If you have sparse installed, running it on the kernel is simply a matter of adding "`C=1`" to the `make` command. External modules can also be checked in this way. 

sparse is still clearly far behind the Stanford checker in terms of the variety of errors it can find. Unlike the checker, however, sparse is free software. The core parsing infrastructure is in place, so the addition of new checks should be relatively straightforward. All that's needed is the application of a bunch of developer time.
