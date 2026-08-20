---
title: Realtime KVM
url: https://lwn.net/Articles/656807/
date: "September 10, 2015"
category: "KVM; Virtualization-KVM"
author: "September 10, 2015 This article was contributed by Paolo Bonzini KVM Forum"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

September 10, 2015

This article was contributed by Paolo Bonzini

* * *

[KVM Forum](<https://lwn.net/Archives/ConferenceByYear/#2015-KVM_Forum>)

Realtime virtualization may sound like an oxymoron to some, but (with some caveats) it actually works and is yet another proof of the flexibility of the Linux kernel. The first two presentations at [KVM Forum 2015](<http://events.linuxfoundation.org/events/kvm-forum>) looked at realtime KVM from the ground up. The speakers were Rik van Riel, who covered the kernel side of the work ([YouTube video](<http://www.youtube.com/watch?v=cZ5aTHeDLDE>) and [slides [PDF]](<http://www.linux-kvm.org/images/2/24/01x02-Rik_van_Riel-KVM_realtime.pdf>)) and Jan Kiszka, who explained how to configure the hosts and how to manage realtime virtual machines ([YouTube video](<https://www.youtube.com/watch?v=SyhfctYqjc8>) and [slides [PDF]](<http://www.linux-kvm.org/images/0/0d/01x03-Jan_Kiszska-KVM_RT_for_masses.pdf>)). This article recaps both talks, beginning with Van Riel's. 

#### The `PREEMPT_RT` kernel

Realtime is about determinism, not speed. Realtime workloads are those where missing deadlines is bad: it results in voice breaking up in telecommunications equipment, missed opportunities in stock trading, and exploding rockets in vehicle control and avionics. These applications can have thousands of deadlines a second; the maximum allowed response time can be as low as a few dozen microseconds, and it has to be met 99.999% of the time, if not ... just always. Speed is useful, but guaranteeing this kind of latency bound almost always results in lower throughput. 

Nearly every latency source in a system comes from the kernel. For example, a driver could disable interrupts and prevent high-priority programs from being scheduled. Spinlocks are another cause of latency in a non-realtime kernel, because Linux cannot `schedule()` while holding a spinlock. These issues can be controlled by running a kernel built with `PREEMPT_RT`, the realtime kernel patch set. A `PREEMPT_RT` kernel tries hard to make every part of the Linux kernel preemptible, except for short sections of code. 

Most of the required changes have been merged into Linus's kernel tree: kernel preemption support, priority inheritance, high-resolution timers, support for interrupt handling in threads, annotation of "raw" spinlocks, and `NO_HZ_FULL` mode. The `PREEMPT_RT` patch, while still large, has to do much less than it used to. The main three things it does are: turn non-raw spinlocks into mutexes with priority inheritance, actually run all interrupt handlers in threads so that realtime tasks can preempt them, and an [RCU implementation](<https://lwn.net/Articles/253651/>) that supports preemption. 

The main remaining problem is in firmware. System management interrupts (SMIs) for x86 take care of things such as fan speed, even on servers. SMIs cannot be blocked by the operating system and can take up to milliseconds to run in extreme cases. During this time, the operating system is completely blocked from running. There is no solution other than buying hardware that behaves well. A kernel module, `hwlatdetect`, can help detect the problem; it blocks interrupts on a CPU, looks for unexpected latency spikes, and uses model-specific registers (MSRs) to correlate the spikes to SMIs. 

#### Realtime virtualization, really?

Now, realtime virtualization may sound implausible, but it can be done. Of course, there are problems: for example, the priority of the tasks in the virtual machine (VM) is not visible to the host and neither are lock holders inside a guest. This limits the scheduler's flexibility and prevents priority inheritance, so all of the virtual CPUs (VCPUs) have to be placed at a very high priority. Only [`ksoftirqd`](<https://lwn.net/Articles/520076/>) has a higher priority, since it delivers interrupts to the virtual CPUs. In order to avoid starving the host, systems have to be partitioned between CPUs running system tasks and isolated CPUs (marked with the `isolcpus` and `nohz_full` kernel command-line arguments) running realtime guests. The guest has to be partitioned in the same way between realtime VCPUs and those that run generic tasks. The latter could occasionally cause exits to the host user space, which are potentially long and--much like SMIs on bare metal--prevent the guest scheduler from running. 

Thus, a virtualized realtime guest uses more resources than the same workload running on bare-metal, and those resources have to be dedicated to a particular guest. But this can be an acceptable price to pay for the improved isolation, manageability, and hardware compatibility that virtualization provides. In addition, lately each generation of processors has made more and more cores available within one CPU socket; Moore's Law seems to be compensating for this problem, at least for now. 

Once the design of realtime KVM was worked out as above, the remaining piece is to fix the bugs. A lot of the fixes were either not specific to KVM, or not specific to `PREEMPT_RT`, so they will benefit all real-time users and all virtualization users. For example, RCU was changed to have an [extended quiescent state](<https://lwn.net/Articles/636123/>) while the guest runs. `NOHZ_FULL` support was extended to disable the timer tick altogether when running a `SCHED_FIFO` (realtime) task. In this case, that task will not be rescheduled, because anything with a higher priority would have already preempted it, so the timer tick is not needed. A few knobs were added to disable unnecessary KVM features that can introduce latency, such as synchronization of time from the host to the guest; this can take several microseconds and the solution is simply to run `ntpd` in the guest. 

Virtualization overhead can be limited by using `PREEMPT_RT`'s "[simple wait queues](<https://lwn.net/Articles/577370/>)" instead of the full-blown Linux wait queues. These only take locks for a bounded time so that the length of the operations is also bounded (wakeups often happen from interrupt handlers, so their cost directly affects latency). Merging simple wait queues in the mainline kernel [is being discussed](<https://lwn.net/Articles/653586/>). 

Another trick is to schedule KVM's timers a little in advance to compensate for the overhead of injecting virtual interrupts. It takes a few microseconds for the hypervisor to pass an interrupt down to the guest, and a parameter in the `kvm` kernel module allows for tuning the adjustment based on the guest's benchmarked latency. 

And finally, new processor technology can help too. This is the case for Intel's "[Cache Allocation Technology](<http://www.intel.com/content/www/us/en/communications/cache-monitoring-cache-allocation-technologies.html>)" (CAT), available on some Haswell CPUs. The combined cost of loads from DRAM and TLB misses can cause a single uncached context switch to add up to over 50 microseconds. CAT allows reserving parts of the cache to specific applications, preventing one workload from evicting another workload from the cache, and it is controlled nicely with a control-groups-based interface. The patches, however, have not yet been included in Linux. 

The results, measured with [cyclictest](<https://rt.wiki.kernel.org/index.php/Cyclictest>), are surprisingly good. Bare-metal latencies are less than 2 microseconds, but KVM's measurement of 6-microsecond latencies is also a very good result. To achieve these numbers, of course, the system needs to be carefully set up to avoid all kinds of high-latency system operations: no CPU frequency changes, no CPU hotplug, no loading or unloading of kernel modules, and no swapping. The applications also have to be tuned to avoid slow devices (e.g. disks or sound devices) except in non-realtime helper programs. So deploying realtime KVM requires deep knowledge of the system (for example, to ensure the time stamp counter is stable and the system will never fall back to another clock source) and the workload. Some new bottlenecks will be found as people use realtime KVM more, but the work on the kernel side is, in general, proceeding well. 

#### "Can I have this in my cloud?"

At this point, Van Riel left the stage to Kiszka, who talked more about the host configuration, how to automate it, and how to manage the systems with libvirt and OpenStack. 

Kiszka is a long-time KVM contributor who works for Siemens. He started using KVM many years ago to tackle hardware-compatibility [problems with legacy software [PDF]](<http://www.linux-kvm.org/images/f/f3/2010-kvm-forum-Embedded-KVM.pdf>). He has been [toying with realtime KVM [YouTube]](<https://www.youtube.com/watch?v=zIDwc6uDszY>) for several years, and people are now asking: "Can I have this in my cloud?". 

The answer is "yes", but there are some restrictions. This is not something for a public cloud, of course. Doing realtime control for an industrial plant will not go well if you need to do I/O from some data center far away. "The cloud" here is a private cloud with a fast Ethernet link between the industrial process and the virtual machine. Many features of a cloud environment will also be left behind, because they do not provide deterministic latencies. For example, the realtime path must not use disks or live migration, but this is generally not a problem. 

In going beyond the basic configuration that Van Riel had explained, the first thing to look at is networking. Most of QEMU is still protected by a "big QEMU lock", and device passthrough has latency problems too. While progress is being made on these fronts, it's already possible to use a paravirtualized device (virtio-net) together with a non-QEMU backend. 

KVM supports two such virtio-net backends, namely vhost-net and vhost-user. vhost-net lies in the kernel; it connects a TAP device from the Linux network stack to a virtio-net device in a virtual machine. However, it does not have acceptable latency, yet, either. vhost-user, instead, lets any user-space process provide networking, and can be used together with specialized network libraries. 

Examples of realtime-capable network libraries include [Data Plane Development Kit](<http://dpdk.org/>) (DPDK) or [SnabbSwitch](<https://github.com/SnabbCo/snabbswitch/blob/master/README.md>). These alternative stacks opt for an aggressive polling strategy; this reduces the amount of event signaling and, as consequence, latency as well. Kiszka's set up uses DPDK as a vhost-user client; of course, it runs at a realtime priority too. For the client to deliver interrupts to VCPUs in a timely fashion, it has to be placed at a higher priority than the VCPU threads. 

Kiszka's application does not have high packet rates, so a single physical CPU is enough to run the switch for all the network interfaces in the systems; more demanding applications might require one physical CPU for each interface. 

After prototyping realtime virtualization in the lab, moving it to the data center requires a lot more work. There are hundreds of VMs and many different networks, some of them realtime and some not; that needs to managed and accounted for flexibly. This requires a cloud-management stack, so OpenStack was chosen and extended with realtime capabilities. The reference architecture then includes (from the bottom up): the `PREEMPT_RT` kernel, QEMU (which has to be there for the guest's non-realtime tasks and to set up the vhost-user switch), the DPDK-based switch, libvirt, and OpenStack. Each host, or "compute node", is set up with isolated physical CPUs as explained in the first half of the talk. IRQ affinities also have to be set explicitly (or through the irqbalance daemon) because, by default, they do not respect the kernel's `isolcpus` setting. But, depending on the workload, little tuning may be needed and, in any case, the setup is easily replicated if there are many similar hosts. There is also a tool called [partrt](<https://github.com/OpenEneaLinux/rt-tools>) that helps to set up isolation. 

#### Libvirt and OpenStack

Higher up comes libvirt, which doesn't require much policy, as it only executes commands from the higher layers. All required tunables are available in libvirt 1.2.13: setting the scheduling parameters (policy, priority, pinning to physical CPUs), asking QEMU to `mlock()` all guest RAM, and starting VMs connected to vhost-user processes. The consumer for these parameters is OpenStack's compute-node-handling [Nova](<https://wiki.openstack.org/wiki/Nova>) component. 

Nova can already be configured to enable VCPU pinning and dedicated physical CPUs. Other settings, though, are missing in OpenStack, and are being discussed in [a blueprint](<https://review.openstack.org/#/c/139688>). While it is not yet complete (for example it doesn't support associating non-realtime physical CPUs to non-realtime QEMU threads), the blueprint will enable the usage of the remaining libvirt knobs. Patches for it are being discussed and the target is OpenStack's "Mitaka" release, due in the first half of 2016. Kiszka's team is integrating the patches into its deployment; the team will come up with extensions to the patches and to the blueprint. 

OpenStack also controls networking through the [Neutron](<https://wiki.openstack.org/wiki/Neutron>) component. However, realtime networks tend to be special: they might not use TCP/IP at all, and Neutron really wants to manage its networks in its own way. Siemens is thus introducing "unmanaged" networks (which do no DHCP and possibly even no IP) into Neutron. 

All in all, work in the higher layers of the stack is mostly about standardizing the basic setup of realtime-capable compute nodes, and a lot of the work will be about improving the tuning process in tools such as partrt. As mentioned during the Q&A session, [tuned](<https://fedorahosted.org/tuned/>) is also being extended to support a realtime tuning profile. However, Kiszka also plans to take another look lower in the stack; the newest chipsets have [functionality](<http://www.intel.com/content/www/us/en/embedded/technology/virtualization/vt-directed-io-spec.html>) that eliminates interrupt latency introduced when assigning devices directly to VMs by directly routing the interrupt without involving the hypervisor. In addition, Kiszka's [older work [PDF]](<http://www.linux-kvm.org/images/2/24/2012-forum-Jan-Kiszka-BQL.pdf>) to let QEMU emulate realtime devices could be brought back sometime in the future.
