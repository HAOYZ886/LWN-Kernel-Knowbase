---
title: RCU mistakes
url: https://lwn.net/Articles/366717/
date: "December 15, 2009"
category: "Read-copy-update"
author: "By Jonathan Corbet December 15, 2009"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
December 15, 2009

Thomas Gleixner has set himself the task of getting rid of the messy rwlock called `tasklist_lock`; in many cases, the solution is to use read-copy-update (RCU) instead. In the process, he [found some problems](<https://lwn.net/Articles/366718/>) with how some code uses RCU. They merit a quick look, since these problems may occur elsewhere, and may reflect an outdated understanding of how RCU works. 

The core idea behind RCU is to delay the freeing of obsoleted, globally-visible data until it is known that no users of that data exist. Traditionally, this has been accomplished by (1) requiring that all uses of RCU-protected data be in atomic code, and (2) not freeing any old data until every CPU in the system has scheduled at least once after that data was replaced by an updated copy. Since atomic code cannot schedule, this set of rules is sufficient to know that no references to the old data exist. 

Needless to say, code working with RCU-protected data must have preemption disabled - otherwise the processor could schedule while a reference to that data still exists. So the `rcu_read_lock()` primitive has traditionally disabled preemption. Based on the code Thomas found, that seems to have led to the conclusion that disabling preemption is sufficient for code using RCU. 

The problem is that [newer forms of RCU](<http://lwn.net/Articles/305782/>) use a more sophisticated batching mechanism to track references to RCU-protected data. This change was necessary to make RCU scale better, especially in situations (realtime, for example) where disabling preemption is undesirable. When using hierarchical (or "tree") RCU, code which simply disables preemption before accessing RCU-protected data will have ugly race conditions. So it's important to always use `rcu_read_lock()` when working with such data. Unfortunately, this is a hard rule to enforce in an automated way, so programmers will simply have to remember it.
