---
title: fincore()
url: https://lwn.net/Articles/371538/
date: "January 27, 2010"
category: "Prefetching; System calls-fincore"
author: "By Jonathan Corbet January 27, 2010"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
January 27, 2010

Linux has long had the `mincore()` system call which allows an application to determine whether a given page is in RAM or not. There is no easy way, though, to tell whether a given page from a file is in the page cache or not. An application can `mmap()` the file and use `mincore()` on it, but that can be slow. So Chris Frost has proposed a new `fincore()` system call to handle this task: 

```
int fincore(int fd, loff_t start, loff_t len, unsigned char *vec);
```

A call to `fincore()` will look at the pages of the file associated with `fd` in the range indicated by `start` and `len`. For each page of the file, one byte of `vec` will be set to a non-zero value if that page is in memory. Naturally, this answer is an approximation - the situation can change while the system call is running. 

That, however, can be good enough for Chris's needs. His objective is to speed up applications which perform large numbers of non-sequential file reads. The traditional readahead code deals poorly with this kind of application, since the access pattern cannot be predicted ahead of time. But the application often _does_ know about a sequence of reads in advance; if the kernel could be told to pull in those pages ahead of time, it could order the I/O operations optimally and make the whole thing go faster. When doing this for sqlite and the GIMP, Chris reports significant speedups. 

The `fadvise()` system call can be used to request prefetching of file data. But there's a problem: it's hard for a prefetch library to know how much system memory is available. If too little data is prefetched, the performance gains will not be what they could be. Prefetching too much data, however, can lead to thrashing. Hence the `fincore()` system call: if prefetched pages are no longer present by the time the application gets around to using them, the library knows that it is asking for too much and can back off. 

Andrew Morton [likes the patch](<https://lwn.net/Articles/371540/>): 

I must say, the syscall appeals to my inner geek. Lot of applications are leaving a lot of time on the floor due to bad disk access patterns. A really smart library which uses this facility could help all over the place. 

Jamie Lokier, though, [wondered](<https://lwn.net/Articles/371541/>) if it might not be a better idea to find a way to inform applications more directly that their pages are being evicted prior to use. 

This is the first posting for this system call, so it has not gotten a lot of attention yet; more discussion will certainly be necessary before it could be merged. In the mean time, the [libprefetch](<http://libprefetch.cs.ucla.edu/>) site has more information on this whole project.
