---
title: Using sparse for endianness verification
url: https://lwn.net/Articles/205624/
date: "October 25, 2006"
category: "Development tools-Sparse; sparse"
author: 
---

Developers like to joke about Al Viro's fearsome presence on linux-kernel, but the truth of the matter is that he has been relatively quiet there for some time. That does not mean, however, that he has become a full-time Plan 9 developer. Instead, he has been steadily working to improve the static analysis tools used to find kernel bugs before they bite users. 

In recent times, Al's work has resulted in a long series of patches merged into the mainline, almost all of which have been marked as "endianness annotations." These patches mostly change the declared types for various functions, variables, and structure members. The new types may be unfamiliar to many, since they are relatively new - though not _that_ new; they were introduced in 2.6.9. These types are `__le16`, `__le32`, `__le64`, `__be16`, `__be32`, and `__be64`. 

What these types represent is an attempt to encode whether the (unsigned) integer value is big-endian (most significant byte first) or little-endian. For most programming, even within the kernel, endianness is not a concern; things just work without much thought on the programmer's part. Kernel code often must work with data encoded in a specific byte ordering which might not match the processor's ordering, though. Network protocols, filesystem on-disk data structures, and device registers are all examples. In general, when the kernel works with data in a non-native ordering, it must first swap the bytes around to match the processor's expectations. Failure to do so can lead to all kinds of strange bugs. 

There are a number of macros provided which can help with this task. There are classic functions like `htonl()`, which converts a 32-bit integer from host to "network" (big-endian) order. More generally, the kernel provides macros like `__le32_to_cpu()`, which will convert a little-endian 32-bit quantity to the ordering required by the processor. These macros make for portable code; they perform the requested transformation on systems where it is needed, and simply vanish in the remaining cases. 

The conversion functions only work, however, when the programmer remembers to use them. In their absence, values in non-native byte orders simply look like integers, and there is no way to catch the error until something blows up. And that might not happen to the original developer at all; the code may work flawlessly until somebody tries to run it on a different architecture and things fall apart. 

It would be nice to catch endianness mistakes at an earlier stage. That is the purpose of types like `__be32`; they allow a programmer to mark data with a specific ordering when it first enters the system. Thereafter, a suitably smart tool can check the code which manipulates that data and ensure that it does not mix that data with native-order data, does not try to do arithmetic with it, etc. Once everything is suitably annotated, whole classes of bugs can be caught before the kernel is even booted. And that can only be a good thing. 

The "suitably smart tool" which does this work is "sparse," a static checker which was originally written by Linus Torvalds. There is support for sparse built into the kernel build system, making it easy to check code for errors. The one thing which remains relatively difficult, for whatever reason, is getting the "sparse" tool in the first place. Few distributors package it, so prospective users must grab a copy and build it themselves. 

The true source for sparse is the git repository on kernel.org. With git, it's a simple matter of of running: 

```
git clone  git://git.kernel.org/pub/scm/devel/sparse/sparse.git
```

A simple "make" in the resulting directory will yield a working `sparse` binary. This tool changes quickly enough that updating from the repository on a regular basis is probably a good idea. For people who don't have git handy, it is also possible to grab a tarball snapshot from [Dave Jones's site](<http://www.codemonkey.org.uk/projects/git-snapshots/sparse/>). 

Once `sparse` is installed, running it on the kernel is a simple matter of going to your local source tree and running: 

```
make C=2
```

The parameter `C=2` causes `sparse` to be run on every `.c` file; if `C=1` is used instead, only files which must be recompiled are checked. Checking for endianness problems requires an additional parameter: 

```
make C=2 CF=-D__CHECK_ENDIAN__
```

The number of warnings which result from this command can be large - though it is dropping as Al works his way through the code. 

Checking code submissions with `sparse` is highly recommended - it is one of the steps in the [patch submission checklist](<https://lwn.net/Articles/205636/>) packaged with the kernel. Use of `sparse` may still be more of an exception than the rule, however. But it is easy enough - and useful enough - that there really is no reason not to run the checker on code before sending it out. It is, after all, much nicer to have the computer find silly mistakes for you, in the privacy of your own computer, before broadcasting them to the world.
