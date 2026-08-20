---
title: Measuring packet classifier performance
url: https://lwn.net/Articles/675056/
date: "February 10, 2016"
category: "Networking-Performance"
author: "By Nathan Willis February 10, 2016 Netdev/Netconf"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Nathan Willis**  
February 10, 2016

* * *

[Netdev/Netconf](<https://lwn.net/Archives/ConferenceByYear/#2016-Netdev>)

At [Netdev 1.1](<http://www.netdevconf.org/1.1/>) in Seville, Spain, Jamal Hadi Salim presented the results of recent tests he has done to assess the performance characteristics of the kernel's various network-packet classifiers. While the raw numbers may be of interest to those seeking the best possible performance, the testing process itself revealed other factors of interest to the kernel networking community. 

There are a number of reasons to run such tests, he said. First, there have been two new classifiers added in the past year: [flower](<https://lwn.net/Articles/642028/>) and the extended Berkeley Packet Filter (eBPF) [classifier](<http://www.spinics.net/lists/netdev/msg316054.html>). Naturally, that makes one curious how they compare to the older mechanisms. Second, he had spent several years working with software-defined networking (SDN) running on big application-specific integrated circuit (ASIC) hardware switches, and was interested in seeing how much could be done within the kernel itself. Third, an examination of kernel performance would also serve to counterbalance all the "noise" one hears about the superiority of the [Data Plane Development Kit](<http://dpdk.org>) (DPDK) and similar user-space libraries. And, he added, it just sounded like an easy topic for a paper--although that notion turned out to be delusional.

[ ![\[Jamal Hadi Salim\]](https://static.lwn.net/images/2016/02-netdev-jamal-sm.jpg) ](<https://lwn.net/Articles/675095/>)

#### Constructing a test

In order to not compare apples and oranges, he said, some care has to be taken in designing both the tests and the test system. If the goal is to allow a comparison between the kernel and ASICs, then the actions tested in kernel classifiers needed to be limited to those a typical ASIC would perform--no hashing or using masks to group flows. As to testing the new classifiers, a baseline is required, so he choose to profile the "Swiss-army knife of packet filtering," the [u32](<http://man7.org/linux/man-pages/man8/tc-u32.8.html>) classifier. 

He then spent a few minutes comparing the design and functionality of the various classifiers. "Classic" BPF, he pointed out, was a register-based virtual machine allowing classification rules that branch but, notably, could not include loops. Classic BPF has now been superseded by eBPF, but the prohibition on loops remains. Still, the virtual-machine byte code suggests a natural mapping to hardware CPU instructions, and the recent work to add just-in-time (JIT) compilation provided substantial performance improvements. In fact, they were substantial enough that he dropped all non-JIT eBPF data from his performance tests, as the JIT version was always significantly superior. 

The new eBPF classification engine allows one to compile multiple, independent "proglets," which are then loaded into the `tc` classifier. More importantly, the proglets can be combined to create policy loops within the `tc` framework. 

The flower classifier was written by Jiří Pírko and, cleverly, makes use of several "commodity" kernel features. As a packet traverses the stack, the flow it uses is cached. Flower then stores the flow in an [rhashtable](<https://lwn.net/Articles/612100/>). Subsequent packets matching a classifier rule can quickly be directed into the correct flow by retrieving the cached copy from the rhashtable. Currently, flower supports rules based on just a subset of the possible flow parameters (source and destination address, ingress and egress ports, MAC addresses, etc), but the design supports extending the set of supported tuples, and it will most likely continue to grow. 

The "u" in the name of the u32 classifier stands for "ugly," he said, and that is a more or less apt description. Consequently, few users know u32 well, but it can surprise them when it is well-tuned. The design is centered on a set of hash tables. Each filter rule can direct a matching packet to any of a series of buckets, each of which can then optionally point to another hash table. All flows begin at the root bucket; a user that knows the exact flow a packet should take can script the u32 rules to be extremely efficient. The fact that few do so points to usability problems, not to performance limitations. 

For his tests, Hadi Salim choose to measure data-path throughput performance. It is the simplest metric and easy to understand: one fires off a bunch of packets and then counts how many make it through in a given time period. He hopes to continue the work and measure latency as well, but did not have time to collect those results to present. All of the classifiers were tested on the same hardware: a quad-core Intel NUC with 16GB of the fastest RAM available (1600MHz). The machine ran kernel net-next 4.4.1-rc1, patched to support flower. 

The classifiers were attached at ingress and connected to an egress queue on a dummy network device (to remove any influence of device driver performance). Several "baseline" tests were run to measure the system characteristics at other points (such as dropping all packets at ingress) in order to properly account for the affects of the rest of the system. Then, each classifier was run with a variety of rule sets (from a single rule up to 1000 rules), and across a variety of packet sizes. 

#### Results and observations

After several thousand test runs, he said, the most interesting conclusions he drew were not which classifier had the highest throughput--the throughput winner was u32 in essentially every test permutation--but the unusual performance characteristics observed in the system across the variables. For instance, there was no measurable variance on any of the classifiers: the mean scores were indistinguishable from the maximums and minimums. 

More importantly, though, he initially tested classifier rules that forwarded packets, but the performance hit was so substantial that it overshadowed the other factors. The test machine was able to produce a peak throughput of 60Gbps (the maximum throughput being observed with 1020-byte packets) when copying packets to the dummy interface. But when forwarding the packets to a "blackhole" destination IP, the throughput rather mysteriously dropped to 25Gbps. 

The lack of variance is probably a good sign from a reliability standpoint, but the forwarding performance suggests that the kernel's forwarding code should get a closer look. The tests also suggested that memory latency is a significant factor in throughput. He said he hopes to find RAM chips with a different latency and re-run the same tests. 

As to the actual throughput numbers of the classifiers, which many were interested in seeing plotted together, the "best case" performance test measured the classifiers on a set of rules where the first rule matched every packet. u32 processed around 180Gbps, eBPF 160Gbps, and flower 60Gbps. The "worst case" test measured the classifiers with 1000-rule sets, for which the last rule matched. Under those circumstances, u32 processed 463Mbps, flower 88Mbps, and eBPF 73Mbps. In between, all three classifiers' throughput dropped off at about the same rate as the size of the rule set increased. He noted, though, that the tests always showed flower's worst-case performance scenario, since the test framework forced flower to cache every packet's flow. Its real-world behavior can only be better. 

Time ran short on the conference schedule, so he had to skip over the details of several test runs (and never got back to the question of comparing the kernel's performance to ASIC hardware), but Hadi Salim took a few minutes at the end to point out that "throughput" is hardly the only factor worth considering. In particular, the design and execution of the tests provided practical insights along the way. The eBPF classifier is the best option for extensibility, he said, but flower is the clear winner on usability. The flower command-line interface is the most human-friendly option, and it is the easiest classifier to control from an external program. In contrast, time spent scripting the u32 classifier could produce a 4x performance improvement compared to u32's unscripted usage, but there are few network administrators who seem to regard that time spent as time well-used. 

_[The author would like to thank the Netconf and Netdev organizers for travel assistance to Seville.]_
