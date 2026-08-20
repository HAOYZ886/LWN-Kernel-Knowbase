---
title: Extended error reporting
url: https://lwn.net/Articles/374794/
date: "February 17, 2010"
category: "Development model-User-space ABI; User-space API"
author: "By Jonathan Corbet February 17, 2010"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
February 17, 2010

Linux contains a number of system calls which do complex things; they take large structures as input, operate on significant internal state, and, perhaps, return some sort of complicated output data. The normal status returned from these system calls, however, is compressed down into a single integer called `errno`. Application programmers dealing with certain subsystems (Video4Linux2 being your editor's favorite in this regard) will all be well familiar with the process of trying to figure out what the problem is when the kernel says only "it failed." 

Andi Kleen [describes](<https://lwn.net/Articles/374796/>) the problem this way: 

I always describe that as a the "ed approach to error handling". Instead of giving a error message you just give ?. Just ? happens to be EINVAL in Linux. 

My favourite example of this is the configuration of the networking queueing disciplines, which configure complicated data structures and algorithms and in many cases have tens of different error conditions based on the input parameters -- and they all just report EINVAL. 

It would be nice to provide application developers with better information than this. A brief discussion covered some of the options: 

  * Use `printk()` to put information into the system logfile. This approach is widely used, but it bloats the kernel with string data, risks flooding the logs, and the resulting information may not be easily accessible to an unprivileged programmer. 

  * Extend specific system calls to enable them to provide richer status information. Just adding a new version of `ioctl()` would address many of the worst problems. 

  * Create an `errno`-like mechanism by which any system call could return extended information. That information could be an error string, some sort of special code, or, as Alan Cox [suggested](<https://lwn.net/Articles/374799/>), a pointer to the structure field which caused the problem. 

One could certainly argue that the narrow `errno` mechanism is showing its age and could use an upgrade. Any enhancements, though, would be Linux-specific and non-POSIX, which always tends to limit their uptake. They would also have to be lived with forever, and, thus, would require careful design. So we're unlikely to see a solution in the mainline anytime soon, even if somebody does take up the challenge.
