---
title: Various tweaks to printk()
url: https://lwn.net/Articles/497218/
date: "May 16, 2012"
category: Kernel messages; Messages
author: "By Jonathan Corbet May 16, 2012"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
May 16, 2012

For the most part, the [logging reliability patches](<https://lwn.net/Articles/492125/>) covered here in April have been quietly stabilizing and appear to be set for merging for 3.5. But `printk()` is a heavily-used function, so there are a lot of people with strong opinions on how it should work. Thus the discussion on how `printk()` can be improved has stretched out for some time. The result, so far, is a better understanding of how continuation lines should be handled and, possibly, a new format for timestamps. 

Messages are sent to the system log with `printk()`, but that function has an interesting bit of historical behavior: like `printf()` in user space, `printk()` can be used to send partial lines to the log. Multiple `printk()` calls can be used to produce a single line in the log stream, piece by piece. The patches for 3.5 make `printk()` much more record-oriented internally, but the API does not change. So there is a bit of an impedance mismatch between a record-oriented logging system and its stream-oriented API. That mismatch has been there since the beginning, but it has become more clear over time. 

The mixed nature of kernel logging leads to a bit of an ambiguity, because any message can be either of two things: (1) a new message to be logged or (2) a continuation of a previous log message. The kernel decides which of the two situations holds by remembering whether the previous log message ended with a newline or not. If there was no trailing newline, a new message will be appended to the previous line. 

This approach works much of the time, but it is not without its hazards. In particular, there is nothing that guarantees that two successive `printk()` calls will be executed one right after the other. Even on a uniprocessor system, interrupt handlers can emit messages between two `printk()` calls that are supposed to produce a single line of output. Adding more processors to the system clearly makes the situation worse; there is only one log buffer containing messages from all processors, so it is easy for one processor to jump into the middle of a sequence of `printk()` calls being executed on another. What happens then is not especially pretty: messages get mashed together and corrupted. The result is a log that is harder for humans to read, and which can totally confuse automated log-processing tools. 

This patch set was supposed to be about increasing logging reliability, so that sort of message corruption is not welcome. The original plan devised by developer Kay Sievers was to require an explicit `KERN_CONT` "log level" marker for continuations. In this scheme, every `printk()` call will generate a new log line unless merging has been explicitly requested with the `KERN_CONT` "log level." There is a little problem in that most continuation lines are not so-marked in current kernels, leading to lines being split up; Kay's plan was to audit the kernel and fix all of those calls to work properly in the new scheme. 

Linus [didn't like that idea](<https://lwn.net/Articles/497223/>), saying that things work well as they are now; to him, adding all those `KERN_CONT` markers just represented unnecessary noise. After some back-and-forth, Kay came around to Linus's point of view, but he still wanted to avoid the corruption of messages whenever possible. The result was [a new patch](<https://lwn.net/Articles/497224/>) that tries to explicitly remember partial `printk()` calls and associate them with a specific process. Lines passed to `printk()` will be merged only if they both come from the same process and only if the second line is clearly not the start of a new log message. The end result is not perfect: if two processors try to output partial lines at the same time, at least one of them will be split. But there will be no more joining of unrelated messages, and that seems like a good thing. 

A different branch of the same discussion got into the formatting of timestamps, which will always be present in the new scheme. In current kernels, that timestamp comes in the form of seconds and microseconds since the system booted. But what developers often really want to see is some combination of the absolute time of an event and the relative time from previous events. After some discussion with Sasha Levin, Linus [requested](<https://lwn.net/Articles/497225/>) a format that looks like this: 

```
[May12 11:27] foo
        [May12 11:28] bar
        [  +5.077527] zoot
        [ +10.235225] foo
        [  +0.002971] bar
        [May12 11:29] zoot
        [  +0.003081] foo
```

In other words, events that are relatively far apart in time would be marked with the absolute time with one-minute precision. When things happen more closely in time, the elapsed time between successive events would be printed instead. For any driver developer trying to figure out the relative timing of device-related events, this kind of output format would help to save a lot of mental arithmetic. 

The patches to produce this format have not yet been posted, so it is looking likely that we will not see it in the 3.5 kernel. The rest of the logging work should be there for 3.5, though, taking Linux one small step closer to the sort of structured and reliable logging that many users and developers would like to see.
