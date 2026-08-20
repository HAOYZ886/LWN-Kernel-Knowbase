---
title: AutoFDO and Propeller
url: https://lwn.net/Articles/995397/
date: "October 28, 2024"
category: Optimization tools
author: "By Jake Edge October 28, 2024 LPC"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jake Edge**  
October 28, 2024

* * *

[LPC](<https://lwn.net/Archives/ConferenceIndex/#Linux_Plumbers_Conference-2024>)

Rong Xu and Han Shen described the kernel-optimization techniques that Google uses in the [toolchains track](<https://lpc.events/event/18/sessions/180/#20240918>) at the [2024 Linux Plumbers Conference](<https://lpc.events/event/18/page/224-lpc-2024-overview>). They talked about [automatic feedback-directed optimization](<https://static.googleusercontent.com/media/research.google.com/en//pubs/archive/45290.pdf>) (AutoFDO), which can be used with the [Propeller](<https://research.google/pubs/propeller-a-profile-guided-relinking-optimizer-for-warehouse-scale-applications/>) optimizer to produce kernels with better performance using profile information gathered from real workloads. There is a fair amount of overlap between these tools and the [BOLT](<https://github.com/llvm/llvm-project/tree/main/bolt#bolt>) post-link optimizer, which was the subject of a [talk](<https://lwn.net/Articles/993828/>) that directly preceded this session. 

#### AutoFDO

Xu started by saying that he would be covering AutoFDO, then Shen would talk about further performance enhancements using Propeller on top of AutoFDO. At Google, where Xu and Shen work, the amount of time spent executing in the kernel on its fleet was measured. Xu put up a slide showing the CPU-cycle percentage of the kernel, with idle time excluded, for a variety of workloads, which can be seen in the [slides](<https://lpc.events/event/18/contributions/1922/attachments/1450/3084/AutoFDO%20&%20Propeller%20in%20LPC%202024.pdf>) or [YouTube video](<https://www.youtube.com/watch?v=tPMvuFKI2Xo>) of the talk. Anywhere from 17-60% of the CPU cycles were spent in the kernel; the fleet-wide average is about 20%. 

There have been lots of efforts toward optimizing the Linux kernel, he said; at Google, one of the main techniques is [feedback-directed optimization](<https://en.wikipedia.org/wiki/Profile-guided_optimization>) (FDO), which uses run-time information to improve code generation. The core idea is to gather profile information on real workloads to allow the compiler to focus on the optimizations that have the most impact based on how the program runs. 

FDO has proven to be effective and is used on nearly all of the applications at Google, for Android, ChromeOS, and in the data center, he said. It improves instruction-cache usage, reduces TLB misses, and improves branch performance. The overall improvement can be up to 20% for the run time, but there is another benefit; FDO usually also reduces code size. 

[ ![\[Rong Xu\]](https://static.lwn.net/images/2024/lpc-xu-sm.png) ](<https://lwn.net/Articles/995371/>)

There are two major types of FDO: instrumentation-based and sample-based. The instrumentation-based variant (also known as iFDO) is the original one; it uses an instrumented kernel to run tests that represent the real workload, which result in the profile to be used. The profile is then fed to the compiler to build an optimized kernel that can be deployed to production. The instrumented kernel provides accurate profiles, without depending on performance monitoring unit (PMU) hardware, but the resulting kernel is up to ten times slower, so it cannot be used in production. The load tests still need to be representative, however. Beyond that, another disadvantage is that iFDO requires kernel changes for the instrumentation. 

Sample-based FDO (or AutoFDO) starts with a kernel built with debug symbols, then the perf tool is used to measure a representative workload. The raw perf data is converted to an AutoFDO profile that is fed to the compiler to build the optimized kernel. Because the perf-based data collection has such low overhead, the test kernel can be used in production with real workloads, which eliminates the concern about having truly representative test cases. But, AutoFDO does not optimize quite as well as iFDO because the profile data is not as complete; ""so if you use the same load test, you will get close performance, but not as good"". In addition, AutoFDO requires support for both a hardware PMU and perf. 

For fast-moving projects with incremental releases, the suggested model is to use an iterative process where the profile information gathered running the kernel in production for a particular release is fed to the compiler building the test kernel for the next release. That release is run in production and the profile gathered is fed in to build the following release—and so on. There are code mismatches, but AutoFDO uses line numbers relative to the start of functions; if there is no major refactoring in the release, ""we find this works pretty well"". The developers think this mode will work well for minor releases of the kernel, he said. 

AutoFDO requires last branch record LBR or similar support in the processor; AMD branch sampling (BRS) and Arm embedded trace macrocell (ETM) can provide the same kind of information. With that information, AutoFDO can create a profile that allows multiple types of optimizations to be made. There are three that usually have the most impact: function inlining, increasing branch fall-through, and indirect-call promotion. 

Inlining helps reduce both the size and run time of the code. Falling through branches is faster than taking a branch even if it is correctly predicted; in addition, that optimization groups hot basic blocks together for better instruction-cache utilization. Lastly, indirect calls can be promoted to direct calls, which allows them to be inlined. Other optimization passes can query the profile data to retrieve counts for basic blocks, branches taken, and so on, so those optimizations will be improved as well. 

Xu gave a brief report of some performance numbers. In nearly all of the tests (a mixture of micro-benchmarks and workload tests), iFDO improved performance slightly more than AutoFDO, and most were fairly modest gains (2-10%) overall. In a workload test of a Google database application, AutoFDO performance improved by 2.6%, while iFDO saw 2.9%. He worked with some Meta engineers who reported a roughly 5% performance increase in one of their services using AutoFDO and around 6% with iFDO. 

Overall, the path for using AutoFDO on the kernel has been fairly smooth. ""The testing turns out to be the most challenging part of the work""; finding representative workloads and benchmarks is difficult. Along the way, the developers have learned that profiles from Intel machines work well on AMD machines. For even better performance, he recommended using [link-time optimization](<https://llvm.org/docs/LinkTimeOptimization.html>) (LTO) and [ThinLTO](<https://clang.llvm.org/docs/ThinLTO.html>). 

#### Propeller

Shen then stepped up to describe using Propeller to build kernels. He started by describing what a post-link optimizer is, but noted that the BOLT talk just prior had covered it well. A post-link optimizer only requires the binary and a profile of it running on a representative workload, it does not usually need the source code. It uses a disassembler to convert the machine code into a compiler intermediate representation (IR), then does analysis and optimization, and finally converts back to machine code for the final output. 

Conceptually, Propeller is a post-link optimizer, but it takes a different approach; it uses the compiler and linker to do the transformations. As with other post-link optimizers, it takes in a binary and a profile to identify a series of optimizations, but it does not directly apply them. Instead, it writes compiler and linker profiles, then redoes the build using the same source code. The compiler and linker do their usual jobs, but use the profile information to apply the optimizations as part of the process. The result is the same, an optimized binary, but the internal steps are different. 

[ ![\[Han Shen\]](https://static.lwn.net/images/2024/lpc-shen-sm.png) ](<https://lwn.net/Articles/995372/>)

There are a few reasons why this approach was used. While working on post-link optimization, the developers found that the disassembler does not work on all binaries. For example, it does not work on binaries with read-only data intertwined with code, or for binaries that must comply with various standards, such as [FIPS](<https://en.wikipedia.org/wiki/Federal_Information_Processing_Standards>) compliance. Beyond that, there are scalability problems: ""the single-machine multithreaded design of the post-link optimizer leads to excessive processing time and memory usage"". That led them to shift some of that burden to the compiler and linker, he said. 

There are a few advantages to the Propeller mechanism. First, the optimization transformation is ""done with the uttermost accuracy"" because they are handled by the compiler and linker. It also ""opens the door for parallelizing because each compilation job is independent"". A distributed build cache can be used; if no optimization instructions are found for a particular source file, it does not need to be rebuilt and the cached object file can be used. It also means that Propeller itself does not need to deal with the kernel metadata that gets built; that is all handled by the existing build process. Propeller does require the source code, however, and the rebuild step, which are disadvantages of the approach. 

There are currently two major optimizations that are made by Propeller: basic-block layout and path cloning. There is also active work on adding inter-procedure register allocation, Shen said. 

Propeller can be combined with AutoFDO by building an AutoFDO profile that is used to build a kernel that is then profiled for Propeller. The profile used to build the AutoFDO kernel is also used to rebuild the kernel again with the Propeller profiles. Each kernel is run in production to generate the profiles needed. The final result is a kernel that has been optimized by both AutoFDO and Propeller. 

Unlike other post-link optimizers, Propeller can optimize kernel modules. Post-link optimizers operate on executables and shared libraries, but kernel modules are relocatable objects. Propeller can resolve the relocation information in the module to link to the proper symbols. 

As mentioned earlier, AutoFDO can tolerate minor source-code changes and small differences in the build options, but that is not true for Propeller. The source code and build options of the kernel used for generating the profile must be identical to those that Propeller uses. The developers ""are working hard to try to mitigate this limitation"", he said. 

Propeller requires specific software and hardware support to do its job. It depends on a [tool](<https://github.com/google/autofdo>) that lives outside of the LLVM source tree. The developers are currently working on migrating the functionality into LLVM. It also needs branch information from the CPU, which comes from features that vary across architectures (LBR for Intel, BRS or LBRv2 for AMD, and SPE or ETM for Arm). For internal applications, Propeller has been verified on each architecture (though not each branch-tracking feature); for the kernel, only Intel LBR has been validated at this point. 

Shen showed some heat maps and graphs of PMU stats for no FDO, iFDO, AutoFDO, and the latter two with ThinLTO and Propeller. As might be guessed, there were improvements in nearly all of the categories for Propeller. 

Currently, patches enabling AutoFDO and Propeller in LLVM have been [sent upstream for review](<https://lwn.net/ml/all/20241014213342.1480681-1-xur@google.com/>), Shen said. Internally, the developers are doing ""large-scale production tests to measure the performance company-wide""; they are also looking at customer kernels based on specific workloads with the idea of building different kernels for individual applications. 

In summary, he said that FDO, using either iFDO and AutoFDO, ""improves kernel performance significantly"". AutoFDO integrates well with the kernel build and allows profiling in production, which is a huge benefit over iFDO. Their advice is to add Propeller on top to get the best possible performance. 

During Q&A, Peter Zijlstra asked about why there was a need for multiple profile runs; the AutoFDO profile should already have the needed information, he said. Shen said that the first-pass optimizations change the code such that the profile information is out of date, which is where the Propeller profiling picks up. Zijlstra noted that the code is transformed again at that point, so another profile would be needed; Shen agreed that it is an iterative process and that more profiling and rebuilding passes improve the results. 

Zijlstra also complained that calling Propeller a post-link optimizer was not accurate. The optimization happens during the build process. Another audience member agreed, saying that the profile information was gathered post-link, but that it was simply another form of profile-guided optimization; Zijlstra suggested that Propeller should simply be called that to avoid the confusion. 

There is, it seems, a lot of work going into producing optimized kernels using LLVM, though BOLT can also operate on kernels built with GCC. That this work is being done by companies with large fleets of cloud systems is no real surprise; a tiny increase in performance multiplied by the fleet size makes for a rather large impact on CPU usage, thus power requirements, number of systems needed, and so on—all of which boils down to spending less money, of course. 

[ I would like to thank LWN's travel sponsor, the Linux Foundation, for travel assistance to Vienna for the Linux Plumbers Conference. ]
