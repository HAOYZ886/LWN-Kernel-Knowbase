---
title: "Kernel Summit 2005: Clustering"
url: https://lwn.net/Articles/144274/
date: "July 20, 2005"
category: Clusters
author: 
---

From LWN's [2005 Kernel Summit coverage](<https://lwn.net/Articles/KernelSummit2005/>).   
---  
Lars Marowsky-Brée and Bruce Walker led a brief and uncontroversial session on clustering. There was a clustering summit happening simultaneously with the kernel summit; not much information was yet available from that meeting. There had been another summit two weeks before, held in Germany. A couple of issues from that gathering were brought to the kernel summit. 

One of those was dealing with memory pressure, which can be an acute problem in clusters. Lars noted, with obvious amusement, that this problem had been "solved" in prior kernel summit discussions. 

A longstanding question for cluster developers has been: how much work needs to be done in the kernel, and how much can be pushed out to user space? Answers are becoming clearer: cluster configuration and node membership tasks are now clearly user space issues. Distributed lock management appears to be a kernel task, however. 

Along those lines, it was briefly noted that the OCFS2 filesystem group is considering dropping its lock manager implementation in favor of Red Hat's DLM. 

One issue still in need of resolution is cluster-wide process management. There are currently about five different process management implementations out there. Bruce, who works with the [OpenSSI](<http://openssi.org/>) project, floated a proposal which would add a set of hooks to the kernel. Users could then employ the process manager of their choice by way of those hooks. The kernel developers have little patience for the addition of more general-purpose hooks, however; they would rather see people bite the bullet and choose a single solution.
