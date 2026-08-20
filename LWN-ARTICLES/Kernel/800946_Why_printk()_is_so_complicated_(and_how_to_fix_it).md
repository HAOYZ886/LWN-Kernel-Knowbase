---
title: Why printk() is so complicated (and how to fix it)
url: https://lwn.net/Articles/800946/
date: "October 3, 2019"
category: Kernel messages; printk
author: "By Jonathan Corbet October 3, 2019 LPC"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
October 3, 2019

* * *

[LPC](<https://lwn.net/Archives/ConferenceByYear/#2019-Linux_Plumbers_Conference>)

The kernel's [`printk()`](<https://www.kernel.org/doc/html/latest/driver-api/basics.html#c.printk>) function seems like it should be relatively simple; all it does is format a string and output it to the kernel logs. That simplicity hides a lot of underlying complexity, though, and that complexity is why kernel developers are still unhappy with `printk()` after 28 years. At the 2019 Linux Plumbers Conference, John Ogness [explained](<https://www.linuxplumbersconf.org/event/4/contributions/290/>) where the complexity in `printk()` comes from and what is being done to improve the situation. 

The core problem, Ogness began, comes from the fact that kernel code must be able to call `printk()` from any context. Calls from atomic context prevent it from blocking; calls from non-maskable interrupts (NMIs) can even rule out the use of spinlocks. At the same time, output from `printk()` is crucial when the kernel runs into trouble; developers do not want to lose any printed messages even if the kernel is crashing or hanging. Those messages should appear on console devices, which may be attached to serial ports, graphic adapters, or network connections. Meanwhile, `printk()` cannot interfere with the normal operation of the system. 

In other words, he summarized, `printk()` is seemingly simple and definitely ubiquitous, but it has to be wired deeply into the system. 

#### The path to the present

Ogness then launched into a detailed history of `printk()`; see [his slides [PDF]](<https://www.linuxplumbersconf.org/event/4/contributions/290/attachments/276/463/lpc2019_jogness_printk.pdf>) for all the details. The first kernel release — v0.01 — included a `printk()` implementation; it was synchronous and simply pushed messages directly to a TTY port with a bit of assembly code. It was reliable, but not particularly scalable; once the kernel started supporting multiple CPUs, things needed to change. 

Version 0.99.7a added console registration; the "log level" abstraction was added in v0.99.13k. The `bust_spinlocks()` mechanism, which prevents waiting for spinlocks when the system is crashing and "goes [![\[John Ogness\]](https://static.lwn.net/images/conf/2019/lpc/JohnOgness-sm.jpg)](<https://lwn.net/Articles/801017/>) against everything everybody has learned", was added in 2.4.0. With 2.4.10, big changes to `printk()` made it asynchronous. By 2.6.26, `printk()` was causing large latency spikes; kernel developers dealt with this problem by ignoring `printk()` in the latency tracer, thus sweeping it under the rug. The 3.4 release added [structured logging](<https://lwn.net/Articles/492125/>), sequence numbers, and the `/dev/kmsg` interface. The "safe buffers" mechanism, used for printing in NMI context, showed up in 4.10. A problem where one CPU could get stuck outputting messages indefinitely was (somewhat) addressed in 4.15. In 5.0, the concept of caller identifiers was added. 

So `printk()` has seen a lot of development over the years, but there are still a number of open issues. One of them is the raw spinlock used to protect the ring buffer; it cannot be taken in NMI context, so `printk()` must output to the lockless safe buffers instead. That will create bogus timestamps when the messages are finally copied to the real ring buffer, can lose messages, and cause the buffers to not be flushed when CPUs don't go offline properly. 

Then, there is the issue of the console drivers, which are slow but are nonetheless called with interrupts disabled. Most console devices are not designed to work in a kernel-panic situation, so they are not reliable when they may be needed most. 

Other problems include the fact that all log levels are treated equally by `printk()`; chatter treated like urgent information can create latency problems, causing some users to restrict the levels that are actually logged. The problem with one CPU being stuck logging forever has been fixed, but the last CPU to come along and take over log output can still be saddled with a lot of work. That makes any `printk()` call potentially expensive. The whole `bust_spinlocks()` mechanism can be described as "ignoring locks and hoping for the best"; there should be a better way, he said. 

#### The better way

The difficulties with `printk()` over the years, Ogness said, come down to the tension between non-interference and reliability. Trying to achieve both goals in the same place has been shown not to work, so a better approach would be to split them apart. Non-interference can be addressed by making `printk()` fully preemptable, making the ring buffer safe in all contexts, and moving console handling to dedicated kernel threads. Reliability, instead, can be achieved by providing a synchronous channel for important messages, an "atomic consoles" concept, and the notion of "emergency messages". 

Both goals depend on support from the `printk()` ring buffer. This buffer has multiple concurrent readers and a single writer; it is stored contiguously in memory and is protected by a special spinlock (the "CPU lock") that can be acquired multiple times on the same CPU. This lock, he said, feels a lot like the old big kernel lock. 

Like any self-respecting kernel-development project, the `printk()` work starts with the creation of [a new ring buffer](<https://lwn.net/ml/linux-kernel/20190807222634.1723-1-john.ogness@linutronix.de/>) meant to address the problems with the current one. It is fully lockless, supporting multiple readers and writers in all contexts. Metadata has been pushed out to a separate descriptor mechanism; it handles tasks like timestamps and sequencing. The ring buffer has some nice features, but it is also complicated, including no less than nine memory-barrier pairs. It is hard to document and hard to review; he is also not convinced that the multiple writer support — which adds a lot of the complexity — is really needed. 

The addition of the per-console kernel threads serves to decouple `printk()` calls from console handling. Each console will now go as fast as it can, and each can have its own log level. Shifting the responsibility for consoles simplifies a lot of things, but leads to some unresolved questions about locking and whether a thread-based implementation can be relied upon to always get the messages out. But reliability, Ogness said, will be handled by other mechanisms; the per-console threads are a non-interference mechanism. 

For reliability, the plan is to add an "atomic console" concept. Consoles with this support would have a `write_atomic()` method that can be expected to work in any context. This method is defined to operate synchronously, meaning that it will slow down the system considerably; it is thus only to be used for emergency messages. The advantage is that there is no longer any need for `bust_spinlocks()` or the global `oops_in_progress` variable. 

The challenge is creating console drivers that can actually implement `write_atomic()`. He did an implementation for consoles attached to an 8250 UART; it was "not trivial". There will almost certainly be a lot of systems that never get atomic-console support, so some other sort of solution will be needed. He said that options include creating a special console that fills a memory area instead, trying to print synchronously outside of atomic context, or just "falling back to the current craziness". 

Associated with atomic consoles is the idea of "emergency messages" that must go out right away. The biggest problem here may be deciding which messages are important enough to warrant that treatment. Log levels are "kind of a gray area" and, he said, not really the way to go. There are only a few situations where `printk()` output is really that important; the right solution might be to hook into macros like `BUG()`. 

Ogness concluded by noting that this work began in February, with the current version having been posted in August. Most of the features described above have been implemented, he said, giving developers "something to play with". 

#### Further discussion

A separate session was held later in the conference; your editor was unfortunately unable to attend. Ogness has posted [a summary](<https://lwn.net/ml/linux-kernel/87k1acz5rx.fsf@linutronix.de/>) of the conclusions that were reached there, though. He thanked the community for its participation in this meeting, which ""certainly saved hundreds of hours of reading/writing emails"". 

From the summary, it seems that [an alternative ring buffer implementation](<https://lwn.net/ml/linux-kernel/20190704103321.10022-1-pmladek@suse.com/>) posted by Petr Mladek will be used instead; it is much simpler and thus easier to review. Ogness has ported the rest of his work to use this buffer and shown that it works. The per-console kernel threads will be used. 

The "emergency messages" idea seems to have been superseded by the idea of an "emergency state" that affects the system as a whole. When the kernel is in that state, all messages will be flushed using the `write_atomic()` callback where it is available. Flushing to consoles without atomic support will not be supported. The CPU lock will remain, but its purpose will be to synchronize the console threads when the system is in the emergency state. 

There will be other changes, including the addition of a `pr_flush()` function that will wait for all messages to be sent out to all consoles. Patches implementing all this work have not yet been posted, but presumably they can be expected soon. 

[Your editor thanks the Linux Foundation, LWN's travel sponsor, for supporting his travel to this event.]
