---
title: "Uprobes returns - again"
url: https://lwn.net/Articles/387257/
date: "May 11, 2010"
category: Tracing; Uprobes
author: "By Jonathan Corbet May 11, 2010"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
May 11, 2010

The Uprobes module is becoming one of the longer-lasting stories in the kernel development community. For a few years now, developers have been trying to get this code - which allows the placement of dynamic tracepoints into user-space programs - into the mainline. We last [looked at Uprobes](<http://lwn.net/Articles/370322/>) back in January; now, as the 2.6.35 merge window approaches, Uprobes is [back for another round](<http://lwn.net/Articles/386654/>). 

At this point, Uprobes has been entirely separated from the utrace layer, which is not a part of this patch series. Utrace is controversial in its own right and has not proved helpful in getting Uprobes merged. Other changes which have been made include the addition of interfaces to the the tracing and perf events subsystem. That means that dynamic probes can be inserted from the command line, then watched using the Ftrace interface or aggregated with perf. 

On the other hand, Uprobes retains the "execute out of line" mechanism for the execution of instructions displaced by probes. XOL works, but it does so at the cost of injecting a new virtual memory area into the probed process; that is a larger disturbance than some developers would like to see. But the alternative - adding an emulator for those instructions to the kernel - is invasive in different ways. 

Review comments so far have focused on relatively small details. That does not mean that Uprobes will be accepted when the merge window opens, but its chances do seem better than they have in the past.
