---
title: "A medley of performance-related BPF patches"
url: https://lwn.net/Articles/808503/
date: "January 2, 2020"
category: BPF; Retpoline
author: "By Jonathan Corbet January 2, 2020"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
January 2, 2020

One of the advantages of the in-kernel BPF virtual machine is that it is fast. BPF programs are just-in-time compiled and run directly by the CPU, so there is no interpreter overhead. For many of the intended use cases, though, "fast" can never be quite fast enough. It is thus unsurprising that there are currently a number of patch sets under development that are intended to speed up one aspect or another of using BPF in the system. A few, in particular, seem about ready to hit the mainline.
