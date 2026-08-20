---
title: Structure holes and information leaks
url: https://lwn.net/Articles/417989/
date: "December 1, 2010"
category: "Security-Vulnerabilities"
author: "By Jonathan Corbet December 1, 2010"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
December 1, 2010

Many of the kernel security vulnerabilities reported are information leaks \- passing the contents of uninitialized memory back to user space. These leaks are not normally seen to be severe problems, but the potential for trouble always exists. An attacker may be able to find a sequence of operations which puts useful information (a cryptographic key, perhaps) into a place where the kernel will leak it. So information leaks should be avoided, and they are routinely fixed when they are found. 

Many information leaks are caused by uninitialized structure members. It can be easy to forget to assign to all members in all paths, or, possibly, the form of the structure might change over time. One way to avoid that possibility is to use something like `memset()` to clear the entire structure at the outset. Kernel code uses `memset()` in many places, but there are places where that is seen as an expensive and unnecessary call; why clear a bunch of memory which will be assigned to anyway? 

One way of combining operations is with a structure initialization like: 

```
struct foo {
            int bar, baz;
        } f = {
        	.bar = 1,
        };
```

In this case, the `baz` field will be implicitly set to zero. This kind of declaration should ensure that there will be no information leaks involving this structure. Or maybe not. Consider this structure instead: 

```
struct holy_foo {
    	short bar;
    	long baz;
        };
```

On a 32-bit system, this structure likely contains a two-byte hole between the two members. It turns out that the C standard does not require the compiler to initialize holes; it also turns out that [GCC duly leaves them uninitialized](<https://lwn.net/Articles/417994/>). So, unless one knows that a given structure cannot have any holes on any relevant architecture, structure initializations are not a reliable way of avoiding uninitialized data. 

There has been some talk of asking the GCC developers to change their behavior and initialize holes, but, as Andrew Morton [pointed out](<https://lwn.net/Articles/417996/>), that would not help for at least the next five years, given that older compilers would still be in use. So it seems that there is no real alternative to `memset()` when initializing structures which will be passed to user space.
