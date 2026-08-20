---
title: NAPI polling in kernel threads
url: https://lwn.net/Articles/833840/
date: "October 9, 2020"
category: "NAPI; Networking-NAPI"
author: "By Jonathan Corbet October 9, 2020"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
October 9, 2020

Systems that manage large amounts of network traffic end up dedicating a significant part of their available CPU time to the network stack itself. Much of this work is done in software-interrupt context, which can be problematic in a number of ways. That may be about to change, though, once [this patch series](<https://lwn.net/ml/netdev/20201002222514.1159492-1-weiwan@google.com/>) posted by Wei Wang is merged into the mainline. 

Once a packet arrives on a network interface, the kernel must usually perform a fair amount of protocol-processing work before the data in that packet can be delivered to the user-space application that is waiting for it. Once upon a time, the network interface would interrupt the CPU when a packet arrived; the kernel would acknowledge the interrupt, then trigger a software interrupt to perform this processing work. The problem with this approach is that, on busy systems, thousands of packets can arrive every second; handling the corresponding thousands of hardware interrupts can run the system into the ground. 

The solution to this problem [was merged in 2003](<https://lwn.net/Articles/30107/>) in the form of a mechanism that was called, at the time, "new API" or "NAPI". Drivers that support NAPI can disable the packet-reception interrupt most of the time and rely on the network stack to poll for new packets at a frequent interval. Polling may seem inefficient, but on busy systems there will always be new packets by the time the kernel polls for them; the driver can then process all of the waiting packets at once. In this way, one poll can replace dozens of hardware interrupts. 

NAPI has evolved considerably since 2003, but one aspect remains the same: it still runs in software-interrupt mode. These interrupts, once queued by the kernel, will be processed at either the next return from a hardware interrupt or the next return from kernel to user mode. They thus run in an essentially random context, stealing time from whatever unrelated process happens to be running at the time. Software interrupts are hard for system administrators to manage and can create surprising latencies if they run for a long time. For this reason, kernel developers have wanted to reduce or eliminate their use for years; they are an old mechanism that is deeply wired into core parts of the kernel, though, and are hard to get rid of. 

Wang's patch set (which contains work from Paolo Abeni, Felix Fietkau, and Jakub Kicinski) doesn't eliminate software interrupts, but it is a possible step in that direction. With these patches applied, the kernel can optionally (under administrator control) create a separate kernel thread for each NAPI-enabled network interface. After that, NAPI polling will be done in the context of that thread, rather than in a software interrupt. 

The amount of work that needs to be done is essentially unchanged with this patch set, but the change in the way that work is done is significant. Once NAPI polling moves to its own kernel thread, it becomes much more visible and subject to administrator control. A kernel thread can have its priority changed, and it can be bound to a specific set of CPUs; that allows the administrator to adjust how that work is done in relation to the system's user-space workload. Meanwhile, the CPU scheduler will have a better understanding of how much CPU time NAPI polling requires and can avoid overloading the CPUs where it is running. Time spent handling software interrupts, instead, is nearly invisible to the scheduler. 

There aren't a lot of benchmark results posted with the patch set; those that are available indicate a possible slight increase in overhead when the threaded mode is used. Users who process packets at high rates tend to fret over every nanosecond, but even they might find little to quibble about if these results hold. Meanwhile, those users should also see more deterministic scheduling for their user-space code, which is also important. 

The networking developers seem to be generally in favor of this work; Eric Dumazet [indicated](<https://lwn.net/ml/netdev/CANn89iJDM97U15Znrx4k4bOFKunQp7dwJ9mtPwvMmB4S+rSSbA@mail.gmail.com/>) a desire to merge it quickly. This feeling is not unanimous, though; Kicinski, in particular, [dislikes the kernel-thread implementation](<https://lwn.net/ml/netdev/20200930130839.427eafa9@kicinski-fedora-pc1c0hjn.dhcp.thefacebook.com/>). He believes that better performance can be had by using the kernel's workqueue mechanism for the polling rather than threads. Dumazet [answered](<https://lwn.net/ml/netdev/CANn89iK2-Wu8HMkWiD8U3pdRbwj2tjng-4-fJ81zVw_a3R6OqQ@mail.gmail.com/>) that workqueues would not perform well on ""server class platforms"" and indicated a lack of desire to wait to see a new workqueue-based implementation at some point in the future. 

So it appears that this work will be merged soon; it's late for 5.10, so landing in the 5.11 kernel seems likely. It's worth noting that the threaded mode will remain off by default. Making the best use of it will almost certainly require system tuning to ensure that the NAPI threads are able to run without interfering with the workload; for now, administrators who are unwilling or unable to do that tuning are probably well advised to stick with the default, software-interrupt mode. Software interrupts themselves are still not going away anytime soon, but this work may help in the long-term project of moving away from them.
