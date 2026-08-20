---
title: kzalloc()
url: https://lwn.net/Articles/147014/
date: "August 10, 2005"
category: kzalloc
author: 
---

The kernel code base is full of functions which allocate memory with `kmalloc()`, then zero it with `memset()`. Recently, Pekka Enberg concluded that much of this code could be cleaned up by using `kcalloc()` instead. `kcalloc()` has this prototype: 

```
void *kcalloc(size_t n, size_t size, unsigned int __nocast gfp_flags);
```

This function will allocate an array of `n` items, and will zero the entire array before returning it to the caller. Pekka's patch converted a number of `kmalloc()`/`memset()` pairs over to `kcalloc()`, but that patch drew [a complaint](<https://lwn.net/Articles/147016/>) from Andrew Morton: 

Notice how every conversion you did passes in `1' in the first argument? And that's going to happen again and again and again. Each callsite needlessly passing that silly third argument, adding more kernel text. 

Very few callers actually need to allocate an array of items, so the extra argument is unneeded in most cases. Each instance of that argument adds a bit to the size of the kernel, and, over time, that space adds up. The solution was to create yet another allocation function: 

```
void *kzalloc(size_t size, unsigned int __nocast gfp_flags);
```

This function returns a single, zeroed item. It has been added to -mm, with its appearance in the mainline likely to happen for 2.6.14.
