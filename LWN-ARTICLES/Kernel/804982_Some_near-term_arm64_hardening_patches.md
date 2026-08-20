---
title: "Some near-term arm64 hardening patches"
url: https://lwn.net/Articles/804982/
date: "November 18, 2019"
category: "Security-Kernel hardening"
author: "By Jonathan Corbet November 18, 2019"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
November 18, 2019

The arm64 architecture is found at the core of many, if not most, mobile devices; that means that arm64 devices are destined to be the target of attackers worldwide. That has led to a high level of interest in technologies that can harden these systems. There are currently several such technologies, based in both hardware and software, that are being readied for the arm64 kernel; read on for a survey on what is coming.
