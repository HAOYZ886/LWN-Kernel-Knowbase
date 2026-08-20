---
title: "Who are kernel defconfigs for?"
url: https://lwn.net/Articles/1026337/
date: "June 24, 2025"
category: "Build system-Kernel configuration"
author: "By Jonathan Corbet June 24, 2025"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
June 24, 2025

Working on the kernel can be a challenging task but, for many, _configuring_ a kernel build can be the largest obstacle to getting started. The kernel has thousands of configuration options; many of those, if set incorrectly, will result in a kernel that does not work on the target system. The key to helping users with complex configuration problems is to provide reasonable defaults but, in the kernel community, there is currently little consensus around what those defaults should be. 

The kernel's configuration options control many aspects of how a kernel will be built. There is one for every device driver, for example, and usually for the subsystems to which the drivers belong as well. Failing to enable a needed driver will result in a non-working kernel; enabling _all_ drivers will result in a long-running build process and a massive kernel at the end. Configuration options also control the availability of many features and system calls, the application of security mitigations and policies, many debugging features, subarchitecture support, and more. Many of the options either depend on or conflict with others. 

Working through this maze of configuration options is not fun; the kernel provides some tools for configuration editing, but they can only provide so much help. There are `make` options to either enable or disable the entire set of configuration options; these are useful for build testing, but are rarely used to create a kernel that will actually run somewhere. A bit more help can be had from `make localmodconfig`, which examines the modules loaded in the currently running kernel, then generates a configuration to build just those modules. Eric Raymond once [attempted to turn the configuration process into an adventure game](</2001/0621/a/kernel-adventure.php3>), but that did little to tame the monsters found therein. 

In the end, many people simply start with a known-working configuration (perhaps from their distributor), adjust it as necessary, and try to not think about it again for as long as the resulting kernel works. Linus Torvalds has [often said](<https://lwn.net/ml/all/CAHk-=wigjok_oSyrwiJMxQgTGYg9QtbG5xomMm9qQeO68MwsRw@mail.gmail.com/>) that the configuration system is a problem: 

> People, I've said this before, and apparently I need to say it again: the kernel config is likely the nastiest part of building a local kernel, and the biggest impediment to people actually building their own kernels. 
> 
> And people building their own kernel is the first step to becoming a kernel developer. 

There is one other operation, `make defconfig`, that will create an architecture-specific configuration that is intended to be a good starting point, providing a set of reasonable defaults that will create a working kernel. Ingo Molnar, one of the x86 maintainers, has recently come to the conclusion that the x86 default configuration has drifted from those goals; he posted [a patch series](<https://lwn.net/ml/all/20250515132719.31868-1-mingo@kernel.org>) that aimed to modernize that configuration: 

> Historically the x86 defconfigs aimed to be distro kernel work-alikes with fewer drivers and a substantially shorter build time. We regularly ask our contributors to test their changes on x86 defconfigs, and we frequently analyze code generation on such kernels as well. 
> 
> In practice, over the past couple of years this goal has diverged from what actual modern Linux distributions do these days, and this series aims to correct that divergence. 

The series makes quite a few changes to the existing default configuration: 

  * [Virtualization changes](<https://lwn.net/ml/all/20250515132719.31868-8-mingo@kernel.org>) include enabling guest support for the Xen, Jailhouse, ACRN, and Hyper-V hypervisors, along with Intel's TDX confidential-computing mechanism. Support for running as a KVM host is also [enabled](<https://lwn.net/ml/all/20250515132719.31868-7-mingo@kernel.org>). 
  * The current x86 default configuration does not include BPF; Molnar's patch set [turns it on](<https://lwn.net/ml/all/20250515132719.31868-9-mingo@kernel.org>), including the BPF security module. 
  * It [enables a number of memory-management options](<https://lwn.net/ml/all/20250515132719.31868-10-mingo@kernel.org>), including memory hotplugging, [kernel samepage merging (KSM)](<https://docs.kernel.org/admin-guide/mm/ksm.html>), transparent huge pages, [userfaultfd](<https://docs.kernel.org/admin-guide/mm/userfaultfd.html>), memory control groups, and the [multi-generational LRU](<https://lwn.net/Articles/856931/>). 
  * On the [core-kernel](<https://lwn.net/ml/all/20250515132719.31868-12-mingo@kernel.org>) front, features like [core scheduling](<https://lwn.net/Articles/861251/>), NUMA balancing, [namespaces](<https://lwn.net/Articles/531114/>), [pressure-stall information](<https://lwn.net/Articles/759781/>), and BSD process accounting are enabled. 
  * A number of [debugging options](<https://lwn.net/ml/all/20250515132719.31868-11-mingo@kernel.org>) are also enabled, including the [kgdb](<https://docs.kernel.org/process/debugging/kgdb.html>) debugger, [UBSAN](<https://docs.kernel.org/dev-tools/ubsan.html>), function profiling, and more. 

The series generated few comments in general, but Peter Zijlstra complained, [suggesting](<https://lwn.net/ml/all/20250614103915.GM2278213@noisy.programming.kicks-ass.net>) that the memory-management changes (and enabling KSM in particular) were ""a giant security fail"". Torvalds, in turn, [strongly told Molnar to stop this work](<https://lwn.net/ml/all/CAHk-=winzTt3SCzv8BWGMm0fzrXS0gb59gK0h4dAe0L6hj3X_w@mail.gmail.com>), saying that the default configuration should be for ""normal people"". Options that are useful to cloud providers (such as the virtualization subsystems) should not be enabled, he said. The fact that all distributors enable a specific option is also not, in his mind, an argument for enabling that option in the default configuration. Torvalds would seemingly like to see the configuration system made easier to navigate, but this is evidently not the way he wants that to be done. 

Molnar [responded](<https://lwn.net/ml/all/aE6tFoBkF3tl9aeH@gmail.com>) with the detailed reasoning behind his changes, saying that he wants easier kernel configuration for the people who contribute patches to the kernel. That group includes developers working for cloud providers, who Torvalds does not see as "normal people". Molnar asked: 

> Why not make the defconfig work out of the box for the testing environments of a broader group of our actual contributors, as long as the build cost isn't overly high? 

He added that kernel developers tend to work (and test) with configurations that differ significantly from what the distributors are shipping, and that can lead to problems downstream when distributors enable the options the developers are avoiding. 

Despite laying out his reasoning, Molnar is seemingly not ready for a fight. He has removed most of the changes from the patch series, saying: 

> These commits are not coming back. Clearly my approach of using the lowest common denominator of distro kernel configs is not appreciated and I have no desire whatsoever to fight such pushback. 

Torvalds did not respond, and nobody else has jumped in, so the conversation ended there. For the time being at least, the x86 default configurations will differ significantly from those used by distributors, and will lack features that many people might prefer to have in their built kernels. It may stay that way for some time. The kernel's build system is an area where few developers choose to go; it is maintained (and probably only really understood) by a single developer. If a consensus cannot be reached even on a set of basic defaults, it seems unlikely that more significant improvements can be expected.
