---
title: BPF and the realtime patch set
url: https://lwn.net/Articles/802884/
date: "October 23, 2019"
category: BPF; Realtime
author: "By Jake Edge October 23, 2019"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
October 23, 2019

Back in July, Linus Torvalds merged a [patch](<https://lwn.net/ml/linux-kernel/20190715150402.798499167@linutronix.de/>) in the 5.3 merge window that added the `PREEMPT_RT` option to the kernel build-time configuration. That was meant as a signal that the realtime patch set was moving from its longtime status as out-of-tree code to a fully supported kernel feature. As the code behind the configuration option makes its way into the mainline, some friction can be expected; we are seeing a bit of that now with respect to the BPF subsystem. 

The thread started with a patch [posted](<https://lwn.net/ml/bpf/20191017090500.ienqyium2phkxpdo@linutronix.de/>) by Sebastian Andrzej Siewior to the BPF mailing list. The patch mentioned three problems with BPF running when realtime is enabled and added Kconfig directives to only allow BPF to be configured into kernels that did not have `PREEMPT_RT`: 

Disable BPF on PREEMPT_RT because 

  * it allocates and frees memory in atomic context 
  * it uses up_read_non_owner() 
  * BPF_PROG_RUN() expects to be invoked in non-preemptible context 

Siewior said that he had tried to [address the memory allocation problems](<https://lore.kernel.org/bpf/20190410143025.11997-1-bigeasy@linutronix.de/>) ""but I have no idea how to address the other two issues"". In that thread, he also gave an [overview](<https://lore.kernel.org/bpf/20190412161406.ok3b4ooyeab6i3wz@linutronix.de/>) of what is needed to "play nicely" with the realtime patch set. 

Daniel Borkmann [replied](<https://lwn.net/ml/bpf/20191017145358.GA26267@pc-63.home/>) that the simple approach Siewior took would not actually disable all of BPF, as there are other BPF-using subsystems that would not be affected by the change. Siewior [asked for feedback](<https://lwn.net/ml/bpf/20191017154021.ndza4la3hntk4d4o@linutronix.de/>) on one possible way to solve that, but David Miller [made it clear](<https://lwn.net/ml/bpf/20191017.132548.2120028117307856274.davem@davemloft.net/>) that he does not think this approach makes sense: ""Turning off BPF just because PREEMPT_RT is enabled is a non-starter it is absolutely essential functionality for a Linux system at this point."" 

However, as Siewior said, there are fundamental incompatibilities between the implementation of BPF and the needs of the realtime patch set. Thomas Gleixner [provided more detail](<https://lwn.net/ml/bpf/alpine.DEB.2.21.1910172342090.1869@nanos.tec.linutronix.de/>) on the problem areas in the hopes of finding other ways to deal with them: 

  * #1) BPF disables preemption unconditionally with no way to do a proper RT substitution like most other infrastructure in the kernel provides via spinlocks or other locking primitives. 
  * #2) BPF does allocations in atomic contexts, which is a dubious decision even for non RT. That's related to #1 
  * #3) BPF uses the up_read_non_owner() hackery which was only invented to deal with already existing horrors and not meant to be proliferated. 

Miller [replied](<https://lwn.net/ml/bpf/20191017.151335.597242104804050107.davem@davemloft.net/>) that BPF is needed by systemd and the IR drivers already; ""We're moving to the point where even LSM modules will be implemented in bpf."" In his earlier message, he said that turning off BPF would disable any packet sniffing so that `tcpdump` and Wireshark would not function. To a certain extent, he oversold the need for BPF as Gleixner [pointed out](<https://lwn.net/ml/bpf/alpine.DEB.2.21.1910180041430.1869@nanos.tec.linutronix.de/>); Gleixner was running Debian testing with Siewior's patch applied and not encountering any systemd or other difficulties. Furthermore, even though packet sniffing is not using BPF, thus requiring a copy to user space for each packet, it does still work, Gleixner said, so that is not really an argument for requiring BPF either. 

Beyond that, though, he was really looking for feedback on ""how to tackle these issues on a technical level"". Some of that did start to come about in a [sub-thread](<https://lwn.net/ml/bpf/CAADnVQJPJubTx0TxcXnbCfavcQDZeu8VTnYYpa8JYpWw9Ze4qg@mail.gmail.com/>) with BPF maintainer Alexei Starovoitov, who wondered about disabling preemption and noted that he is a ""complete noob in RT"". Gleixner [explained](<https://lwn.net/ml/bpf/alpine.DEB.2.21.1910180152110.1869@nanos.tec.linutronix.de/>) the situation at some length. 

Essentially, the realtime kernel cannot disable preemption or interrupts for arbitrarily long periods of time. The realtime patches substitute realtime-aware locks for the spinlocks and rwlocks that do disable preemption or interrupts in the non-realtime kernel. Those realtime-aware locks can sleep, however, so they cannot be used from within code sections that have explicitly disabled preemption or interrupts. 

As Starovoitov [explained](<https://lwn.net/ml/bpf/20191018055222.cwx5dmj6pppqzcpc@ast-mbp/>), BPF disables preemption ""because of per-cpu maps and per-cpu data structures that are shared between bpf program execution and kernel execution"". But, he said, BPF does not call into code that might sleep, so there should be no problems on that score. But that is only when looking at the BPF code from a non-realtime perspective, Gleixner [said](<https://lwn.net/ml/bpf/alpine.DEB.2.21.1910181256120.1869@nanos.tec.linutronix.de/>); because of the lock substitution, code that does not look like it could sleep actually can sleep since the realtime locks (e.g. sleeping spinlocks) do so. That's what makes using `preempt_disable()` (and `local_irq_disable()`) problematic in the realtime context. He said that the `local_lock()` mechanism in the realtime tree might be a way forward to better handle the explicit preemption disabling in BPF. 

But, he said, there is still the outstanding problem of BPF making calls to [`up_read_non_owner()`](<https://elixir.bootlin.com/linux/v5.3.6/source/kernel/locking/rwsem.c#L1616>), which allows a read-write semaphore (rwsem) to be unlocked by a process that is not the owner of the lock. That breaks the realtime requirement that the locker is the same as the unlocker in order to deal with priority inheritance correctly. 

Starovoitov also said that BPF does not have unbounded runtime within the preemption-disabled sections, since it has a bound on the number of instructions that can be in a BPF program. But the limit on the number of instructions was recently raised from 4096 to one million, which will result in unacceptable preemption-disabled windows as Gleixner [noted](<https://lwn.net/ml/bpf/alpine.DEB.2.21.1910181031040.1869@nanos.tec.linutronix.de/>): 

Assuming a instruction/cycle ratio of 1.0 and a CPU frequency of 2GHz, that's 500us of preempt disabled time. Out of bounds by at least one order of [magnitude] for a lot of RT scenarios. 

Even the earlier limit of 4096 would result in 2µs of preemption-disabled time, which may be problematic Clark Williams [said](<https://lwn.net/ml/bpf/20191018074936.36f15bd1@tagon/>); "[...] "there are some customer cases on the horizon where 2us would be a significant fraction of their max latency"". 

The `local_lock()` scheme [seemed viable](<https://lwn.net/ml/bpf/20191018230540.l6e4jtrlu44hk7q5@ast-mbp/>) to Starovoitov, but he thought the overall approach taken by Siewior's patch was backward: 

But reading your other replies the gradual approach we're discussing here doesn't sound acceptable ? And you guys insist on disabling bpf under RT just to merge some out of tree code ? I find this rude and not acceptable. 

If RT wants to get merged it should be disabled when BPF is on and not the other way around. 

But Gleixner [did not see things that way](<https://lwn.net/ml/bpf/alpine.DEB.2.21.1910201043460.2090@nanos.tec.linutronix.de/>) at all; he noted that he was planning to investigate local locks as a possible way forward and that there was no insistence on anything. In addition, turning off realtime when BPF was enabled was always an option; "[...] "I could have done that right away without even talking to you. That'd have been dishonest and sneaky."" He also lamented that the discussion had degraded to that point. 

For his part, Starovoitov [said](<https://lwn.net/ml/bpf/20191022014324.yfrvdthj6rss742c@ast-mbp.dhcp.thefacebook.com/>) that he simply thinks disabling an existing in-kernel feature in order to ease the path for the realtime patches is likely to ""backfire on RT"". He suggested getting the code upstream without riling other subsystem developers was a better way forward: 

imo it's better to get key RT bits (like local_locks) in without sending contentious patches like the one that sparked this thread. When everyone can see what this local_lock is we can figure out how and when to use it. 

That's where things were left at the time of this writing. It is not clear that the practical effect of which subsystem disables the other makes any real difference to users. For now, users of BPF will not be able to use the realtime patches and vice versa. Fixing the underlying problems that prevent the two from coexisting certainly seems more important than squabbling over who disables who; there will be users who want both in their systems after all. For those who run mainline kernels, though, that definitely cannot happen until realtime gets upstream; once it does, a piecemeal approach to resolving the incompatibilities between BPF and realtime can commence.
