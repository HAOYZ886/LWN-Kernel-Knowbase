---
title: "Why glibc's fstat() is slow"
url: https://lwn.net/Articles/944214/
date: "September 14, 2023"
category: System calls
author: "By Jonathan Corbet September 14, 2023"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
September 14, 2023

The [`fstat()`](<https://man7.org/linux/man-pages/man2/stat.2.html>) system call retrieves some of the metadata — owner, size, protections, timestamps, and so on — associated with an open file descriptor. One might not think of it as a performance-critical system call, but there are workloads that make a lot of `fstat()` calls; it is not something that should be slowed unnecessarily. As it turns out, though, the GNU C Library (glibc) has been doing exactly that, but a fix is in the works. 

Mateusz Guzik has been working on a number of x86-related performance issues recently. As part of that work, he [stumbled into](<https://lwn.net/ml/linux-kernel/CAGudoHH95OKVgf0jW5pz_Nt2ab0HTnt3H9hbmU=aSHozOS5B0Q@mail.gmail.com/>) the realization that glibc's implementation of `fstat()` is expressed in terms of [`fstatat()`](<https://man7.org/linux/man-pages/man3/fstatat.3p.html>). Specifically, a call like: 

```
result = fstat(fd, &stat_buf);
```

is turned into: 

```
result = fstatat(fd, "", &stat_buf, AT_EMPTY_PATH);
```

These calls are semantically equivalent; by the POSIX definition, a call to `fstatat()` providing an empty string for the path and the `AT_EMPTY_PATH` flag operates directly on the provided file descriptor. But the difference in the kernel is significant; implementing `fstat()` in this way is significantly slower, for a couple of reasons. 

One of those is that `fstatat()` is a more complex system call, so it does preparatory work that is not useful for the simple `fstat()` case. Once alerted to the problem, Linus Torvalds [posted a patch](<https://lwn.net/ml/linux-kernel/CAHk-=wg6bzTdQHSsswHPYFUbb1DfszyWTZ97hZv7bYxaNHVkHw@mail.gmail.com/>) that detects this case and avoids that extra work. But the result is still, [according to Guzik's measurements](<https://lwn.net/ml/linux-kernel/20230903204858.lv7i3kqvw6eamhgz@f/>), about 12% slower than calling `fstat()` directly. 

That performance loss is the result of the second problem: `fstatat()` must check the provided path and ensure that it is empty. One might think that it makes no sense to even look at the path when the user has provided a flag (`AT_EMPTY_PATH`) that says there is nothing to be seen there but, as Al Viro [pointed out](<https://lwn.net/ml/linux-kernel/20230903232802.GO3390869@ZenIV/>), POSIX mandates this behavior. Checking the path means accessing user-space data from the kernel; that, in turn, can require disabling guardrails like [supervisor mode access prevention](<https://lwn.net/Articles/517475/>). It all adds up to a significant amount of overhead to check an empty string. 

Torvalds [made it clear](<https://lwn.net/ml/linux-kernel/CAHk-=wh+=W2k1V_0Om=_=QpPAN_VgHzdZ4FLXSfcyTSK7xo0Eg@mail.gmail.com/>) that he thought glibc's behavior made no sense and wondered why things were done that way. A bit later, though, he [found a plausible explanation](<https://lwn.net/ml/linux-kernel/CAHk-=wjrvn+J=z0_schGSROK0HCK-xs4wgky6pRKy7kVLhDeLg@mail.gmail.com/>) for this choice. On an x86-64 system, the kernel exports a number of related system calls, including `fstat()` (number 5) and `newfstatat()` (number 262). Torvalds concluded: 

> The glibc people found a "__NR_newfstatat", and thought it was a newer version of 'stat', and didn't find any new versions of the basic stat/fstat/lstat functions at all. So they thought that everything else was implemented using that 'newfstatat()' entrypoint. 
> 
> But on x86-64 (and most half-way newer architectures), the regular __NR_stat *is* that "new" stat. 

The "new" `fstat()`, after all, came about in the 0.97 release in 1992, so there was no reason for the x86-64 architecture (which arrived rather later than that) to use anything else. But, if Torvalds's explanation reflects reality, the glibc developers were fooled by the "new" part of the `newfstatat` name and passed over the entry point they should have used to implement `fstat()`. 

There are a few observations that one could make from this little bit of confusion: 

  * The system calls (and their names) provided at the kernel boundary are not the same as those expected by user-space programmers. The glibc people know this better than anybody else, since part of their job is to provide the glue between those two interfaces, but confusion still seems to happen. 
  * The fact that the kernel's documentation of the interface it presents to user space is ... mostly nonexistent ... certainly does not help prevent confusion of this type. 
  * Using qualifiers like "new" in the naming of functions, products, or one's offspring tends to be unwise; what is new today is old tomorrow. 

Be that as it may, even with Torvalds's change (which was [merged](<https://git.kernel.org/linus/9013c51c630a>) for the 6.6-rc1 release and will presumably show up in a near-term stable update), `fstat()` is slower than it needs to be when glibc is being used. In an attempt to improve the situation, Guzik [raised the issue](<https://lwn.net/ml/libc-alpha/CAGudoHG83bsjmsy9nvmqUGrSORRdnu0D8tQDRq=qm8+WWT00Eg@mail.gmail.com/>) on the libc-alpha list. Adhemerval Zanella Netto [responded](<https://lwn.net/ml/libc-alpha/680b330d-6ef3-adc5-9ba6-cf74dd53e422@linaro.org/>) that the library developers are trying to simplify their code by using the more generic system calls whenever possible, that the `AT_EMPTY_PATH` problem is likely to affect all of the `*at()` system calls, and that, as a consequence, the problem would be ""better fixed in the kernel"". 

Torvalds [pointed out](<https://lwn.net/ml/libc-alpha/CAHk-=wg_6CYB0qDQ9eDfjZSsqzgKcF2ZMd2DObfeFc_N-PxUjg@mail.gmail.com/>) that, while other system calls have to handle `AT_EMPTY_PATH`, `fstatat()` is the only one that is likely to matter from a performance perspective; none of the others should be expected to show up as problems in real-world programs. Meanwhile, despite the misgivings expressed previously, Zanella [put together a patch](<https://lwn.net/ml/libc-alpha/6d0e4e9e-ab69-0c73-bb9d-ce344b4a043b@linaro.org/>) causing glibc to use ordinary `fstat()` when appropriate. Torvalds [agreed that it looked correct](<https://lwn.net/ml/libc-alpha/CAHk-=wgkKHn9_aG8wTz8BkxtC2g=+93vN29HNVnC_s+cPCOteA@mail.gmail.com/>), but complained that the implementation was messy; he seemed to [prefer](<https://lwn.net/ml/libc-alpha/CAHk-=wgfHhF=kZtGmWKckrcbJuO=_gTa=EzMoTeirkRv-3ZR2Q@mail.gmail.com/>) an [alternative implementation](<https://lwn.net/ml/libc-alpha/b22fcbd5-eb69-7755-c76a-01706006b3cd@linaro.org/>) that Zanella posted later. 

As of this writing, neither version of the patch has found its way into the glibc repository; the latter version is [under consideration](<https://lwn.net/ml/libc-alpha/20230905203421.2127750-1-adhemerval.zanella@linaro.org/>). It is probably safe to assume that a version of this patch will be applied at some point; nobody has an interest in glibc being slower than it needs to be. This particular story has a happy ending, but it does stand as an example of what can happen in the absence of clarity around the interfaces between software components.
