---
title: Scheduler benchmarking with MMTests
url: https://lwn.net/Articles/820823/
date: "May 19, 2020"
category: "Benchmarking; Development tools-MMTests"
author: "By Jonathan Corbet May 19, 2020 OSPM"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
May 19, 2020

* * *

[OSPM](<https://lwn.net/Articles/820337/>)

The [MMTests](<https://github.com/gormanm/mmtests>) benchmarking system is normally associated with its initial use case: testing memory-management changes. Increasingly, though, MMTests is not limited to memory management testing; at the 2020 [Power Management and Scheduling in the Linux Kernel summit](<http://retis.sssup.it/ospm-summit/>) (OSPM), Dario Faggioli talked about how he is using it to evaluate changes to the CPU scheduler, along with a discussion of the changes he had to make to get useful results for systems hosting virtualized guests. 

Kernel benchmarking, he began, is typically done on bare metal. Developers want to know what the impact of a given kernel change might be, so they run a series of tests to measure performance in a reproducible setting. But Faggioli works in SUSE's virtualization lab, which has a more complicated set of objectives. A kernel change might have one effect on a host, but a different effect in guests running on that host. That leads to a need to run benchmarks with various combinations of baseline and changed kernels. Life gets even more interesting when you consider that benchmarks can take varying amounts of time to run between the host and a guest, or even between guests. Without some extra effort, a series of tests running simultaneously will not line up in any sort of predictable or repeatable way. 

For example, consider a test for hypervisor scheduling fairness. If the scheduler is fair, guests with equal computing requirements should get equal amounts of CPU time. One way to test that is to ensure that every benchmark takes the same amount of time to run. Even in the presence of fair scheduling, though, there may be differences in run times between one virtual machine and the next. If a series of tests is being run, the VMs could end up running different tests at any given time, muddying the results. [![\[Dario Faggioli\]](https://static.lwn.net/images/conf/2020/ospm/DarioFaggioli-sm.png)](<https://lwn.net/Articles/820824/>) The only way to get clear and deterministic results is to ensure that the benchmark runs on all systems run in a synchronized manner. 

There are, he said, a lot of testing and benchmarking suites to choose from. None of them, though, is able to perform synchronized runs in multiple virtual machines. He decided that the time had come to implement a suite that could do that, but he didn't want to start from scratch, so he based his work on MMTests. 

The MMTests suite dates back to at least 2012, Faggioli said (LWN [covered it](<https://lwn.net/Articles/509577/>) in August of that year). While it was initially focused on memory-management changes, that is no longer the case. It is mostly implemented in a combination of Bash and Perl. The core suite is able to fetch, build, configure, and run a whole range of benchmarks. Multiple runs can be made, with MMTests collecting and storing both the configuration that was used and the results that were obtained. A set of tools exists to compare results between runs, create plots, and more. There is also a "monitor" functionality that can capture the output from various monitoring commands (`top`, `vmstat`, `iostat`, etc.) as well as from sources like ftrace and perf events. The set of benchmarks that can be run is large, consisting of most of the tools that kernel developers have found useful over the years. 

The configuration file for MMTests is a Bash script containing a lot of `export` lines describing the tests to be run. There are commands to query system characteristics, such as the number of NUMA nodes; the results can be used to size the benchmarks appropriately. It is quite intuitive, Faggioli said — as long as you are familiar with the specific benchmarks you want to run. The `run-mmtests.sh` script will actually run the tests; there is a `compare_mmtests.pl` script to see what changed between different runs. Use `graph-mmtests.sh` to make pretty plots. 

It is possible to try running MMTests as a regular user, he said, but that's not necessarily the best idea. The tests won't fail, but MMTests will not be able to do everything it needs to get a proper run. It may, for example, try to make changes to the CPU-frequency governor. It tries to undo such changes at the end, but it's still better to run the tests on a disposable machine if possible. MMTests will download benchmarks from the net, then run them as root, which may give some users pause. It is possible to set up a local mirror, which can be good for both performance and security. 

For tests involving virtualization in particular, the `run-kvm.sh` script should be used; it will get results from both the host and guest(s). The script sets up and starts any virtual machines, as well as generating SSH keys to connect to those machines. The MMTests directory is copied directly into the virtual machines and the tests are run there. There are different configuration files for the host and the virtual machines; one may want to collect different data in the two settings, he said. 

Synchronization, which Faggioli had to add to MMTests, is handled by passing tokens between the host and the virtual machines; the guests never talk directly to each other. The host implements a "barrier" before each benchmark run; once every virtual machine has informed the host that it is ready for the next test, they are all told to proceed to the next one. This ensures that the tests on all systems start at the same time. 

Faggioli has various patches that he had really intended to submit before the talk, but that didn't happen despite his proclaimed affinity for "conference-driven development". That should happen soon. With regard to documentation, he said, there is absolutely none. But there is a nice ASCII-art diagram in the script for virtual-machine synchronization, at least. He concluded by saying he has considered rewriting the whole thing in Go, but he was not sure if Mel Gorman, the maintainer of MMTests, would be up for such an idea. Gorman, who was present at the event, held his peace regarding this idea. 

Douglas Raillard spoke up after Faggioli finished to say that Arm has [a test suite](<https://workload-automation.readthedocs.io/en/latest/>) that it uses; it lacks virtual-machine synchronization, though. It does some statistical testing on the results; he wondered if there were plans for adding that to MMTests. Faggioli said that he is not a statistician and wouldn't add that capability himself. Gorman said that MMTests does enough evaluation to try to guess whether a specific difference is significant; that is rather subtly marked in the output and is often missed. The fact that it is undocumented probably doesn't help. Raillard also asked about getting output in JSON format; Faggioli said there is JSON "in there somewhere" but he doesn't use it. 

The session concluded at this point. See [Faggioli's slides [PDF]](</images/conf/2020/ospm/faggioli-mmtests.pdf>) for details, example plots, configuration files, and more.
