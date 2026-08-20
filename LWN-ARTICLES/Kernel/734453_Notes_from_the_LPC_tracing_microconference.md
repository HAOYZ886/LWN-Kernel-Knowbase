---
title: Notes from the LPC tracing microconference
url: https://lwn.net/Articles/734453/
date: "September 21, 2017"
category: BPF; Tracing
author: "By Jonathan Corbet September 21, 2017 Linux Plumbers Conference"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
September 21, 2017

* * *

[Linux Plumbers Conference](<https://lwn.net/Archives/ConferenceByYear/#2017-Linux_Plumbers_Conference>)

The "tracing and BPF" microconference was held on the final day of the 2017 Linux Plumbers Conference; it covered a number of topics relevant to heavy users of kernel and user-space tracing. Read on for a summary of a number of those discussions on topics like BPF introspection, stack traces, kprobes, uprobes, and the Common Trace Format. 

Unfortunately, your editor had to leave the session before it reached its end, so this article does not reflect all of the topics discussed there. For those who are interested, [this Etherpad instance](<https://etherpad.openstack.org/p/LPC2017_Tracing>) contains notes taken by participants at the session. 

#### BPF introspection

Martin Lau started the session by noting that BPF programs typically use maps to communicate with the kernel or user space. It can, however, be hard for an interested person to see what is actually in any given map. A look at a BPF program's [![\[Martin Lau\]](https://static.lwn.net/images/conf/2017/ossna-lpc/MartinLau-sm.jpg)](<https://lwn.net/Articles/734509/>) source will reveal what it is storing in a map, but that source may not always be available. What Lau would like to have is some sort of easy way to pretty-print the contents of a map. 

His proposed solution was to attach a bit of metadata to each map describing the entries found therein. It would look like a C structure definition. The proposed name for this description was the "compact C-type format" or CTF, but that name will almost certainly have to change if this work goes forward, since that acronym is already used for the common trace format. The description would be created with a utility program, then passed into the kernel via the `bpf()` system call that creates the map. The kernel would verify the data and store it, making it available later on request. 

This project may not get that far, though; there was a fair amount of doubt about whether it was really needed. If there are users who truly need a separate description of the contents of a map, it should be possible to manage that information in user space. So, while this idea may not be dead, it will clearly face some headwinds if the work goes forward.
