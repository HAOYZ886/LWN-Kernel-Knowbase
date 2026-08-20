---
title: Security requirements for new kernel features
url: https://lwn.net/Articles/902466/
date: "July 28, 2022"
category: "Development model-Code review; Security-Security modules; io uring-Security"
author: "By Jonathan Corbet July 28, 2022"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
July 28, 2022

The relatively new [io_uring subsystem](<https://lwn.net/Articles/776703/>) has changed the way asynchronous I/O is done on Linux systems and improved performance significantly. It has also, however, begun to run up a record of disagreements with the kernel's security community. A recent discussion about security hooks for the new uring_cmd mechanism shows how easily requirements can be overlooked in a complex system with no overall supervision. 

Most of the operations that can be performed within io_uring follow the usual I/O patterns — open a file, read data, write data, and so on. These operations are the same regardless of the underlying device or filesystem that is doing the work. There always seems to be a need for something special and device-specific, though, and io_uring is no exception. For the kernel as a whole, device-specific operations are made available via [`ioctl()`](<https://man7.org/linux/man-pages/man2/ioctl.2.html>) calls. That system call, however, [has built up a reputation](<https://lwn.net/Articles/897202/>) as a dumping ground for poorly thought-out features, and there is little desire to see its usage spread. 

In early 2021, io_uring maintainer Jens Axboe [floated an idea](<https://lwn.net/Articles/844875/>) for a command passthrough mechanism that would be specific to io_uring. A year and some later, that idea has evolved into uring_cmd, which was pulled into the mainline during the 5.19 merge window. There is a new io_uring operation that, in turn, causes an invocation of the underlying device or filesystem's `uring_cmd()` `file_operations` function. The actual operation to be performed is passed through to that function with no interpretation in the io_uring layer. The [first user](<https://lwn.net/ml/io-uring/20220511054750.20432-1-joshi.k@samsung.com/>) is the NVMe driver, which provides a direct passthrough operation. 

#### Missing security hooks

Just over one year ago, there was [a bit of a disagreement](<https://lwn.net/Articles/858023/>) after the developers of the kernels Linux Security Module (LSM) and auditing subsystems figured out that there were no security or auditing hooks in all of that new io_uring code. That put io_uring operations outside the control of any security module that a given system might be running and made those operations invisible to auditing. Those gaps were filled in, but not before the security developers expressed their unhappiness about how io_uring had been designed and merged without thought for LSM and audit support. 

Given that, one might expect that the addition of a new feature like `uring_cmd` would have seen more involvement from the security community. To an extent, that happened; Luis Chamberlain [posted a patch](<https://lwn.net/ml/io-uring/YiuC1fhEiRdo5bPd@bombadil.infradead.org/>) adding LSM support back in March. In short, it added a new `security_uring_async_cmd()` hook that would be called before passing a command through to the underlying code; it could examine that command and decide whether to allow or deny the operation. There were some disagreements over how well this would work; in particular, Casey Schaufler [complained](<https://lwn.net/ml/io-uring/8adf55db-7bab-f59d-d612-ed906b948d19@schaufler-ca.com/>) that security modules would have to gain an understanding of every device-specific command, which clearly would not scale well. The conversation wound down shortly thereafter. 

When the new feature was pushed into the mainline, there was no LSM support included with it. On July 13, Chamberlain [reposted his patch](<https://lwn.net/ml/linux-block/20220714000536.2250531-1-mcgrof@kernel.org/>) adding the new security hook. Schaufler [was equally unimpressed](<https://lwn.net/ml/linux-block/30dee52c-80e7-f1d9-a2e2-018e7761b8ea@schaufler-ca.com/>) this time around: 

> You're passing the complexity of uring-cmd directly into each and every security module. SELinux, AppArmor, Smack, BPF and every other LSM now needs to know the gory details of everything that might be in any arbitrary subsystem so that it can make a wild guess about what to do. And I thought ioctl was hard to deal with. 

SELinux and audit maintainer Paul Moore [agreed](<https://lwn.net/ml/linux-block/CAHC9VhSjfrMtqy_6+=_=VaCsJKbKU1oj6TKghkue9LrLzO_++w@mail.gmail.com/>) with that assessment. The end result, he said, was that security modules would be unable to distinguish between low-level operations, so they would end up simply enabling all io_uring passthrough commands for any given subsystem or none of them; ""I think we can all agree that is not a good idea"". He later [acknowledged](<https://lwn.net/ml/linux-block/CAHC9VhQMABYKRqZmJQtXai0gtiueU42ENvSUH929=pF6tP9xOg@mail.gmail.com/>) that there does not appear to be a better solution at hand and merging Chamberlain's patch looked like the only path forward: ""Without any cooperation from the io_uring developers, that is likely what we will have to do"". The current plan appears to be to get Chamberlain's patch into the mainline during the next merge window, with backports to the stable kernels to be done thereafter. 

#### Grumpiness

This particular problem appears to be solved, albeit in a way that is less than satisfying to the security community. A better solution may materialize in the future, though providing a way to control access to device-specific functionality in a general way is a hard problem. But a harder problem may be addressing the residual grumpiness in the security community and preventing such problems from recurring in the future. As Moore [put it](<https://lwn.net/ml/linux-block/CAHC9VhRCW4PFwmwyAYxYmLUDuY-agHm1CejBZJUpHTVbZE8L1Q@mail.gmail.com/>): 

> I feel that expressing frustration about the LSMs being routinely left out of the discussion when new functionality is added to the kernel is a reasonable response; especially when one considers the history of this particular situation. 

For his part, Axboe [acknowledged](<https://lwn.net/ml/linux-block/711b10ab-4ac7-e82f-e125-658460acda89@kernel.dk/>) that the security concerns should not have been allowed to fall through the cracks, but he didn't necessarily offer a lot of hope for changes in the future: 

> I guess it's just somewhat lack of interest, since most of us don't have to deal with anything that uses LSM. And then it mostly just gets in the way and adds overhead, both from a runtime and maintainability point of view, which further reduces the motivation. 

Even when the motivation is there, mistakes can happen. Kernel development is a complex business. A lot of effort has gone into making the kernel sufficiently modular that developers need not worry about what is happening in the rest of the system, but there are limits to how far that process can go. 

For example, developers must be aware of locking and the locking requirements of subsystems they call into or things may go badly wrong. Memory must be handled according to the constraints placed on the memory-management subsystem, and developers creating complex caches may have to implement shrinkers to release memory on demand. CPU hotplug affects many subsystems and must be taken into account. The same is true of power-management events. Changes to the user-space API can create unhappiness years later. Inattention to latency constraints may create trouble in realtime applications. A failure to properly document a subsystem will make life harder for developers and users — but they are all used to that by now. 

And, of course, a failure to provide proper security hooks will hobble the ability of administrators to control process behavior by way of LSM policies. 

The fact that developers do not always succeed in keeping all of these constraints in mind — and consequently make mistakes — is unsurprising. Catching such omissions is one of the reasons for the existence of the kernel's sometimes tiresome review process. But nothing ensures that a given change will be properly reviewed by, for example, a developer who understands the needs of Linux security modules, and there is little that forces the suggestions from any such review to be heeded. 

So important things will occasionally fall through the cracks, and it is not clear that much can be done to improve the situation. It would be wonderful if more companies would pay developers to spend more time reviewing patches to provide, as an example, an overall security-oriented eye on code heading into the mainline, but that does not appear to be the world that we are living in. Attempts to impose requirements with a more bureaucratic process would mostly create friction and lead to the distribution of more out-of-tree (and severely unreviewed) code. 

The best path toward improvement may be, as Axboe [put it](<https://lwn.net/ml/linux-block/2c6541c2-d55b-4fbc-ec03-3b84722b7264@kernel.dk/>), ""one subsystem being aware of another one's needs"". Working toward that goal — and the ability to fix mistakes in the stable kernels when they do happen — seems to work reasonably well most of the time.
