---
title: Keeping printk() under control
url: https://lwn.net/Articles/66091/
date: "January 13, 2004"
category: printk ratelimit
author: 
---

Log messages from the kernel can often be an indispensable aid in tracking down problems or generally figuring out what is going on inside the system. As most system administrators find out sooner or later, however, kernel logging can also become a problem in its own right. If a situation develops which causes the kernel to continually spew out logging information, disks can fill up and log messages can be lost. What can be worse, however, is when log messages sent to the console cause the kernel to spend all of its time just scrolling the console frame buffer. In this case, the system can become completely unresponsive. The logging code already tries to mitigate this problem by detecting and suppressing streams of identical messages. That simple mechanism breaks down, however, when the messages being logged differ from each other. 

As a way of improving the situation, Anton Blanchard has put together a new rate limiting scheme which has found its way into the -mm patch tree. This code, which is derived from a rate limiting mechanism used in the networking subsystem, does not automatically solve the problem, since it requires explicit changes to code which could generate message floods. Such code is often easy to identify, however, and easy to fix. 

The patch adds a new function: 

```
int printk_ratelimit(void);
```

Code which could generate lots of messages should call `printk_ratelimit()` and only call `printk()` if the return value is nonzero. Thus, `printk_ratelimit()` returns a failure status if rate limiting is currently in effect and `printk()` output should be avoided. 

By default, the code limits messages to one every five seconds. It will, however, allow ten messages through in a short period before the rate limiting clamps down on the rest. These values are, of course, tuneable via sysctl parameters. 

A mechanism like this is only useful if it is used throughout the code. Core kernel code can be fixed up relatively easily; the patch includes a fix for the page allocator, for example. The source of message floods, however, is often a driver which want to be sure that its "my device has joined the Dark Side" messages are heard. Fixing all of those is a daunting task, but even a partial solution leaves the kernel less susceptible to this particular problem than before.
