---
title: Static code checks for the kernel
url: https://lwn.net/Articles/683476/
date: "April 13, 2016"
category: "Development tools-Static analysis"
author: "By Nathan Willis April 13, 2016 ELC"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Nathan Willis**  
April 13, 2016

* * *

[ELC](<https://lwn.net/Archives/ConferenceByYear/#2016-Embedded_Linux_Conference>)

At the 2016 [Embedded Linux Conference](<http://events.linuxfoundation.org/events/embedded-linux-conference>) in San Diego, Arnd Bergmann presented a [session](<http://openiotelc2016.sched.org/event/6DAc>) on what he called a "lighter topic," his recent efforts to catch and fix kernel bugs through static tests. Primarily, his method involved automating a large number of builds, first to catch compilation errors that caused build failures, then to catch compiler warning messages. He has done these builds for years, progressively fixing the errors and then the warnings for a range of kernel configurations. 

There are two motives for this particular side project, he said: to help automate the testing of the many pull requests seen in the arm-soc tree (for which the sheer number of SoCs presents a logistical challenge), and to put significant code-refactoring work to the test. Previously, he explained, he had attempted to review every pull request in arm-soc and fix every regression, but that quickly proved too time-consuming to be done manually. Testing each patch automatically first reduced the time required. As for refactoring, he noted that he was a veteran of the big kernel lock removal days and was now helping out with the effort to implement year-2038 compliance. In both cases, the refactoring touched hundreds of separate drivers, which can mean a glut of regressions. 

Broadly speaking, he said, there are two approaches to testing scores of builds. One can either record all known warnings and send an email whenever a new warning appears, or one can try to eliminate all known warnings. Bergmann has opted for the second approach, running a near-constant stream of kernel builds, and creating a patch for every compiler warning he sees. At present, he reported, there are about 500 such patches, most of them tiny. He is currently automating builds with a script he wrote that creates a random kernel configuration and attempts a build. He is averaging 50 builds a day, almost all for 32-bit ARM, with occasional forays into 64-bit ARM and, rarely, other architectures. 

Getting to this current state has taken some time. In 2011, he began by fixing all of the failures produced by running `make defconfig` (that is, "default configuration") and `make allmodconfig` (that is, configuring as many symbols to "module" as possible) builds in the arm-soc tree. By 2012, those failures were eliminated, so he set out to eliminate all compiler warnings [ ![\[Arnd Bergmann\]](https://static.lwn.net/images/2016/04-elc-bergmann-sm.jpg) ](<https://lwn.net/Articles/683558/>) produced by `defconfig` builds. By 2013, those warnings had been eliminated, and he began running his build tests with `make randconfig`--which creates a randomized kernel configuration. In 2013, he had eliminated all `randconfig` failures, and turned to eliminating the `allmodconfig` warnings. He began chipping away at `randconfig` warnings in mid-2014. Although that process is not yet complete, he has also begun to run build tests using the Clang compiler instead of GCC, which, as one would expect, generates entirely different errors and warnings. 

The most common bugs he discovers with `randconfig` builds are missing dependency statements, he said, which cause necessary parts of the kernel to not get built. In particular, he cited missing Netfilter dependencies and ALSA codec dependencies as common, although he also noted that x86 developers seem to forget that, at least on ARM, I2C can be configured as a module and thus needs to be listed as a dependency if it is needed. The ALSA problems suggest that we need a better way to express codec dependencies, he said, although he conceded that kernel configurations are confusing in plenty of ways. For example, he showed this patch he had written: 

```
--- a/net/openvswitch/Kconfig
        +++ b/net/openvswitch/Kconfig
        @@ -7,7 +7,9 @@ config OPENVSWITCH
          depends on INET
          depends on !NF_CONNTRACK || \
               (NF_CONNTRACK && ((!NF_DEFRAG_IPV6 || NF_DEFRAG_IPV6) && \
        -               (!NF_NAT || NF_NAT)))
        +               (!NF_NAT || NF_NAT) && \
        +               (!NF_NAT_IPV4 || NF_NAT_IPV4) && \
        +               (!NF_NAT_IPV6 || NF_NAT_IPV6)))
          select LIBCRC32C
          select MPLS
          select NET_MPLS_GSO
```

and asked "what does it even mean for it to depend on `NF_NAT` or _not_ `NF_NAT`?" The answer, he said, is that the test is being used to set an "is it a module or not" dependency for later usage, but it is hardly surprising that such syntax leads to bugs. 

After "modules, modules, and more modules," Bergmann said, the next most common class of bugs he catches is uninitialized variables. He noted that Rusty Russell has [written](<http://rusty.ozlabs.org/?p=232>) about how uninitialized variables are useful for error catching, but argued that they cause plenty of other errors. He showed a few examples, noting that often the flow of the code may mean that a reference to an uninitialized variable can never be reached, but he writes patches anyway to eliminate the warning. He also pointed out Steven Rostedt's [patch to override `if`](<https://lwn.net/Articles/308586/>) (for tracing purposes), saying it totally confused GCC, but that it helps to uncover quite a few bugs. 

Next, Bergmann discussed some of the other code-checking tools available for kernel development, like `scripts/checkpatch.pl` and [Sparse](<http://git.kernel.org/pub/scm/devel/sparse/chrisl/sparse.git>). Checkpatch looks for basic coding-style issues, he said, so while it is beneficial for submitting patches, it is not particularly valuable to run against existing code. 

Sparse, however, makes use of annotations in the kernel source, therefore it can catch problems that GCC, with its lack of "domain-specific knowledge," simply cannot. Its big drawback is that it generates a lot of false positives. From the audience, Darren Hart noted that he uses Sparse regularly, but finds it problematic because it runs on complete files, rather than on patches alone. Therefore it tends to generate a lot of warnings that, upon inspection, were present in the original file and not the patch. Mauro Carvalho Chehab replied that some subsystem maintainers made an effort to remove all Sparse warnings in order to eliminate that particular problem, though far from all. 

Bergmann also said he makes use of some extra GCC warnings to catch additional bugs. Kernel builds can employ a sort of "graduated" warning level thanks to [work](<https://git.kernel.org/linus/a6de553da01c2c110c647fbd1c5bad29f5a82487>) by Michael Marek: the `W=12` switch includes all warnings from `W=1` and `W=2`; `W=123` adds the `W=3` warnings as well. Using `make W=1` is generally useful, he said, with `W=12` adding little of value in a lot more noise, and `W=123` being clear overkill, mostly due to an "explosion" of false positives. In the arm-soc tree, for instance, `W=1` generates 631 instances of the most common warning, `W=12` tops out at 94,235 for its top offender, and `W=123` generates 782,719. The additional warnings of greatest interest to Bergmann include missing headers and missing prototypes. Bergmann also noted that he has recently run build tests with GCC 6, with promising results among the [new warnings](<https://gnu.wildebeest.org/blog/mjw/2016/02/15/looking-forward-to-gcc6-many-new-warnings/>)--so far, he has written 32 patches based on GCC 6 warnings. Most have already been applied. 

Bergmann touched briefly on his experiments looking for build errors and warnings with Clang. That effort requires support from the [LLVMLinux](<http://llvm.linuxfoundation.org/index.php/Main_Page>) project, of course, and at the moment the patch set necessary to even compile the kernel with Clang is broken for mainline. But, since January (when he started his experiments), he has found "tons of new warnings." He eliminated the build errors found with Clang on `randconfig` builds, but has not yet tackled writing patches for the warnings. Clang also has a built-in static analyzer, he noted, which can produce rather nice-looking output and for which you can write your own checks, but he has not yet had the time to work with it. 

Moving a bit further afield, he mentioned the proprietary Coverity scanning tool, for which Dave Jones has done "some amazing work" to record and annotate the known findings (which is necessary because Coverity requires manual categorization of the bugs it finds). The downside from Bergmann's perspective, though, is that Coverity is x86-only. He also pointed the audience to Julia Lawall's [Coccinelle](<http://coccinelle.lip6.fr/>), which can do sophisticated pattern matching. He has worked with it for his own static checking, he said, though he has found it "really slow." Consequently, it is not a tool he would use in his own work, though he admitted he may be doing something wrong. 

Another tool Bergmann does not use regularly, but that he cited for its "surprisingly good" warnings, is Dan Carpenter's [Smatch](<http://repo.or.cz/w/smatch.git>). Carpenter has used it to catch thousands of bugs, he said, and pointed audience members to Carpenter's recent [blog post](<https://blogs.oracle.com/linuxkernel/entry/smatch_static_analysis_tool_overview>) for further information. Next, Bergmann highlighted the convenience of the [0day build bot](<https://lwn.net/Articles/514278/>) maintained by Fengguang Wu; in addition to monitoring public Git trees, it recently started testing patch submissions and generating patches. And, finally, he noted the [kernelci.org](<http://kernelci.org/>) build-and-boot testing infrastructure. The most interesting part of the project for Bergmann is that the service is ARM-centric and the build farm includes a wide variety of machines. 

By that point in the session, time had run out, so there was not much opportunity for the audience to ask questions. Nevertheless, it was surely an informative look at how static code checking benefits the arm-soc tree, where the ever-expanding list of supported hardware makes for a daunting maintainer workload. Furthermore, as Bergmann pointed out more than once, there are benefits to squashing warnings in addition to compilation errors, regardless of what code one is testing. 

_[The author would like to thank the Linux Foundation for travel assistance to attend ELC 2016.]_
