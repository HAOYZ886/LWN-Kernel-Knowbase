---
title: Two new ways to read a file quickly
url: https://lwn.net/Articles/813827/
date: "March 6, 2020"
category: "System calls-readfile; io uring"
author: "By Jonathan Corbet March 6, 2020"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
March 6, 2020

System calls on Linux are relatively cheap, though the mitigations for speculative-execution vulnerabilities have made them more expensive than they once were. But even cheap system calls add up if one has to make a large number of them. Thus, developers have been working on ways to avoid system calls for a long time. Currently under discussion is a pair of ways to reduce the number of system calls required to read a file's contents, one of which is rather simpler than the other.
