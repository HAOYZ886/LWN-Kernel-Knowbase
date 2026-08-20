---
title: Defending against Rowhammer in the kernel
url: https://lwn.net/Articles/704920/
date: "October 28, 2016"
category: "Security-Security technologies"
author: "By Jonathan Corbet October 28, 2016"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
October 28, 2016

The [Rowhammer](<https://en.wikipedia.org/wiki/Row_hammer>) vulnerability affects hardware at the deepest levels. It has proved to be surprisingly exploitable on a number of different systems, leaving security-oriented developers at a loss. Since it is a hardware vulnerability, it would appear that solutions, too, must be placed in the hardware. Now, though, an interesting software-based mitigation mechanism is under discussion on the linux-kernel mailing list. The ultimate effectiveness of this defense is unproven, but it does show that there may be hope for a solution that doesn't require buying new computers. 

Rowhammer works by repeatedly reading the same memory location a large number of times. With contemporary DRAM, reading a location is a destructive act; the memory controller must rewrite the data into that location after each read. Those rewrites can cause neighboring memory cells to discharge slightly; if an attacker causes rewriting to happen too many times before the next regular refresh cycle happens, they can corrupt data in those neighboring cells. The result is seemingly random bit flips in nearby memory. 

This would appear to be a difficult vulnerability to exploit. An attacker must find memory that is known to be adjacent to data of interest, then manage to corrupt that data in a useful way. But attackers can do surprising things; a fair number of Rowhammer exploits have now been posted. That includes [the "Drammer" exploit](<http://arstechnica.com/security/2016/10/using-rowhammer-bitflips-to-root-android-phones-is-now-a-thing/>) that works on many Android devices. Rowhammer is thus a serious problem. Unfortunately, the only proper solution appears to be to increase the memory refresh rate, something that cannot generally be done in deployed hardware. 

An intriguing alternative turned up on the linux-kernel list, though its nature wasn't immediately clear. Pavel Machek [asked a question](<https://lwn.net/Articles/704921/>) that raised some eyebrows: ""I'd like to get an interrupt every million cache misses... to do a printk() or something like that."" Developers naturally wondered what he was up to. The answer turns out to be an in-kernel Rowhammer defense. 

Contemporary CPUs are generally equipped with performance-monitoring units (PMUs) that can track many aspects of how the system is running. Normally the PMU is used by utilities like `perf` for system profiling and performance tuning. But one of the events the PMU can track is memory-cache misses. For Rowhammer to work, it must act on main memory; reads from cache will not be effective. That means forcing a cache miss for each of, generally, hundreds of thousands of reads to the same address. If the PMU can be used to detect those cache misses, it might be able to detect — and mitigate — Rowhammer attacks. 

The patch is evolving rapidly as this is being written; the [current version](<https://lwn.net/Articles/704926/>) takes the form of a "`nohammer`" kernel module. It has a (currently hardwired) parameter called `dram_max_utilization_factor`, which determines the maximum cache-miss rate allowed in the system. If it is set to 8 (the default), then the `nohammer` module will trigger if the cache-miss rate exceeds 1/8 of the theoretical maximum. When that happens, the CPU will be forced to delay for a period long enough to allow the next DRAM refresh to run; 64ms by default. In theory, this delay should slow down a Rowhammer attack enough to make it ineffective. 

It's a nice theory, but it still suffers from a number of practical problems at this point. To begin with, a 64ms hard delay will add a huge latency to anything the affected CPU is supposed to be doing. If it happens with any frequency at all, it will be noticed, even on systems that are not highly latency-sensitive. Ingo Molnar has [suggested](<https://lwn.net/Articles/704927/>) making the delay shorter and more frequent; that would reduce the maximum imposed latency, but doesn't change the overall nature of the defense. 

The PMU can detect a high rate of cache misses, but it cannot tell the kernel whether all of those misses involved the same address or not. So it could be triggered by an application that is, for example, reading quickly through a large array of data in memory. Thus, it seems entirely plausible that a number of legitimate workloads will generate high rates of cache misses over time that will be mistaken for Rowhammer attacks. Those workloads will be penalized severely by this patch, for no actual gain. That will quickly lead to people turning the Rowhammer defense off. 

The PMU is a per-CPU mechanism, but memory is globally accessible in a multiprocessor system. The patch has some tests for an attack that is conducted by two CPUs simultaneously, but does not scale well to systems with more processors than that. It's not entirely clear how it can be made to work in a setting where, say, eight processors are all pounding the same location simultaneously. 

Finally, Mark Rutland [raised an important point](<https://lwn.net/Articles/704928/>): this mechanism depends entirely on counting cache misses. If the attacker is able to obtain an uncached memory mapping, all operations on that memory will bypass the cache entirely and will not be counted. It would appear that Drammer makes use of just such a mapping, so this module may well not be an effective defense against it. Detecting attacks against uncached memory could prove to be a much harder problem. 

So it is far too soon to say that the kernel has a useful defense against Rowhammer attacks. But this work shows that, when one is willing to pay the price, a defense might just be possible, at least for some types of attacks. That is an improvement over a world where the only real defense is to buy new hardware — once the vendors get around to producing Rowhammer-resistant systems. It will be interesting to watch where this work goes and how effective it becomes.
