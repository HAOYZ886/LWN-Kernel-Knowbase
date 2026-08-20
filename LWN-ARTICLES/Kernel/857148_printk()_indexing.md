---
title: printk() indexing
url: https://lwn.net/Articles/857148/
date: "May 27, 2021"
category: Messages; printk
author: "By Jonathan Corbet May 27, 2021"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
May 27, 2021

When kernel developers want to communicate something about the state of a running kernel, they tend to use `printk()`; that results in a log entry that is intended — with varying success — to be human-readable. As it happens, though, the consumers of that information are often not human; the kernel's log output is also read by automated monitoring systems that are looking for problems. The result is an impedance mismatch that often ends with the monitoring system missing important messages. The [`printk()` format indexing patch set](<https://lwn.net/ml/linux-kernel/cover.1621338324.git.chris@chrisdown.name/>) is the latest of many attempts to improve this situation. 

Monitoring systems are installed by administrators who want to know when there is a problem with the systems they manage. So, for example, if the CPU bursts into flames, and the administrator doesn't happen to be in the room to witness the event, they would at least like to receive an alert telling them to call their hardware vendor and the fire department, probably in that order. To produce this alert, the monitoring system will be watching the kernel log for the "CPU on fire" message printed by [the relevant code](<https://elixir.bootlin.com/linux/v5.12.6/source/arch/x86/kernel/cpu/mce/p5.c#L23>) in the kernel. If all goes well, the message will escape before the CPU melts and the replacement system can be ordered in a timely manner. 

Then, one day, along comes a well-meaning contributor of trivial patches who decides that the message would be more aesthetically pleasing if it read "CPU in flames" instead. The resulting patch will be duly merged by the maintainer after a rigorous review process and shipped in a stable kernel update; the administrator, upon seeing the improved message, will be overcome by the beauty of the kernel's expression. 

But that will only happen _if_ the administrator sees this message. Unfortunately, the monitoring system is still looking for the old "CPU on fire" message; it is not only unmoved by the new phrasing, but it also misses the message entirely. So no alarm is sent, and the administrator continues the background project of testing the latency of a favorite social-networking site until the sprinklers go off and make a huge mess. By then it is too late in the day to order a replacement system and the service goes down. 

System administrators hate it when that happens. 

The problem is that, while messages emitted by way of `printk()` (and all the functions layered on top of it) are consumed by user-space tools, those messages have never really been considered to be a part of the kernel's ABI, so kernel developers feel free to change (or delete) them at any time. That creates trouble for anybody who has created a massive collection of regular expressions meant to detect the messages of interest; it also thwarts any effort at translating messages into other languages. 

There have been many attempts to address this problem over the years. The 2011 Kernel Summit featured [a combative session](<https://lwn.net/Articles/464276/>) around a proposal to add a 128-bit binary tag to every kernel message. That idea, like all others before and after, failed to get anywhere near the mainline. The kernel community has, quite simply, never felt the need to impose any structure on the logging information it puts out via `printk()`; the additional effort that would be required does not seem to be worth the cost. 

The latest proposal, posted by Chris Down, does not attempt to impose any such structure. Instead, it simply collects every `printk()` format string built into the kernel and makes them all available via a file in debugfs. Specifically, `printk()` becomes a wrapper macro that declares a new structure to hold the format information: 

```
struct pi_entry {
    	const char *fmt;
    	const char *func;
    	const char *file;
    	unsigned int line;
    	const char *level;
    	const char *pre_fmt;
    	const char *post_fmt;
        };
```

This structure holds the format string passed to `printk()`, along with the log level, information on where in the source this call is to be found, and "pre" and "post" data that is added by various wrappers (such as the `dev_printk()` functions). This structure is placed into a special section (`.printk_index`) of the built kernel. The wrapper also calls the original `printk()` — now called `_printk()` — to output the message, of course. 

When the kernel is built, all of those `pi_entry` structures are placed together in the `.printk_index` section. After mounting debugfs, the administrator can look at `printk/index/vmlinux`, which contains the entire set of formats; there are also files under `printk/index` for each loadable module. Paging through these files, which will contain thousands of format strings, is likely to be almost as interesting as the administrator's social-media feed, but that is not really the intended use. Instead, monitoring systems can use this information to make sure that all of their tests still match messages that the kernel will actually emit. If the "CPU on fire" test no longer finds a match, then either the CPU has been rendered fireproof or the message has been changed. Should the latter prove to be the case, the test can be updated accordingly. 

There are a couple of interesting questions that have not, as of this writing, been raised in the minimal review that this patch set has seen. The first would be: why does this information need to be built into the kernel? It could be placed into a separate file that would not require memory on a running system. The answer, perhaps, is that this mechanism makes it easy to keep the correct set of format strings with the kernel as it is deployed across systems. 

The other is that debugfs is explicitly not meant for production systems, but this feature looks like it is meant to be used with production kernels. Should this mechanism be accepted into the mainline, it may have to find a home in a more stable setting, such as `/sys`. 

Whether it will be accepted remains to be seen, though. Since this mechanism does not require any additional effort from kernel developers, and since it imposes no cost when it is turned off, it might encounter less resistance than the previous efforts to ease automated parsing of kernel log messages. If so, monitoring systems will not be shielded from changing kernel log messages, but they will at least know when their tests need updating.
