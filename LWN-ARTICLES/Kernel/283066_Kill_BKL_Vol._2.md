---
title: Kill BKL Vol. 2
url: https://lwn.net/Articles/283066/
date: "May 21, 2008"
category: Big kernel lock; lock kernel
author: "By Jonathan Corbet May 21, 2008"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
May 21, 2008

[Last week's big kernel lock article](<http://lwn.net/Articles/281938/>) discussed a BKL-related performance regression and concluded that we would likely see a new interest in its elimination. In the intervening week, that interest has indeed come to the fore. There are now a couple of different efforts afoot to get rid of this long-lasting lock. 

One might well wonder why the BKL is so persistent. Over the last (approximately) fifteen years, thousands of locks have been added to the kernel, pushing the BKL into increasingly obscure corners. But there are a _lot_ of those corners, including a great many explicit `lock_kernel()` calls, the `open()` method for every char device, most `ioctl()` implementations, all `fasync()` implementations, and more. The BKL can be found throughout the kernel, and doesn't appear ready to go without a fight. 

Part of the problem is simply that locking is hard. So going in and changing the locking of some crufty, old driver is not at the top of the list for a lot of developers, who would generally rather be creating crufty new drivers. Beyond that, though, the BKL is special. It was originally created to be more than just a locking primitive; its purpose is to allow BKL-covered code to pretend that it is still running on an old, uniprocessor system. So its semantics are very different from any other lock in the Linux kernel. 

For example, the BKL nests, so programmers can add `lock_kernel()` calls anywhere without worrying about whether the BKL might already have been acquired elsewhere. As with a mutex, code holding the BKL can sleep; however, the scheduler will magically release the BKL until the holding thread wakes up again. So there can be various threads in kernel space, all of which think they hold the BKL, but only one of them will actually be running at any given time. The end result is that it is hard to get a handle on what is happening with the BKL at any given time; code can depend on it without ever really being aware of its existence. 

As Ingo Molnar put it in his [kill the BKL tree announcement](<http://lwn.net/Articles/282319/>): 

Furthermore, the BKL is not covered by lockdep, so its dependencies are largely unknown and invisible, and it is all lost in the haze of the past ~15 years of code changes. All this has built up to a kind of Fear, Uncertainty and Doubt about the BKL: nobody really knows it, nobody really dares to touch it and code can break silently and subtly if BKL locking is wrong. 

That doesn't mean that people aren't willing to try; Ingo's tree - to which we will return shortly - is a major effort in that direction. But first, consider another initiative which, somewhat accidentally, turned up an example of just how subtle BKL-related issues can be. As was mentioned above, the kernel grabs the BKL whenever a process opens a char device; the BKL is held while the associated driver's `open()` function runs. To eliminate BKL, one must remove this particular use of it; one cannot just take it out, however, without breaking every driver which does not have proper locking internally. So, in fact, this `lock_kernel()` call cannot be removed until every driver's `open()` function has been audited and, if necessary, fixed. That's a big flag day. 

An alternative, which your editor rashly jumped into doing, is to push the acquisition of the BKL down one level. Every `open()` function is forced to be correct through the addition of explicit `lock_kernel()` and `unlock_kernel()` calls; once all of the in-tree drivers have been fixed in this way, the higher-level call in `chrdev_open()` can be removed. This work may seem like a step backward, in that it replaces a single `lock_kernel()` call with approximately 100 others. But it's actually a big step forward, in that each driver can now be audited and fixed independently. This work has now been done, the resulting tree is in linux-next, and, if all goes well, it should be ready for 2.6.27. 

While doing this work, though, your editor noticed quite a few drivers with open functions that were either completely empty (all they do is "`return 0`") or they do something relatively trivial. These functions, one would think, do not need to acquire the BKL; they touch no global resources and cannot possibly race with any other part of the kernel. In fact, as was suggested by others, the empty `open()` functions could just be removed altogether. 

It was Alan Cox who [pointed out](<https://lwn.net/Articles/283067/>) that life is not quite so simple. Under the current regime, an open function which looks like this: 

```
static int empty_open(struct inode *inode, struct file *filp)
        {
        	return 0;
        }
```

is really better modeled as this: 

```
static int empty_open(struct inode *inode, struct file *filp)
        {
            lock_kernel();
    	unlock_kernel();
        	return 0;
        }
```

These two may seem the same, but there is a crucial difference: in the second form, `empty_open()` will not return until it can acquire the BKL. In other words, after `empty_open()` runs, one knows that the BKL became available at least once. And this matters: a classic device driver error is to (1) register a device with the kernel, then (2) initialize all of the internal data structures needed to manage that device. Should some other process attempt to open and use the device between those two steps, unpleasant things can happen. The `lock_kernel()` call in the `open()` function, despite protecting no critical section directly, serializes the opening of the device with the driver's initialization, and thus prevents mayhem. So, [says Alan](<https://lwn.net/Articles/283068/>), 

I think it would be best to make them lock/unlock kernel in the first pass and then work through them. The BKL can be subtle and evil, but as I brought it into the world I guess I must banish it ;) 

Alan will not be alone in that effort, though, and Ingo Molnar's "kill the BKL" tree is likely to help this work considerably. Ingo's approach is to get rid of most of the features which make the BKL special. So, with his patches, the BKL becomes just another mutex which, crucially, can be tracked with the [lock validator](<http://lwn.net/Articles/185666/>). It is no longer released when a thread calls `schedule()`, a change which forced the addition of a few explicit "release, schedule, and reacquire" changes in code which would otherwise deadlock. There's a number of warnings added to point out calls made holding the BKL which should not be. And so on. 

This patch set, in essence, removes the BKL entirely, replacing it with just another big lock which happens to do nesting. And the nesting might go too at some point. So the BKL becomes more visible and easier to understand. And, presumably, easier to eliminate. 

Linus [likes this approach](<https://lwn.net/Articles/283069/>), though he would like to see it reworked to the point that it can be merged into the mainline relatively soon. Doing that would require putting most of the changes behind a configuration option decorated with a sufficient number of scary warnings; then people who wanted to test this code could turn it on and see what explodes. The number of explosions would probably be relatively small - but probably not zero. 

This set of changes, along with the other work being done, suggests that significant progress toward the elimination of the BKL can be expected over the next few kernel development cycles. Once it's gone, we'll have a kernel without legacy locking issues, and without the unpleasant performance issues that the BKL can bring. That will still take a while, though; there is simply no substitute for actually looking at all the BKL-covered code and ensuring that it will run safely in the absence of that protection. It's a painstaking job requiring moderate skills which can only be rushed so much.
