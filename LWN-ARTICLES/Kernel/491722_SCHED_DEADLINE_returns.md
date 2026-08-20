---
title: SCHED_DEADLINE returns
url: https://lwn.net/Articles/491722/
date: "April 11, 2012"
category: "Scheduler-Deadline scheduling"
author: "By Jonathan Corbet April 11, 2012"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
April 11, 2012

[Deadline scheduling](<https://lwn.net/Articles/412745/>) has been discussed on these pages a number of times. In short: rather than using priorities, a deadline scheduler characterizes each process with a maximum CPU time required and the deadline by which it must receive that CPU time. A properly-written deadline scheduler can ensure that every process meets its deadlines while refusing to take work that would cause any deadlines to be missed. Patches adding a deadline scheduler to Linux have existed for a few years, but their progress toward the mainline has been slow; recently that progress has been _very_ slow since the principal developer involved has moved on to other projects. 

The deadline scheduler patches now have a new developer in the form of Juri Lelli; Juri has posted [a new version of the patches](<https://lwn.net/Articles/490944/>) to restart the discussion. The changes from the last time around are mostly minor: review comments addressed and an improved migration mechanism added. The plan is to continue to fill in the gaps required to make the deadline scheduler production-worthy and, eventually, to get it into the mainline. To that end, there is a new git repository, an application designed to test deadline scheduling, and [a new mailing list](<http://feanor.sssup.it/pipermail/linux-dl/>). One assumes Juri would be most appreciative of testing and patches.
