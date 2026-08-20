---
title: "Security topics: io_uring, VM attestation, and random-reseed notifications"
url: https://lwn.net/Articles/943239/
date: "September 4, 2023"
category: "Confidential computing; Random numbers; io uring-Security"
author: "By Jonathan Corbet September 4, 2023"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
September 4, 2023

The kernel-development community has recently been discussing a number of independent patches, each of which is intended to help improve the security of deployed systems in some way. They touch on a number of areas within the kernel, including the question of how widely io_uring should be available, how to allow virtual machines to attest to their integrity, and the best way to inform applications when their random-number generators need to be reseeded. 

#### Disabling io_uring

The [io_uring interface](<https://lwn.net/Articles/776703/>) has been a boon to users striving for the best performance with I/O-heavy workloads; it has finally given Linux an approach to asynchronous I/O (and more) that the community can be proud of. It has also brought a number of security-related bugs, to the point the Google recently [described it](<https://security.googleblog.com/2023/06/learnings-from-kctf-vrps-42-linux.html>) as being ""safe only for use by trusted components"". It is thus not surprising that somebody (Matteo Rizzo, in this case) has put together [a patch](<https://lwn.net/ml/linux-kernel/x49y1i42j1z.fsf@segfault.boston.devel.redhat.com/>) allowing the system administrator to disable io_uring entirely. 

This patch adds a new sysctl knob (`kernel.io_uring_disabled`) that controls the availability of the io_uring feature. At the knob's default value of zero, io_uring remains available as always. Setting it to one disables it for unprivileged users, where "privileged" is defined as having the `CAP_SYS_ADMIN` capability. In response to [a request](<https://lwn.net/ml/linux-kernel/20230726174549.cg4jgx2d33fom4rb@awork3.anarazel.de/>) from Andres Freund after a previous posting, Rizzo added another knob, `kernel.io_uring_group`, that can be set with a group number; any process that is a member of the indicated group is also allowed to use io_uring. Finally, setting `kernel.io_uring_disabled` to two turns the feature off entirely. 

After five revisions, the patch seems about ready to go into the mainline; there does not seem to be any real opposition to it. One might wonder how long it will really be useful, though. As Ben Hawkes recently [wrote](<https://blog.isosceles.com/what-is-a-good-linux-kernel-bug/>), the bulk of the io_uring problems may have already been found: 

> The era of io_uring is probably coming to an end, but it's been a very popular area of research recently. It reminds me of the gold rush around unprivileged user namespaces. Basically these complex new kernel features are consistently more bug-prone than we'd like, and this pattern seems to repeat itself every few years. 

In the case of io_uring, perhaps the worst problems have been found and the stream of vulnerabilities will begin to taper off. 

#### Virtual-machine attestation

The field of confidential computing has put a lot of effort into the ability to run virtual machines that cannot be compromised or spied upon, even by the host computer on which those machines are run. Getting to that point requires a lot of system hardening, use of encryption, and hardware that provides features (such as encrypted memory) to protect virtual machines from the surrounding world. All that work will be for nothing, though, if a virtual machine is compromised in some way: if, for example, its data has been tampered with, or if the hardware features it is depending on are not really there. 

Users of confidential-computing systems tend to start them and, after convincing themselves that all is well, entrusting them with the encryption keys or other secrets they need to get their job done. For a virtual machine, convincing an orchestration system is a matter of using the available integrity-measurement mechanisms and having the CPU attest to its own integrity using a secret key buried deeply inside. All of this information can be signed by a device like a trusted platform module, then passed out of the machine, where it can be verified externally. 

Numerous vendors are working on this functionality and, naturally, each is solving the problem in its own way. This, as Dan Williams noted in [this patch series](<https://lwn.net/ml/linux-kernel/169342399185.3934343.3035845348326944519.stgit@dwillia2-xfh.jf.intel.com/>), is not the best way forward: 

> The approach of adding adding new char devs and new ioctls, for what amounts to the same logical functionality with minor formatting differences across vendors, is untenable. Common concepts and the community benefit from common infrastructure. 

Williams is working to provide that infrastructure. The result is a configfs interface where the orchestration system can create a directory, write nonce data to a special file (called `inblob`). The virtual machine will then read the nonce data, incorporate it into its attestation report, and make it available to be read from `outblob`. The orchestrator can then verify the signatures and nonce data; if everything checks out, the machine should be safe to use. 

It's worth noting that this proposal says nothing about the format of the data written to and read from these configfs files; they are still specific to the confidential-computing mechanism that is in use. There is, evidently, a discussion underway concerning the standardization of this data, but it is not clear if or when that will happen. Meanwhile, though, there will at least be a uniform interface for working with this information. 

#### Random reseeding

The kernel's random-number generator is meant to be fast, but it is still not fast enough for some users. In such cases, it is common to implement a pseudo-random-number generator in user space, which is seeded from the kernel at application startup. That can work well, but there is a problem: sometimes the random seed may be in danger of compromise and in need of replacement. This can happen, for example, if a virtual machine is snapshotted and later restored, resulting in two machines generating the same "random" number series from the same seed. This problem was [addressed in the kernel](<https://lwn.net/Articles/887207/>) in 2022, but it remains for user space. 

The kernel is aware of events that may require reseeding a random-number generator; it is just a matter of making that information available to interested processes in user space. A system call to check whether reseeding is necessary could be added, but that would defeat the purpose of using the user-space generator in the first place; something faster is needed. 

The approach currently under consideration can be seen in [this short patch series](<https://lwn.net/ml/linux-kernel/20230823090107.65749-1-bchalios@amazon.es/>) from Babis Chalios. It allows a process to open `/dev/random`, invoke a new `ioctl()` to get a special-purpose file descriptor, then pass that descriptor to `mmap()` to map a single page of shared memory into the process's address space. That page contains a 32-bit value split into two fields: an eight-bit "notifier ID" and a 24-bit "epoch counter". 

There are numerous notifiers in the kernel that may detect and signal the need to reseed the random-number generator; each of these is assigned a unique ID. Examples of notifiers might include the virtual-machine snapshot mechanism or a periodic timer. Whenever a notifier decides that a reseed is warranted, it increments the epoch counter and writes its own ID into the notifier-ID field; the combination of the two values ensures that the full 32-bit value will change with every update regardless of any races between notifiers. With this mechanism in place, a user-space process need only read this value before generating a random number; if it has changed since the last read, a reseed should happen before anything else. 

Some discussion on the details of the reporting format are still ongoing (Greg Kroah-Hartman [suggested](<https://lwn.net/ml/linux-kernel/2023082322-semester-heave-e5bc@gregkh/>) using two 32-bit values), but otherwise this mechanism, which was evidently hashed out at the 2022 Linux Plumbers Conference, appears to be uncontroversial. Unless something surprising happens, reseed notifications should be ready for merging by the time the 6.7 merge window opens.
