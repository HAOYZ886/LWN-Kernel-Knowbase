---
title: A different approach to kernel configuration
url: https://lwn.net/Articles/733405/
date: "September 12, 2017"
category: "Build system-Kernel configuration"
author: "By Jonathan Corbet September 12, 2017 Open Source Summit"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
September 12, 2017

* * *

[Open Source Summit](<https://lwn.net/Archives/ConferenceByYear/#2017-Open_Source_Summit_North_America>)

The kernel's configuration system can be challenging to deal with; Linus Torvalds recently [called it](<https://lwn.net/Articles/733418/>) ""one of the worst parts of the whole project"". Thus, anything that might help users with the process of configuring a kernel build would be welcome. A talk by Junghwan Kang at the 2017 Open-Source Summit demonstrated an interesting approach, even if it's not quite ready for prime time yet. 

Kang is working on a Debian-based, cloud-oriented distribution; he wanted to tweak the kernel configuration to minimize the size of the kernel and, especially, to reduce its attack surface by removing features that were not needed. The problem is that the kernel is huge, and there are a lot of features that are controlled by configuration options. There are over 300 feature groups and over 20,000 configuration options in current kernels. Many of these options have complicated dependencies between them, adding to the challenge of configuring them properly. 

Kang naturally turned to the work that others have already done in an attempt to simplify his kernel-configuration task. One interesting project is [undertaker-tailor](<https://vamos.informatik.uni-erlangen.de/trac/undertaker/wiki/UndertakerTailor>) (also known as "the valiant little tailor"), a project that came out of the [VAMOS project](<https://www4.cs.fau.de/Research/VAMOS/>). This tool uses the ftrace tracing mechanism to watch a kernel while the system runs a representative workload. From the resulting traces, it concludes which parts of the kernel are actually used, finds the configuration options controlling those [![\[Junghwan Kang\]](https://static.lwn.net/images/conf/2017/ossna-lpc/JunghwanKang-sm.jpg)](<https://lwn.net/Articles/733422/>) parts, then generates a configuration that only includes the needed subsystems. This system, Kang said, is novel, but "incomplete". 

In particular, undertaker-tailor has a number of bugs; "it doesn't work and needs an overhaul". Kang tracked down and fixed some of the bugs, sending his fixes upstream in the process. The tool was badly confused by address-space layout randomization, for example. He fixed a few issues until he could get a configuration out of it. Unfortunately, the resulting kernel failed to boot. It turns out that this tool requires the user to spend some time setting whitelists and blacklists, but that brings the user back to the original configuration issue. 

Another tool for trimming down a kernel configuration is the `make localmodconfig` command. It simply looks at the modules loaded into the running kernel and assumes that each is there because something needed it. It generates a kernel configuration that builds in those modules and leaves out the rest. This approach did create a working kernel, but that kernel was still "fat", with numerous features configured in that were not really needed. 

So Kang went off to create a solution of his own. He wanted to come up with an automated system that would create a minimally sized but working kernel for his specific workload. His solution uses undertaker-tailor to collect system traces with the use cases of interest running. But then a separate "tailoring manager" runs to create the configuration from the trace data. As was the case before, this configuration is unlikely to boot and run properly. So another process works to "fill in" configuration options until the kernel eventually works. 

This filling-in stage uses the `localmodconfig` configuration as a starting point; it thus won't fill in options that are already known not to be necessary. The first stage looks at warnings from the configuration system itself, adding options until the warnings are addressed. Then kernels are built and tested using [Xnee](<https://www.gnu.org/software/xnee/>) to simulate a desktop session. There is also a hand-built blacklist used to explicitly exclude some options. 

This process, which involves building and testing a lot of kernels in virtual machines, takes about five hours to run. It generates a kernel that is quite a bit smaller than what `make localmodconfig` provides, with almost all modules configured out. As a bonus, this kernel boots in 1/5 of the time. 

Future steps include creating a larger set of workloads to be sure that all use cases for this distribution have been addressed. At some point, Kang also plans to add support for kernels running on bare metal; currently, only virtualized kernels can be configured in this way. Even now, though, he said that the resulting tool is useful for non-expert kernel users who are trying to build a kernel using something smaller than a kitchen-sink distribution configuration. Those users will have to wait, though, since Kang has not yet released this project to the world; he said he would like to do that once he receives management approval. 

#### Postscript

Presentations of this type are often as useful for the problem they pose as for the solutions they present. In this case, it's not entirely clear that "non-expert users" will find it easier to create representative workloads that cover all needed tasks, run them with a kernel under tracing, create a suitable blacklist, and generate their final configuration. The task still seems daunting. 

The problem is not Kang's solution, though; the problem is that he was driven to create such a solution just to get through the task of configuring a kernel to his needs. The kernel's configuration system is, indeed, one of the worst parts of the project. But it is also a part that nobody is really working on; it receives a bit of maintenance, but there does not appear to be any significant effort out there to address its shortcomings. Two-hundred companies support work on each kernel development cycle, but none of them see the configuration system as one of the problems that they need to solve. Until that changes, we are likely to continue to see users struggling with it. 

[Thanks to the Linux Foundation, LWN's travel sponsor, for supporting your editor's travel to the Open Source Summit.]
