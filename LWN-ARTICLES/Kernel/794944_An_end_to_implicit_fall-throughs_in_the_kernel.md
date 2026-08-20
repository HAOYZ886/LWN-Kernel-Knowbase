---
title: "An end to implicit fall-throughs in the kernel"
url: https://lwn.net/Articles/794944/
date: "August 1, 2019"
category: "Coding style; Security-Kernel hardening"
author: "By Jonathan Corbet August 1, 2019"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
August 1, 2019

The C `switch` statement has, since the beginning of the language, required the use of explicit `break` statements to prevent execution from falling through from one `case` to the next. This behavior can be a useful feature, allowing for more compact code, but it can also lead to bugs. The effort to rid the kernel of implicit fall-through coding patterns came to a conclusion with the 5.3-rc2 release, where the last cases were fixed. There is a good chance that these fixes will have to be redone in the future, though. 

The problem with C's fall-through behavior is that it is implicit, with no indication of whether the behavior is intended or not. Developers learn (the hard way, sometimes) to end each case with a `break` statement as a matter of habit, but it's still an easy thing to forget, and the resulting code is seen by the compiler as being entirely valid. A forgotten `break` almost certainly introduces a bug, even if it might not manifest itself for years. Many developers have had reason to wish that the C language required an explicit indication by the programmer that fall-through behavior is desired. 

#### Making fall-through behavior explicit

The GCC compiler has, for some time, offered a warning option (`-Wimplicit-fallthrough`) intended to catch missing `break` statements. It will trigger for any `case` that falls through into a succeeding `case` unless an explicit marker has been placed to indicate that the behavior is correct. It turns out that that there are many ways of placing this marker, depending on the value provided with `-Wimplicit-fallthrough`; the full set can be found in [the GCC documentation](<https://gcc.gnu.org/onlinedocs/gcc/Warning-Options.html#index-Wimplicit-fallthrough>). At its most lax, the presence of any comment at all prior to a `case` statement will allow warning-free fall-through into that case. Using `-Wimplicit-fallthrough=3` causes the compiler to look for comments matching a fairly large set of regular expressions. 

In the kernel, one is most likely to find comments that look like: 

> Style| Count  
> ---|---  
> `/* fallthrough */`| 350  
> `/* fall through */`| 3,091  
> `/* falls through */`| 8  
> `/* fall-through */`| 167  
  
Note that case was ignored in generating the above numbers, and that there are probably other variants in the code as well. All of those comment styles will be recognized by GCC with `-Wimplicit-fallthrough=3` and will thus cause warnings to be suppressed. 

As can be seen, there are a lot of these comments in the kernel source now, some of which date back to before the beginning of the Git era. Their number has increased in recent times, though, due mostly to a focused effort by Gustavo A. R. Silva, who has been working to eliminate every implicit fall-through warning in the kernel. The point of this effort, beyond fixing any bugs caused by missing `break` statements, is to make it possible to enable `-Wimplicit-fallthrough=3` in the kernel build by default. As long as that option generates a pile of warnings, developers will ignore them and nobody will notice when new ones are added. Once the kernel is warning-free, though, it should be possible to keep it that way. 

This work, which was supported by the Linux Foundation, has resulted in over 400 patches since 2017. As Silva has worked through the kernel source eliminating fall-through warnings, he has encountered (and fixed) a number of bugs. A quick search turns up the following commits, for example: 

> [`d64062b57eeb`](<https://git.kernel.org/linus/d64062b57eeb>)| drm/amdgpu/gfx10: Fix missing break in switch statement  
> ---|---  
> [`737298d18836`](<https://git.kernel.org/linus/737298d18836>)| drm/amdkfd: Fix missing break in switch statement  
> [`60747828eac2`](<https://git.kernel.org/linus/60747828eac2>)| net: socket: Fix missing break in switch statement  
> [`84242b82d81c`](<https://git.kernel.org/linus/84242b82d81c>)| rtlwifi: rtl8723ae: Fix missing break in switch statement  
> [`7850b51b6c21`](<https://git.kernel.org/linus/7850b51b6c21>)| scsi: mpt3sas: Add missing breaks in switch statements  
> [`5e420fe63581`](<https://git.kernel.org/linus/5e420fe63581>)| scsi: aacraid: Fix missing break in switch statement  
> [`09186e503486`](<https://git.kernel.org/linus/09186e503486>)| security: mark expected switch fall-throughs and add a missing break  
> [`b5be853181a8`](<https://git.kernel.org/linus/b5be853181a8>)| crypto: ccree - fix missing break in switch statement  
> [`7264235ee74f`](<https://git.kernel.org/linus/7264235ee74f>)| IB/hfi1: Add missing break in switch statement  
> [`cc5034a5d293`](<https://git.kernel.org/linus/cc5034a5d293>)| drm/radeon/evergreen_cs: fix missing break in switch statement  
> [`479826cc8611`](<https://git.kernel.org/linus/479826cc8611>)| staging: comedi: ni_660x: fix missing break in switch statement  
> [`5340f23df8fe`](<https://git.kernel.org/linus/5340f23df8fe>)| gpio: sprd: Add missing break in switch statement  
> [`df997abeebad`](<https://git.kernel.org/linus/df997abeebad>)| iscsi_ibft: Fix missing break in switch statement  
> [`2f10d8237396`](<https://git.kernel.org/linus/2f10d8237396>)| drm/amd/powerplay: Fix missing break in switch  
> [`307b00c5e695`](<https://git.kernel.org/linus/307b00c5e695>)| rtl8xxxu: Fix missing break in switch  
> [`5d25ff7a5448`](<https://git.kernel.org/linus/5d25ff7a5448>)| scsi: ips: fix missing break in switch  
> [`a7ed5b3e7dca`](<https://git.kernel.org/linus/a7ed5b3e7dca>)| staging: comedi: tio: fix multiple missing break in switch bugs  
> [`c24bfa8f21b5`](<https://git.kernel.org/linus/c24bfa8f21b5>)| spi: slave: Fix missing break in switch  
> [`ad0eaee6195d`](<https://git.kernel.org/linus/ad0eaee6195d>)| ASoC: wm8994: Fix missing break in switch  
> [`9ba8376ce1e2`](<https://git.kernel.org/linus/9ba8376ce1e2>)| ptp: fix missing break in switch  
> [`4e57562b4846`](<https://git.kernel.org/linus/4e57562b4846>)| iio: imu: inv_mpu6050: fix missing break in switch  
  
The above list is clearly incomplete, though, since a number of the other fixes do not explicitly mention missing `break` statements in the subject line; see [commit `ed4ed4043a12`](<https://git.kernel.org/linus/ed4ed4043a12>), for example. 

On July 27, a milestone of sorts was hit with [this pull](<https://git.kernel.org/linus/88c508344245>) into the mainline from Silva's repository: the final implicit fall-through warning has been fixed, and `-Wimplicit-fallthrough=3` [is now the default](<https://git.kernel.org/linus/a035d552a93b>) for kernel builds. In other words, the kernel's coding style has just been tweaked to disallow the use of implicit fall-through in `switch` statements. It must be a satisfying conclusion to a long project. 

#### Moving beyond comments

Naturally, though, some developers are not entirely happy with this change. Nobody who actually works in the kernel community seems to disagree with the idea of eliminating implicit fall-throughs, but the mechanism used to do so does not sit well with everybody; as Peter Zijlstra [put it](<https://lwn.net/ml/linux-kernel/20190624193123.GI3436@hirez.programming.kicks-ass.net/>) back in June: ""I still consider it an abomination that the C parser looks at comments"". Zijlstra, like others, would prefer that there were another way. 

As it happens, there _is_ another way. GCC starting with version 7 allows the use of a special attribute on a statement instead: 

```
__attribute__((fallthrough))
```

The use of this attribute can perhaps made more visually pleasing with a definition like: 

```
#define __fallthrough __attribute__((fallthrough))
```

The `__fallthrough` symbol could then be used instead of the magic comments to eliminate implicit fall-through warnings. A [patch set](<https://lwn.net/ml/linux-kernel/20181021171414.22674-1-miguel.ojeda.sandonis@gmail.com/>) implementing this option was posted by Miguel Ojeda in October 2018; it generated a fair amount of interest but was not adopted for a couple of reasons: it doesn't work with older versions of GCC, and the LLVM Clang compiler does not implement that attribute at all. Since Clang does successfully build the kernel in some settings (and is evidently used for the Android build), breaking it is not an appealing option. 

The good news is that the Clang developers appear to be working on implementing this attribute. They also, it seems, have an implementation aimed at the proposed next standard for the C language, currently called C2X. The bad news is that the syntax there is different: 

```
[[fallthrough]]
```

If this version becomes an actual part of the C standard, it would probably be the preferable one to use going forward. But that may not be fully resolved for some time. In any case, the actual mechanism used to mark explicit fall-through behavior can be hidden behind a `#define` as shown above. 

The one thing that _can't_ be done is: 

```
#define __fallthrough /* fall through */
```

since the preprocessor will simply delete the comment rather than substituting it into the source. So the comments have to remain for now. But one can imagine that, at some point in the future when a better alternative is available, those thousands of comments are likely to be replaced with something like `__fallthrough` in a set of massive, tree-wide patches. 

Meanwhile, though, the kernel has finally reached a point where there are no `switch` statements with implicit fall-through behavior, and the build process has been amended to prevent such behavior from being added in the future. One source of kernel bugs has been closed, a switch that can only be seen as a best-case scenario.
