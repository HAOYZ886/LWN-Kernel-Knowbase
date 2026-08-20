---
title: "The search for the correct amount of split-lock misery"
url: https://lwn.net/Articles/911219/
date: "October 19, 2022"
category: "Architectures-x86"
author: "By Jonathan Corbet October 19, 2022"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
October 19, 2022

Unlike many other architectures, x86 systems support atomic operations that affect more than one cache line. This support comes at a cost, though, in terms of overall system performance and, even, security. Over the last few years, kernel developers have worked to discourage the use of this sort of "split-lock" operation. Now, though, one group of users is feeling a little too discouraged, leading to a discussion of how much misery can appropriately be inflicted upon users who use problematic but architecturally legal operations. 

The problem with atomic operations that cross cache-line boundaries is that the system bus must take special measures to ensure that both cache lines are simultaneously protected from concurrent access. In practice, that means locking the bus for the duration of the operation, which can stall every other processor in the system. A malicious program executing a tight loop with a split-lock operation can destroy the performance of the system as a whole. For this reason, split-lock operations have long been frowned upon. 

Unfortunately, software that is malicious (or just poorly written) turns out to be remarkably indifferent to even the most severe of frowns. So, [starting in 2019](<https://lwn.net/Articles/790464/>), kernel developers sought more persuasive ways to get their point across. The initial work was done by Fenghua Yu but, in the end, [this patch](<https://git.kernel.org/linus/6650cdd9a8cc>) by Peter Zijlstra was merged in January 2020 for the 5.7 kernel release. It gives the kernel the ability to respond to traps caused by split-lock operations and provides three modes for that response, selectable by the `split_lock_detect=` command-line parameter: 

  * `off` causes the kernel to behave as it did before; split-lock operations are not detected and nothing is done when they occur. 
  * `warn` (the default) causes a (rate-limited) warning to go to the system log when a split-lock operation is detected. 
  * `fatal` causes the kernel to immediately kill (with `SIGBUS`) any process attempting split-lock operations. 

The hope was that the `warn` mode would be sufficient to alert users to the problem, and lead to software being fixed, while not actually interfering with anybody's use of their systems. By the time the 5.19 development cycle came around earlier this year, though, it seemed that little progress toward the removal of split lock operations had been made, and the denial-of-service problem was as present as before. So it was decided to take a stronger stance against split locks. 

One option, of course, would be to just switch to the `fatal` mode by default, but that would be a rather draconian solution. Instead, Tony Luck [wrote a patch](<https://git.kernel.org/linus/b041b525dab9>) with the descriptive title of "make life miserable for split lockers". It modified the `warn` mode to punish processes doing split locks without actually killing them. Instead, detection of a split lock would lead to a 10ms delay, then serialization via a semaphore. When this mode is selected, a malicious program performing split locks succeeds in slowing itself down, but no longer has much effect on the system as a whole. This change was applied during the 5.19 merge window. 

In mid-September, a GitHub user named "pibberflibbits" [posted a bug report](<https://github.com/doitsujin/dxvk/issues/2938>) saying that the performance of the [God of War game](<https://en.wikipedia.org/wiki/God_of_War_\(franchise\)>) on Linux had become ""insanely low"". It took a little while, but the participants in the resulting discussion eventually figured out that the problem was the split-lock penalty. Evidently one cannot be a proper god of war using just ordinary locks, so the game does a lot of split locking. Luck's patch had achieved its intended purpose; God of War players are now suitably miserable. 

Guilherme G. Piccoli, though, was not celebrating this victory over the Gods; instead, he posted [a patch](<https://lwn.net/ml/linux-kernel/20220928142109.150263-1-gpiccoli@igalia.com/>) arguing that ""it seems unacceptable to regress old/proprietary userspace programs through a default configuration that previously worked"". This patch restored the old behavior of the `warn` mode and added a new `seq` mode that would slow down split-lock users like `warn` mode does now. The `warn` mode would remain the default, lifting the misery from the game-playing world. 

Opinions on this change were mixed. Luck [pointed out](<https://lwn.net/ml/linux-kernel/SJ1PR11MB6083113884DD0B3031FE372CFC549@SJ1PR11MB6083.namprd11.prod.outlook.com/>) that gamers can simply disable split-lock detection by rebooting with `split_lock_detect=off` on the kernel command line. If the `seq` mode were to be added, he said, it should be the default. He also suggested filing a bug with the publishers of God of War to get its misbehavior fixed. 

Others disagreed, though. Joshua Ashton [argued](<https://lwn.net/ml/linux-kernel/f568c82c-738b-c5ec-5059-36659b3f5b44@froggi.es/>) that the problem is more widespread: ""It's not just about God of War specifically. There are many old titles that will never, ever, get updated to fix this problem. These titles worked perfectly fine and were performant before."" Others pointed out that many gamers are unlikely to be comfortable with adjusting kernel command-line parameters. Dave Hansen [observed](<https://lwn.net/ml/linux-kernel/24f31510-5b33-ada5-9f0e-117420403e8c@intel.com/>) that the misery-inflicting mode had worked as intended and had brought the problem to light. Even so, he continued: 

> My gut says we should keep the warnings and kill the misery. The folks who are going to be able to fix the issues are probably also the ones looking at dmesg and don't need the extra hint from the misery. The folks running Windows games don't look at dmesg and just want to play their game without misery. 

Luck, though, [argued](<https://lwn.net/ml/linux-kernel/SJ1PR11MB60839A08E01742321F8446BCFC579@SJ1PR11MB6083.namprd11.prod.outlook.com/>) that split locking creates its own misery for processes other than the one responsible, and that the current mode ""serves a very useful purpose on multi-user systems"". He suggested that perhaps some sort of heuristic could be developed to confine the misery to multi-user systems. 

The [definitive answer](<https://lwn.net/ml/linux-kernel/87pmf4bter.ffs@tglx/>), though, came from Thomas Gleixner, who pointed out that slowing down split lockers by default is the only choice that distributors could make; anything else creates an easily exploitable denial-of-service vulnerability. So the slowdown needs to remain: ""Attack vector prevention has precedence over broken applications"". He did suggest, though, that a sysctl knob could be added to control split-lock detection; that would allow users of broken applications to get their performance back without the need to figure out how to change command-line parameters or reboot their systems. 

That is the approach that Piccoli has taken for [his second attempt](<https://lwn.net/ml/linux-kernel/20221014180506.211592-1-gpiccoli@igalia.com/>) at addressing the problem. The new `kernel.split_lock_mitigate` knob, if set to zero, will disable the penalization of processes using split locking (while retaining the warning sent to the system log). The default is to retain the slowdown. This patch seems to have pleased everybody involved and looks likely to find its way into the 6.2 kernel. Affected gamers will have to set the new knob appropriately, but knowing which sysctls to tweak could be said to be part of being a true God of War.
