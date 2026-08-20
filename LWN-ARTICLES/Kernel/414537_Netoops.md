---
title: Netoops
url: https://lwn.net/Articles/414537/
date: "November 10, 2010"
category: "Development tools-Kernel debugging"
author: "By Jonathan Corbet November 10, 2010"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
November 10, 2010

A kernel oops produces a fair amount of data which can be useful in tracking down the source of whatever went wrong. But that data is only useful if it can be captured and examined by somebody who knows how to interpret it. Capturing oops output can be hard; it typically will not make it to any logfiles in persistent storage. That's why we still see oops output posted in the form of a photograph taken of the monitor. Using cameras as a debugging tool can work for a desktop system, but it certainly does not scale to a data center containing thousands of systems. Google is thought to operate a site or two meeting that description, so it's not surprising to see an interest in better management of oops information there. 

Google has had its own oops collection tool running internally for years; that has recently been posted for merging as [netoops](<https://lwn.net/Articles/414031/>). Essentially, netoops is a simple driver which will, in response to a kernel oops, collect the most recent kernel logs and deliver them to a server across the net. The functionality seems useful, but the first version of the patch was questioned: netoops looks somewhat similar to the existing netconsole system, so it wasn't clear that a need for it exists. Why not just add any missing features to netconsole? 

Mike Waychison, who posted the patch, responded with a number of reasons which have since found their way into the changelog. Netoops only sends data on an oops, so it is less hard on network bandwidth. The data is packaged in a more structured manner which is easier for machines and people to parse; that has enabled the creation of a vast internal "oops database" at Google. Netoops can cut off output after the first oops, once again saving bandwidth. And so on. There are enough differences that netconsole maintainer Matt Mackall [agreed](<https://lwn.net/Articles/414541/>) that it made sense for netoops to go in as a separate feature. 

That said, there is clear scope for sharing some code between the two and, perhaps, improving netconsole in the process. The current version of the netoops patch includes new work to bring about that sharing. There seems to be no further opposition, but it's worth noting that Mike, in the patch changelog, notes that he's not entirely happy with either the user-space ABI or the data format. So this might be a good time for others interested in this sort of functionality to have a look and offer their suggestions and/or patches.
