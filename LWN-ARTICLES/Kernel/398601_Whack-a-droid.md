---
title: "Whack-a-droid"
url: https://lwn.net/Articles/398601/
date: "August 3, 2010"
category: "Power management-Opportunistic suspend"
author: "By Jonathan Corbet August 3, 2010"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
August 3, 2010

Paul McKenney, it seems, is now working with the Linaro project, an assignment which has given him a new interest in power management. He has decided to start off with a bang by [attempting to summarize the suspend blocker discussion](<https://lwn.net/Articles/398602/>) with the goal of really understanding what Android's requirements are. Needless to say, he has kicked off a new, lengthy discussion which has cast the players' positions in a new light. 

To oversimplify: one side seems to believe in addressing power management (and poorly-behaving applications in particular) by shining a light on the problems and fixing them (or applying social pressure to get them fixed) one at a time. This is the approach taken by developers like Arjan van de Ven, who have developed and used tools like PowerTop to great effect. The other side pushes for a more general solution; Paul [describes the difference in view](<https://lwn.net/Articles/398603/>) this way: 

I agree that much progress has been made over the past four years. My laptop's battery life has increased substantially, with roughly half of the increase due to software changes. Taken over all the laptops and PCs out there, this indeed adds up to substantial and valuable savings. 

So, yes, you have done quite well. 

However, your reliance on application-by-application fixes, while good up to a point, is worrisome longer term. The reason for my concern is that computing history has not been kind to those who fail to vigorously automate. The Android guys have offered up one way of automating power efficiency. There might be better ways, but their approach does at the very least deserve a fair hearing -- and no one who read the recent LKML discussion of their approach could possibly mistake it for anything resembling a fair hearing. 

So far, the conversation has not yet really returned to the Android approach; it has stayed more focused on the requirements and whether the "whack-a-mole" approach to power management is sufficient in the long term. Chances are good that Paul will be sending out an updated version of his requirements description sometime in the near future. Then, perhaps, there can be a calm discussion of how those requirements might best be satisfied.
