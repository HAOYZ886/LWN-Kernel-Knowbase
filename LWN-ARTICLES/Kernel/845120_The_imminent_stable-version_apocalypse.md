---
title: "The imminent stable-version apocalypse"
url: https://lwn.net/Articles/845120/
date: "February 5, 2021"
category: "Development model-User-space ABI; Releases-Stable updates"
author: "By Jonathan Corbet February 5, 2021"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
February 5, 2021

As has often been pointed out, the stable-kernel releases are meant to be stable; that means they should be even more averse to ABI breaks than mainline releases, if that is possible. This may be a hard promise to keep for the next set of stable kernels, though, for the most mundane of reasons: nobody thought that there would be more than 255 minor updates to any given kernel release. 

For most of the existence of the kernel project, few developers within the project itself have maintained any given kernel release for more than a couple years or so, and maintenance releases were relatively rare. There were some exceptions; the 2.4 release happened at the beginning of 2001, and Willy Tarreau finally [stopped maintaining it](<https://lwn.net/Articles/491245/>) more than eleven years later. Even then, the final version was 2.4.37, though one could perhaps call it 2.4.48 after the final set of eleven small "fixup" releases. Releases for kernels maintained for the long term were relatively few and far apart. 

In recent years, though, that situation has changed, with some older kernels receiving much more long-term-maintenance attention. Thus, February 3 saw the release of the [4.9.255](<https://lwn.net/ml/linux-kernel/1612393450245231@kroah.com/>) and [4.4.255](<https://lwn.net/ml/linux-kernel/1612393445215192@kroah.com/>) updates. Those kernels have received 18,765 and 16,986 patches, respectively, and there is no sign of things slowing down. The current posted plan is to maintain 4.9 through January 2023 and 4.4 through February 2022. 

These kernel-release numbers are now a problem, as was [pointed out](<https://lwn.net/ml/linux-kernel/7pR0YCctzN9phpuEChlL7_SS6auHOM80bZBcGBTZPuMkc6XjKw7HUXf9vZUPi-IaV2gTtsRVXgywQbja8xpzjGRDGWJsVYSGQN5sNuX1yaQ=@protonmail.com/>) by Jari Ruusu. There are a couple of macros defined within the kernel relating to version codes; these can be found in `include/generated/uapi/linux/version.h` in a built kernel: 

```
#define LINUX_VERSION_CODE 330496
        #define KERNEL_VERSION(a,b,c) (((a) << 16) + ((b) << 8) + (c))
```

The first macro, `LINUX_VERSION_CODE`, is calculated in the top-level makefile; it is the result of: 

```
(5 << 16) + (11 << 8) + 0
```

That number (which is `0x50b00`) identifies this as a 5.11-rc kernel; it is the same result one gets from `KERNEL_VERSION(5,11,0)`. 

One does not have to look long to see that neither of these macros is going to generate the expected result once the minor version ("`c`" in the `KERNEL_VERSION()` macro) exceeds 255. Running that macro on a 4.9.255 kernel yields `0x409ff`, but on 4.9.256 it will instead return `0x40a00` — which looks like 4.10.0. That might just cause some confusion in the user community. 

This problem does not come as a complete surprise to the stable-kernel maintainers; Sasha Levin posted [this patch](<https://lwn.net/ml/linux-kernel/20210118014951.250815-1-sashal%40kernel.org/>) in mid-January in an attempt to fix it. It changes both `LINUX_VERSION_CODE` and `KERNEL_VERSION()` to use 16 bits for the minor version, thus eliminating the overflow. This patch got into linux-next, but seems unlikely to stay there; as Jiri Slaby [noted](<https://lwn.net/ml/linux-kernel/78ada91b-21ee-563f-9f75-3cbaeffafad4@kernel.org/>), these macros are used by user space and constitute a part of the kernel's ABI. He added that both the GNU C Library and the GCC compiler (the BPF code in particular) use the kernel version code in its current form and would not handle a change well. There are also many other places in the kernel that exchange these version codes with user space; see [this media `ioctl()` command](<https://www.kernel.org/doc/html/latest/userspace-api/media/v4l/vidioc-querycap.html#description>), for example. Changing the kernel's idea of how `KERNEL_VERSION()` works will break programs compiled with the older macro, which is not something that is allowed. 

So what is to be done? As of this writing that has not yet been worked out, but there are a couple of options on the table: 

  * Ruusu's note pointing out the problem suggested that stable releases could start incrementing the `EXTRAVERSION` field instead; this is the field that normally contains strings like `-rc7` (for mainline test releases), or a Git commit ID. The minor version would presumably remain at 255. This would avoid breaking ABI, but would also make it harder for user-space code to distinguish between stable releases after 255. It might also create minor trouble for distributors who are using that field to identify their own builds. 
  * Stable maintainer Greg Kroah-Hartman [suggested](<https://lwn.net/ml/linux-kernel/YBu1d0+nfbWGfMtj@kroah.com/>) that he could ""leave it alone and just see what happens"". But, as Slaby [pointed out](<https://lwn.net/ml/linux-kernel/a85b7749-38b2-8ce9-c15a-8acb9a54c5b5@kernel.org/>), that will create the wrapping problem described above, which could confuse user space. If this is done, he said, it would be necessary to mask the minor version to eight bits, causing it to wrap back around to zero; whether _that_ would cause confusion is another question. Version numbers are normally expected to increase monotonically. 

The most likely outcome can be seen in the kernel's history, though. Once upon a time, mainline kernel releases had three significant numbers rather than two — 2.6.30, for example. In those days, the minor version field wasn't available for stable updates, so the `EXTRAVERSION` field was used instead. Looking at the 2.6.30.3 makefile, one sees: 

```
VERSION = 2
        PATCHLEVEL = 6
        SUBLEVEL = 30
        EXTRAVERSION = .3
        NAME = Man-Eating Seals of Antiquity
```

That solution worked for years, so there should be no real reason why it wouldn't work now as well. Most likely `SUBLEVEL` would remain stuck at 255, with `EXTRAVERSION` indicating the real release number. 

It is evidently Leon Trotsky who once said that "old age is the most unexpected of all things that can happen to a man". Perhaps similar forces are at play here; running out of bits is the most unexpected of things that can happen to a kernel developer. This version-number overflow could have been foreseen some time ago, and the date of its occurrence forecast with reasonable certainty. But now some sort of solution has to be found before the next stable-kernel release can be made. Happily, the problem should be easier to resolve than that of old age. 

**Update** : Kroah-Hartman appears to have chosen the "do nothing" option with the release of [4.9.256](<https://lwn.net/Articles/845195/>) and [4.4.256](<https://lwn.net/Articles/845196/>), both of which increment the version number but make no other change. ""I'll try to hold off on doing a 'real' 4.9.y release for a week to give everyone a chance to test this out and get back to me. The pending patches in the 4.9.y queue are pretty serious, so I am loath to wait longer than that, consider yourself warned..."" 

**Update 2** : In the end, it [appears](<https://lwn.net/ml/linux-kernel/20210208145805.898658055@linuxfoundation.org/>) that the clamping solution will be taken, with the minor number fixed at 255 going forward.
