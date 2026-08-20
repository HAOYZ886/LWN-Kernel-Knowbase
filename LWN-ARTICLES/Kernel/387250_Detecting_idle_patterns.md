---
title: Detecting idle patterns
url: https://lwn.net/Articles/387250/
date: "May 11, 2010"
category: Power management
author: "By Jonathan Corbet May 11, 2010"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
May 11, 2010

The [cpuidle](<http://lwn.net/Articles/384146/>) subsystem is charged with putting the CPU into the optimal sleep state when there is nothing for it to do. One of the key inputs into this decision is the next scheduled timer event; that event puts an upper bound on how long the processor can expect to be able to sleep undisturbed. A more distant next timer event suggests that a deeper sleep state is appropriate. 

But timer events are not the only way to wake up a processor; device interrupts will also do that. There are times when hardware can be expected to interrupt well before the next timer expiration, but those times can be hard for the processor to predict. There is seemingly an exception, though: sometimes hardware interrupts are so regular that they become a sort of timer tick in their own right. A moving mouse can generate that sort of pattern; network traffic can do it too. In such situations, the current cpuidle "menu" governor may repeatedly choose the wrong sleep state. 

Arjan van de Ven has come to the rescue with [a simple cpuidle patch](<http://lwn.net/Articles/386990/>) which maintains an array of the last eight actual sleep periods. Whenever it is time to put the processor to sleep, the standard deviation of those sleep periods is calculated; if it is small, then the average sleep is considered to be a better guide to the expected sleep period than the next timer event. 

As machine learning goes, this code is a relatively simple example. But it should be smart enough to catch simple patterns and run the hardware in something closer to an optimal mode.
