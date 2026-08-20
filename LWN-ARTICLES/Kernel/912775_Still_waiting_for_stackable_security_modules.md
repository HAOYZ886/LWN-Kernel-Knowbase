---
title: Still waiting for stackable security modules
url: https://lwn.net/Articles/912775/
date: "October 31, 2022"
category: "Security-Security modules"
author: "By Jonathan Corbet October 31, 2022"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
October 31, 2022

The Linux security module (LSM) mechanism was created as a result of the [first Kernel Summit](</2001/features/KernelSummit/>) in 2001; it was designed to allow the development of multiple approaches to Linux security. That goal has been met; there are several security modules available with current kernels. The LSM subsystem was not designed, though, to allow multiple security modules to work together on the same system. Developers have been working to rectify that problem almost since the LSM subsystem was merged, but with limited success; some small security modules can be stacked on top of the "major" ones, but arbitrary stacking is not possible. Now, a full 20 years after security-module support went into the 2.5 development kernel series, it looks like a solution to the stacking problem may finally be getting closer. 

#### The challenge

The early thinking was that an LSM would enforce a security policy on the entire system, and that there would be only one of them. The fact that the only existing LSM for several years was SELinux helped to reinforce that belief, but developers quickly realized that there could be good reasons to run multiple LSMs on a system. A proper stacking scheme would, for example, make it possible to use a variety of small LSMs, each of which is aimed at a piece of the security policy. More recent developments, such as containers, have increased the number of settings where even having multiple full-system modules loaded might make sense. 

There has been no shortage of attempts to solve this problem. Some of those that were covered here over the years include: 

  * Serge Hallyn may have made [the first attempt](<https://lwn.net/Articles/110432/>) in 2004. 
  * [David Howells](<https://lwn.net/Articles/426921/>) in 2011. 
  * [Casey Schaufler](<https://lwn.net/Articles/518345/>) in 2012. 
  * [Schaufler again](<https://lwn.net/Articles/635771/>) in 2015. 
  * [Yet again](<https://lwn.net/Articles/804906/>) in 2019. 

Anybody who wants to solve this particular problem is going to have to face a number of challenges. One of those is deciding whether to allow an operation if there are multiple active LSMs and they disagree with each other. The simplest approach there is to give any LSM veto power; all modules that express an opinion on any specific operation must agree to allow it, or it will be denied. The hardest problems may well be elsewhere. Figuring out what the user-space interfaces should look like when multiple LSMs are active is not straightforward; tracking down policy problems can be painful even when there is only one module in the mix. 

Another significant problem is giving LSMs the means to attach their own metadata to objects in the system. The original LSM patches handled this by adding pointers to various kernel data structures, but no provision was made for the problem of multiple modules needing to store data. Any solution has to allow LSMs to cooperate in this regard as well while, at the same time, not having a measurable effect on performance. 

#### A viable solution?

Schaufler does not lack for persistence; ten years after starting on this project, he is still trying to get a solution for security-module stacking that addresses these problems into the mainline kernel. [Version 38](<https://lwn.net/ml/linux-security-module/20220927195421.14713-1-casey%40schaufler-ca.com/>) of his stacking patch set was posted in late September; it does not solve the entire problem, but it does make it possible to stack the AppArmor LSM with any other module. After all those years and versions, it might not be surprising to learn that Schaufler is ready to see this work merged; back in August, he [asked](<https://lwn.net/ml/linux-security-module/791e13b5-bebd-12fc-53de-e9a86df23836@schaufler-ca.com/>) whether that could happen during the 6.1 kernel cycle: 

> I would like very much to get v38 or v39 of the LSM stacking for Apparmor patch set in the LSM next branch for 6.1. The audit changes have polished up nicely and I believe that all comments on the integrity code have been addressed. The interface_lsm mechanism has been beaten to a frothy peak. 

This plan was complicated by an independent event, though: longtime LSM maintainer James Morris [stepped aside](<https://lwn.net/ml/linux-security-module/165731248606.335227.15503387932450699761.stgit@olly/>) and Paul Moore took over the maintainership of that subsystem. This change arguably had both positive and negative effects with regard to the stacking patches. On the positive side, Moore appears to have more time to engage with the stacking patch set and a stronger desire to see it merged into the mainline. Less positive, at least with regard to a quick merging of the patches, is that Moore [felt the need](<https://lwn.net/ml/linux-security-module/CAHC9VhSF8hWg=7tbFiCrizNF61vpwJcU3793LcStiu-anW4i1g@mail.gmail.com/>) to re-review the patch set from the beginning, which inevitably led to comments and requests for changes. 

Specifically, Moore was unhappy with the user-space API, which is an extension of the existing, `/proc`-based interface that even Schaufler described as ""hideous"". Moore [suggested](<https://lwn.net/ml/linux-security-module/CAHC9VhTDGwO789t59EyOV0SwnwGrdyBhRiuJpoY7cB4MSe02BQ@mail.gmail.com/>) that perhaps the time had come to add a set of LSM-specific system calls instead: 

> We have avoided this in the past for several reasons, but over the past couple of decades the LSMs have established themselves as a core part of Linux with many (all?) major Linux distributions shipping and supporting at least one LSM; I think we can justify a handful of well designed syscalls, and with Landlock we have some precedence too. 

Moore laid out a rough design for the system-call API that he had in mind as well. Schaufler was [less than pleased](<https://lwn.net/ml/linux-security-module/e2b6ae44-1037-666f-5012-6abd4d46c0b7@schaufler-ca.com/>) with this idea, though: 

> I wish you'd suggested this three years ago, when I could have done something with it. If stacking has to go on a two year redesign because of this it is dead. We've spent years polishing the /proc interfaces. Changed the names, the content, even bent over backwards to accommodate the security module that refused to adopt an attr/subdir strategy. 

User-space interfaces can be exceedingly difficult to change once they have been included in a kernel release; if significant changes are required, they usually need to happen before the code is merged. So it is not entirely surprising that Moore [was insistent](<https://lwn.net/ml/linux-security-module/CAHC9VhQ+UcJw4G=VHNE8wMa+EBG-UcoZ7ox0vNqLHoSKAd9XZQ@mail.gmail.com/>), saying that he could not accept the proposed interface; Schaufler eventually [threw in the towel](<https://lwn.net/ml/linux-security-module/9f79faa7-0d6a-c023-4561-15a261d4c60d@schaufler-ca.com/>) and started discussing what he needed to do: 

> OK, so what interfaces need to be redone? I have been polishing what's just become a turd for a %^&*(ing long time. I need to know whether it is something I can address, or whether I just toss the entire thing in the proverbial bit bucket. 

#### The system-call API

Schaufler eventually came back with [a proposal](<https://lwn.net/ml/linux-kernel/20221025184519.13231-1-casey@schaufler-ca.com/>) for two new system calls. The first of those is: 

```
struct lsm_ctx {
    	unsigned int		id;
    	unsigned int		flags;
    	__kernel_size_t		ctx_len;
    	unsigned char		ctx[];
        };
    
        int lsm_self_attr(struct lsm_ctx *context, size_t *size, int flags);
```

Here, `context` is a buffer that is `*size` bytes in length; the `flags` argument must be zero. This call will return all of the attributes assigned to the calling process by the security module(s) currently in force, in the buffer pointed to by `context`; [this patch](<https://lwn.net/ml/linux-kernel/20221025184519.13231-7-casey@schaufler-ca.com/>) describes the format of the returned data. The `size` parameter will be updated with the actual size of the returned data. The second system call can be used to determine which LSMs are currently active: 

```
int lsm_module_list(unsigned int *ids, size_t *size, unsigned int flags);
```

This call will fill the `ids` array with the ID numbers assigned to each of the active modules. These ID numbers are [defined](<https://lwn.net/ml/linux-kernel/20221025184519.13231-3-casey@schaufler-ca.com/>) in a new header file that is intended to be a part of the user-space API; Schaufler's Smack module, for example, is defined as: 

```
#define LSM_ID_SMACK	34
```

Much of this design follow Moore's initial suggestions. It appears to be mostly uncontroversial — with one significant exception. Tetsuo Handa, a developer of the Tomoyo LSM, has vociferously and repeatedly [objected](<https://lwn.net/ml/linux-security-module/a0567b10-fa83-50f4-7bf6-937e0c677e60@I-love.SAKURA.ne.jp/>) to the use of integer module IDs assigned within the kernel code itself. This practice will, he has argued, make it impossible to use run-time loadable LSMs that are not currently part of the kernel source. As a result, it will be hard for developers of LSMs to test them or (especially) get others to work with them. That, in turn, [spells](<https://lwn.net/ml/linux-security-module/ff43e254-0f41-3f4f-f04d-63b76bed2ccf@I-love.SAKURA.ne.jp/>) a ""death sentence"" for any new LSMs in the future, he said. 

As others have pointed out, there are a few problems with this argument, starting with the fact that the kernel-development community has never gone out of its way to make life easier for out-of-tree code. Another is that LSMs, whether in-tree or not, cannot be loaded at run time now. That capability was removed many years ago and [seems unlikely to return](<https://lwn.net/Articles/526983/>); among other things, it is too easy for LSMs to bypass the restrictions normally applied to kernel modules. For this reason, Handa's request to simply export the `security_hook_heads` variable to kernel modules is unlikely to be viewed favorably. Schaufler has also [said](<https://lwn.net/ml/linux-security-module/cc14bbde-529e-376c-7d27-8512ec677db3@schaufler-ca.com/>) repeatedly that any new mechanism for loadable LSMs would have to be treat those modules quite differently than built-in LSMs, since loadable LSMs would have to be more severely restricted. That is another big job that he personally has no intention of taking on. 

For all of these reasons, Handa's objections seem unlikely to prevail in the end. But this work, which has had such a turbulent history for so long, may still not be merged immediately. New system calls require extensive review, and that process has just begun; it wouldn't be surprising if more changes were called for. Even so, the end of the process for limited LSM stacking may be getting closer. Then all that is left is "universal stacking", a prospect that, [according to Schaufler](<https://lwn.net/ml/linux-security-module/7d0feee1-a9e6-ac82-6962-89b17af025eb@schaufler-ca.com/>), is ""at least a year off"". There is visible progress, but this lengthy discussion is not yet finished.
