---
title: Removing four bytes from the kernel ABI
url: https://lwn.net/Articles/498128/
date: "May 23, 2012"
category: "Development model-User-space ABI"
author: "By Jake Edge May 23, 2012"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jake Edge**  
May 23, 2012

Four bytes may not seem like a lot of space--typically it isn't--but when that space is wasted millions of times, it starts to add up. In addition, if the extra space has become part of the kernel ABI (intentionally or not), it will be difficult to remove it. That particular problem came up again in a recent linux-kernel discussion regarding the trace event header. 

Just over a year ago, we [looked](<https://lwn.net/Articles/442113/>) at the unused `lock_depth` field in event headers. Frederic Weisbecker had added the field temporarily to assist in removal of the big kernel lock (BKL), and once the BKL was gone Steven Rostedt removed those, now useless, four bytes from the header. Unfortunately, in the interim, PowerTOP had started accessing events in the perf ring buffer, so removing `lock_depth` broke PowerTOP. That field wasn't actually used by PowerTOP, but the tool expected the header to have a particular size, which changed after Rostedt removed the wasted space. 

That led to a reversion of the removal, which means that every event recorded by ftrace or perf has added overhead. The event format is fully self-describing, however, so there is no need for utilities like PowerTOP to grub around in the binary data making assumptions about what the format is. It was, however, easier to read the data directly rather than parse the format description, which is why PowerTOP did so. Rostedt has created a library to parse trace events using the format data that the kernel provides to avoid that situation in the future. That library was picked up by the [recently released PowerTOP 2.0](<https://01.org/powertop/blogs/ceferron/2012/powertop-v2.0-release>), so Rostedt [posted an RFC](<https://lwn.net/Articles/498194/>) asking when the `lock_depth` field--renamed to `padding` as part of the revert--could be removed. 

Linus Torvalds was [not particularly concerned](<https://lwn.net/Articles/498213/>) about the wasted space, but did want to understand which distributions were picking up the new PowerTOP. It turns out that the version in Fedora 14 (which Torvalds said he still uses sometimes) is old enough that it doesn't use perf events at all, so it is unaffected. More recent Fedoras (16, 17) are using PowerTOP 1.98 which won't work with kernels built without the padding. 

The assumption in the thread is that distributions will be picking up PowerTOP 2.0 for releases coming later in the year, but that still leaves users who build their own kernels on existing distributions in a bit of a bind if the padding is removed. Existing distributions also have various lifespans, and some will not be picking up the latest PowerTOP at all. Rostedt [asked](<https://lwn.net/Articles/498228/>) how long the kernel needed to support older distributions. PowerTOP, it seems, is in a different category from other applications because it is a developer-oriented tool. So Torvalds was [willing](<https://lwn.net/Articles/498229/>) to see the kernel change even if some distributions get left behind: 

But breaking something like a F14-15 timeframe distro or something staid like a SLES (or "Debian Stale" or whatever they call that thing that only takes crazy-old binaries)? It's fine. We don't want to *rush* into it, but no, if those distros are basically not updating, we can't care about them forever for something like powertop. 

Things that break *normal* applications are different. There the rule really must be "never". 

Arjan van de Ven [concurred](<https://lwn.net/Articles/498232/>), pointing to 3.6 as a potential time frame to remove the padding, noting that those who haven't updated their distribution to get the newer PowerTOP are unlikely to be updating their kernel either. Rostedt said he will queue the patch up for 3.6 or 3.7. 

While the four bytes seems unimportant to both Torvalds and [Ingo Molnar](<https://lwn.net/Articles/498235/>), Rostedt [pointed out](<https://lwn.net/Articles/498239/>) that it is a frequent problem for tracing users. Beyond that, though, he [disagrees](<https://lwn.net/Articles/498236/>) with Molnar's contention that the wasted space is merely a ""cosmetic detail"": 

4 bytes is not cosmetic for a 32 byte event. That's 1/8th overhead. If we could get rid of 4 bytes from struct page, would we do that? It's only just 4 bytes for [every] 4096 bytes. Just a 1/1024 overhead. Of course perf events are much bigger than 32 bytes. It's one of the biggest complaints I hear about perf, the size of its events. We should be trying hard to fix that. 

For memory-constrained situations, for example on embedded devices or for users trying to squeeze every process they can onto their systems, reducing the overhead of events can make a difference. By capturing more events in the same amount of memory, there is a better chance of finding the problem that tracing was enabled for. When the issue came up a year ago, David Sharp of Google [noted](<https://lwn.net/Articles/442341/>) that the size of events was a big problem for the search giant. Others undoubtedly face similar challenges. 

While the format of the perf ring buffer data may soon be a solved problem--though it's possible, if unlikely, that other tools are manually pulling data from the ring buffer--tracepoints as a whole are still an unresolved ABI issue. Right now, much of the work is in adding new tracepoints, but some day one or more of those may need to come out or be modified. If tools are dependent on specific tracepoints providing the exact same information in just the right place in the code, changing those will be a real problem. And it will be one that is difficult for a library to paper over.
