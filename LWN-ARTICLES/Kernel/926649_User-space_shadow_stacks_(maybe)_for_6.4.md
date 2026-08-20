---
title: "User-space shadow stacks (maybe) for 6.4"
url: https://lwn.net/Articles/926649/
date: "March 24, 2023"
category: "Releases-6.6; Security-Control-flow integrity"
author: "By Jonathan Corbet March 24, 2023"
---

By **Jonathan Corbet**  
March 24, 2023

Support for shadow stacks on the x86 architecture has been long in coming; LWN [first covered this work](<https://lwn.net/Articles/758245/>) in 2018. After five years and numerous versions, though, it would appear that user-space shadow stacks on x86 might just be supported in the 6.4 kernel release. Getting there has required a few changes since we last [caught up with this work](<https://lwn.net/Articles/885220/>) in early 2022. 

Shadow stacks are a defense against [return-oriented programming (ROP)](<https://en.wikipedia.org/wiki/Return-oriented_programming>) attacks, as well as others that target a process's call stack. The shadow stack itself is a hardware-maintained copy of the return addresses pushed onto the call stack with each function call. Any attack that corrupts the call stack will be unable to change the shadow stack to match; as a result, the corruption will be detected at function-return time and the process terminated before the attacker can take control. The above-linked 2022 article has more details on how x86 shadow stacks, in particular, work. 

The [current version](<https://lwn.net/ml/linux-kernel/20230319001535.23210-1-rick.p.edgecombe@intel.com/>) of the patch set is the eighth revision posted by Rick Edgecombe (who took it over after some 30 revisions posted by Yu-cheng Yu). 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

#### API changes

The user-space API for working with shadow stacks has not changed much in the last year. Most operations are done with [`arch_prctl()`](<https://man7.org/linux/man-pages/man2/arch_prctl.2.html>) calls, specifically: 

  * `ARCH_SHSTK_ENABLE` turns on the shadow stack for the current thread; shadow stacks are not enabled by the kernel when a process starts. 
  * `ARCH_SHSTK_DISABLE` disables the use of the shadow stack for the current thread. 
  * `ARCH_SHSTK_LOCK` prevents any further changes to a thread's shadow-stack status. Among other things, this operation can keep an attacker from somehow disabling the shadow stack before corrupting the call stack. 
  * `ARCH_SHSTK_UNLOCK` undoes the effect of `ARCH_SHSTK_LOCK`. This option was added to [version 4](<https://lwn.net/ml/linux-kernel/20221203003606.6838-1-rick.p.edgecombe@intel.com/>) of the patch set in December; it exists to support functionality like [Checkpoint/Restore in User Space](<https://criu.org/Main_Page>) that needs to be able to change the shadow-stack status after a process has launched. This option is only available when invoked via [`ptrace()`](<https://man7.org/linux/man-pages/man2/ptrace.2.html>); a process cannot use it on itself directly. 
  * `ARCH_SHSTK_STATUS` returns the current shadow-stack status. 

Normally, the kernel handles the allocation and placement of shadow stacks, but there are occasions where an application will need to manage its shadow stacks directly. The `map_shadow_stack()` system call exists for this purpose; its prototype has changed a bit over the course of the last year: 

```
void *map_shadow_stack(unsigned long address, unsigned long size,
        			   unsigned int flags);
```

**Same old SHSTK**  
At one point, Andrew Morton [complained](<https://lwn.net/ml/linux-kernel/20230119142602.97b24f3cdba75f20f97786d3@linux-foundation.org/>) about the "shstk" abbreviation, saying that it ""sounds like me trying to swear in Russian while drunk"". As a result, that term was pulled out of much of the generic code, but remains in the x86 portion. 

This call will attempt to set up a shadow stack at the given `address` of the requested `size`, returning the actual mapped address on success. The one possible value for `flags` is now called `X86_FEATURE_USER_SHSTK`; it requests that the necessary "restore token" — which, among other things, prevents multiple threads from sharing the same shadow stack — be stored into the newly created stack. 

There is one other subtle change to `map_shadow_stack()` that affects how shadow stacks are handled in general. The shadow-stack feature has incompatibilities with 32‑bit code, especially when signals are involved. The kernel will refuse to enable a shadow stack for a thread that is running in the 32-bit mode and, in [version 4](<https://lwn.net/ml/linux-kernel/20221203003606.6838-1-rick.p.edgecombe@intel.com/>) of the patch set, code was added to simply disable any signal handlers if a process switched to 32-bit mode after the shadow stack was enabled. 

Beyond seeming like a bit of a hack, this approach did not fully solve the problem. As it turns out, a 64-bit thread can switch to the 32-bit mode without the kernel's knowledge or permission — meaning that the disabling of signal handlers can be circumvented. After some deliberation on how to avoid subtle problems when this happens, the decision was made (for [version 5](<https://lwn.net/ml/linux-kernel/20230119212317.8324-1-rick.p.edgecombe@intel.com/>)) to just always map the shadow stack at a virtual address above 4GB, making it inaccessible to 32-bit code. As a result, any attempt to switch to the 32-bit mode when a shadow stack is enabled will cause an immediate crash. 

This change resulted in a new [`mmap()`](<https://man7.org/linux/man-pages/man2/mmap.2.html>) flag, `MAP_ABOVE4G`, which forces the mapping to be created above the 4GB virtual-address boundary. The `address` passed to `map_shadow_stack()` (if not zero, indicating no preference) must also be above 4GB or the call will fail. Someday, somebody with sufficient motivation could perhaps find a way to make 32-bit code work with shadow stacks, but given how little interest there is in 32-bit code in general, that seems unlikely to happen. 

#### The glibc problem

While it might be nice to run all programs with shadow stacks enabled, there are applications that would break in that environment. Anything that manipulates its own call stack — just-in-time compilers, for example — will find itself out of sync with the shadow stack and brought to an untimely end. So the enabling of the shadow stack must be limited to code that can handle it. 

The scheme that was developed, some time ago, was to place a special note in the `.note.gnu.property` ELF section of the program's executable image. If that note exists (as the result of compiler options provided when the program was built), that indicates that it is safe to run the program with the shadow stack enabled. That note is not sufficient for the kernel to make the decision, though, so the enabling of the shadow stack is left to user space, and to the C library's program loader in particular. 

Enthusiastic developers in the GNU C Library (glibc) community quickly wired up support for turning on the shadow stack when it seemed appropriate; current versions of glibc are poised to turn on the shadow stack as soon as the kernel supports the feature. There is only one little problem: the glibc support was written with an early version of the user-space API in mind. That API no longer exists; trying to use it would result in crashing programs and a failure to boot. That will indeed secure it against ROP attacks, but users can be picky about just how that kind of security was achieved and may complain. 

That problem was resolved early on by changing the API enough that glibc simply doesn't find it anymore and thinks that the shadow-stack functionality is not present. The glibc developers have said, though, that they intend to implement the new shadow-stack API once it is merged; thereafter, when an updated glibc shows up on a system, any program that indicates a readiness for a shadow stack will get one. 

That leads to a new problem, as noted in the version-3 cover letter: not all applications that are marked as being ready really are. 

> But many application binaries with the bit marked exist today, and critically, it was applied widely and automatically by some popular distro builds without verification that the packages actually support shadow stack. So when glibc is updated, shadow stack will suddenly turn on very widely with some missing verification. 

Applications that will break in this environment evidently include node.js and PyPy, so this seems like a real problem. A quick check on a Fedora 37 system shows that PyPy is indeed built with the shadow stack enabled: 

```
$ readelf -n /usr/bin/pypy
        Displaying notes found in: .note.gnu.property
          Owner                Data size        Description
          GNU                  0x00000040       NT_GNU_PROPERTY_TYPE_0
              Properties: x86 feature: IBT, **SHSTK**
        [...]
```

Even if the root cause lies in user space, it can be provoked by upgrading to a new kernel, and thus looks like a kernel regression. Kernel developers generally prefer to avoid breaking systems, even if that breakage can be said to be somebody else's fault. 

The ideal solution, according to Edgecombe, would be to simply move to a new ELF bit to identify _real_ shadow-stack readiness and have glibc use that. Distributors could then be encouraged to be more careful about marking applications as being shadow-stack ready. But, he said, ""it doesn’t seem like the glibc developers are interested in working on a solution"", so something else is needed. In version 3, that something else was [a patch](<https://lwn.net/ml/linux-kernel/20221104223604.29615-38-rick.p.edgecombe@intel.com/>) disabling the shadow-stack API when the ELF bit is detected. The idea was that distributors would eventually disable that check once they had confirmed that all of the packages they ship included correctly marked binaries. 

The patch was described as ""a bit dirty"" and included for the sake of discussion — which indeed resulted. H.J. Lu [suggested](<https://lwn.net/ml/linux-kernel/CAMe9rOpfSccXVWmgK6E0Y0DXC=VX3PpdxXookN1Ty8soeAxrKw@mail.gmail.com/>) that the right approach was just to avoid upgrading glibc until the system was ready for it. Florian Weimer [added](<https://lwn.net/ml/linux-kernel/87h6zaiu05.fsf@oldenburg.str.redhat.com/>) that most of the incompatible code is to be found in libraries that are loaded after a process starts; the kernel test would not detect those, and it may be too late to disable the shadow stack in any case. 

After a while, Edgecombe [asked Linus Torvalds](<https://lwn.net/ml/linux-kernel/7d8133c7e0186bdaeb3893c1c808148dc0d11945.camel@intel.com/>) what he thought should be done about this problem. Torvalds [answered](<https://lwn.net/ml/linux-kernel/CAHk-=wgP5mk3poVeejw16Asbid0ghDt4okHnWaWKLBkRhQntRA@mail.gmail.com/>) that he did not want to preemptively disable shadow-stack support without a reason: 

> Once [shadow-stack functionality] is enabled in the kernel, and it turns out that people complain that it breaks existing binaries, at that point I guess it gets disabled again. Possibly at that point using something like your suggested patch. But I'm not doing it until actual problems appear, and until we actually have this code in the kernel. 

The patch disabling the shadow-stack API was duly taken out of the series. Weimer [described](<https://lwn.net/ml/linux-kernel/87a645prpj.fsf@oldenburg.str.redhat.com/>) a couple of plans for ensuring that shadow stacks could be safely enabled in distributions, claiming that adopting a new ELF bit would delay that process considerably. Shadow-stack support, he said, is not much different from supporting a new system call; that, too, can break existing applications, mostly as the result of [`seccomp()`](<https://man7.org/linux/man-pages/man2/seccomp.2.html>) filters that do not understand the new call. 

#### On to 6.4

The result of the discussion is that the kernel will take no special steps to avoid breaking binaries that were incorrectly marked as being ready for shadow stacks — at least, not before a problem is demonstrated. Most of the other outstanding issues appear to be resolved, to the point that Edgecombe prefixed the current version with a remark that ""we have a pretty good initial shadow stack implementation here"". There are a number of desired enhancements, but those might be done better, he said, after there has been some real-world use of the code that exists now. 

So, after all this work, the 40 shadow-stack patches have been added to the [tip tree](<https://www.kernel.org/doc/html/latest/process/maintainer-tip.html>), which feeds them into linux-next. If no show-stopping problems turn up over the course of the next month or so, user-space shadow-stack support for x86 systems will, most likely, move upstream during the 6.4 merge window. Finally, after a long development period, the shadow (stack) will truly know what evil lies in the heart of ROP attackers.
