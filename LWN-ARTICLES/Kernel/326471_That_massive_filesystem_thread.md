---
title: That massive filesystem thread
url: https://lwn.net/Articles/326471/
date: "March 31, 2009"
category: Filesystems
author: "By Jonathan Corbet March 31, 2009"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
March 31, 2009

Long, highly-technical, and animated discussion threads are certainly not unheard of on the linux-kernel mailing list. Even by linux-kernel standards, though, [the thread that followed the 2.6.29 announcement](<http://news.gmane.org/group/gmane.linux.kernel/thread=811167/force_load=t>) was impressive. Over the course of hundreds of messages, kernel developers argued about several aspects of how filesystems and block I/O work on contemporary Linux systems. In the end (your editor will be optimistic and say that it has mostly ended), we had a lot of heat - and some useful, concrete results. 

One can only pity Jesper Krogh, who almost certainly didn't know what he was getting into when he posted [a report](<https://lwn.net/Articles/326490/>) of a process which had been hung up waiting for disk I/O for several minutes. All he was hoping for was a suggestion on how to avoid these kinds of delays - which are a manifestation of the famous ext3 `fsync()` problem - on his server. What he got, instead, was to be copied on the entire discussion. 

### Journaling priority

One of the problems is at least somewhat understood: a call to `fsync()` on an ext3 filesystem will force the filesystem journal (and related file data) to be committed to disk. That operation can create a lot of write activity which must be waited for. But contemporary I/O schedulers tend to favor read operations over writes. Most of the time, that is a rational choice: there is usually a process waiting for a read to complete, but writes can be done asynchronously. A journal commit is not asynchronous, though, and it can cause a lot of things to wait while it is in progress. So it would be better not to put journal I/O operations at the end of the queue. 

In fact, it would be better not to make journal operations contend with the rest of the system at all. To that end, Arjan van de Ven has long maintained [a simple patch](<http://lwn.net/Articles/301467/>) which gives the `kjournald` thread realtime I/O priority. [According to Alan Cox](<https://lwn.net/Articles/326492/>), this patch alone is sufficient to make a lot of the problems go away. The patch has never made it into the mainline, though, because [Andrew Morton has blocked it](<https://lwn.net/Articles/326494/>). This patch, he says, does not address the real problem, and it causes a lot of unrelated I/O traffic to benefit from elevated priority as well. Andrew says the real fix is harder: 

The bottom line is that someone needs to do some serious rooting through the very heart of JBD transaction logic and nobody has yet put their hand up. If we do that, and it turns out to be just too hard to fix then yes, perhaps that's the time to start looking at palliative bandaids. 

Bandaid or not, this approach has its adherents. The ext4 filesystem has a new mount option (`journal_ioprio`) which can be used to set the I/O priority for journaling operations; it defaults to something higher than normal (but not realtime). More recently, Ted Ts'o has posted [a series of ext3 patches](<http://lwn.net/Articles/324822/>) which sets the `WRITE_SYNC` flag on some journal writes. That flag marks the operations as synchronous, which will keep them from being blocked by a long series of read operations. According to Ted, this change helps quite a bit, at least when there is a lot of read activity going on. The ext3 changes have not yet been merged for 2.6.30 as of this writing (none of Ted's trees have), but chances are they will go in before 2.6.30-rc1. 

### data=ordered, fsync(), and fbarrier()

The real problem, though, according to Ted, is the ext3 `data=ordered` mode. That is the mode which makes ext3 relatively robust in the face of crashes, but, says Ted, it has done so at the cost of performance and the encouragement of poor user-space programming. He went so far as to [express his regrets](<https://lwn.net/Articles/326503/>) for this behavior: 

All I can do is apologize to all other filesystem developers profusely for ext3's data=ordered semantics; at this point, I very much regret that we made data=ordered the default for ext3. But the application writers vastly outnumber us, and realistically we're not going to be able to easily roll back eight years of application writers being trained that fsync() is not necessary, and actually is detrimental for ext3. 

The only problem here is that not everybody believes that ext3's behavior is a bad thing - at least, with regard to robustness. Much of this branch of the discussion covered the same issues raised by LWN in [Better than POSIX?](<http://lwn.net/Articles/323752/>) a couple of weeks before. A significant subset of developers do not want the additional robustness provided by ext3 `data=ordered` mode to go away. Matthew Garrett [expressed this position well](<https://lwn.net/Articles/326504/>): 

But you're still arguing that applications should start using fsync(). I'm arguing that not only is this pointless (most of this code will never be "fixed") but it's also regressive. In most cases applications don't want the guarantees that fsync() makes, and given that we're going to have people running on ext3 for years to come they also don't want the performance hit that fsync() brings. Filesystems should just do the right thing, rather than losing people's data and then claiming that it's fine because POSIX said they could. 

One option which came up a couple of times was to extend POSIX with a new system call (called something like `fbarrier()`) which would enforce ordering between filesystem operations. A call to `fbarrier()` could, for example, cause the data written to a new file to be forced out to disk before that file could be renamed on top of another file. The idea has some appeal, but [Linus dislikes it](<https://lwn.net/Articles/326505/>): 

Anybody who wants more complex and subtle filesystem interfaces is just crazy. Not only will they never get used, they'll definitely not be stable... 

So rather than come up with new barriers that nobody will use, filesystem people should aim to make "badly written" code "just work" unless people are really really unlucky. Because like it or not, that's what 99% of all code is. 

And that is almost certainly how things will have to work. In the end, a system which just works is the system that people will want to use. 

### relatime

Meanwhile, another branch of the conversation revisited [an old topic](<http://lwn.net/Articles/244829/>): atime updates. Unix-style filesystems traditionally track the time that each file was last accessed ("atime"), even though, in reality, there is very little use for this information. Tracking atime is a performance problem, in that it turns every read operation into a filesystem write as well. For this reason, Linux has long had a "noatime" mount option which would disable atime updates on the indicated filesystem. 

As it happens, though, there can be problems with disabling atime entirely. One of them is that the `mutt` mail client uses atime to determine whether there is new mail in a mailbox. If the time of last access is prior to the time of last modification, `mutt` knows that mail has been delivered into that mailbox since the owner last looked at it. Disabling atime breaks this mechanism. In response to this problem, the kernel added a "relatime" option which causes atime to be updated only if the previous value is earlier than the modification time. The relatime option makes `mutt` work, but it, too, turns out to be insufficient: some distributions have temporary-directory cleaning programs which delete anything which hasn't been used for a sufficiently long period. With relatime, files can appear to be totally unused, even if they are read frequently. 

If relatime could be made to work, the benefits could be significant; the elimination of atime updates can get rid of a lot of writes to the disk. That, in turn, will reduce latencies for more useful traffic and will also help to avoid disk spin-ups on laptops. To that end, Matthew Garrett posted a patch to modify the relatime semantics slightly: it allows atime to be updated if the previous value is more than one day in the past. This approach eliminates almost all atime updates while still keeping the value close to current. 

This patch was proposed for merging, and more: it was suggested that relatime should be made the default mode for filesystems mounted under Linux. Anybody wanting the traditional atime behavior would have to mount their filesystems with the new "strictatime" mount option. This idea ran into some immediate opposition, for a couple of reasons. Andrew Morton [didn't like the hardwired 24-hour value](<https://lwn.net/Articles/326508/>), saying, instead, that the update period should be given as a mount option. This option would be easy enough to implement, but few people think there is any reason to do so; it's hard to imagine a use case which requires any degree of control over the granularity of atime updates. 

Alan Cox, instead, [objected](<https://lwn.net/Articles/326509/>) to the patch as an ABI change and a standards violation. He tried to "NAK" the patch, saying that, instead, this sort of change should be done by distributors. Linus, however, [said he doesn't care](<https://lwn.net/Articles/326511/>); the relatime change and strictatime option were the very first things he merged when he opened the 2.6.30 merge window. His position is that the distributors have had more than a year to make this change, and they haven't done so. So the best thing to do, he says, is to change the default in the kernel and let people use strictatime if they really need that behavior. 

For the curious, Valerie Aurora has [written a detailed article](<http://valhenson.livejournal.com/36519.html>) about this change. She doesn't think that the patch will survive in its current form; your editor, though, does not see a whole lot of pressure for change at this point. 

### I/O barriers

Suppose you are a diligent application developer who codes proper `fsync()` calls where they are necessary. You might think that you are then protected against data loss in the face of a crash. But there is still a potential problem: the disk drive may lie to the operating system about having written the data to persistent media. Contemporary hardware performs aggressive caching of operations to improve performance; this caching will make a system run faster, but at the cost of adding another way for data to get lost. 

There is, of course, a way to tell a drive to actually write data to persistent media. The block layer has long had support for barrier operations, which cause data to be flushed to disk before more operations can be initiated. But the ext3 filesystem does not use barriers by default because there is an associated performance penalty. With ext4, instead, barriers are on by default. 

Jeff Garzik [pointed out](<https://lwn.net/Articles/326513/>) one associated problem: a call to `fsync()` does not necessarily cause the drive to flush data to the physical media. He suggested that `fsync()` should create a barrier, even if the filesystem as a whole is not using barriers. In that way, he says, `fsync()` might actually live up to the promise that it is making to application developers. 

The idea was not controversial, even though people are, as a whole, less concerned with caches inside disk drives. Those caches tend to be short-lived, and they are quite likely to be written even if the operating system crashes or some other component of the system fails. So the chances of data loss at that level are much smaller than they are with data in an operating system cache. Still, it's possible to provide a higher-level guarantee, so Fernando Luis Vazquez Cao posted [a series of patches](<https://lwn.net/Articles/326514/>) to add barriers to `fsync()` calls. And that is when the trouble started. 

The fundamental disagreement here is over what should happen when an attempt to send a flush operation to the device fails. Fernando's patch returned an `ENOTSUPP` error to the caller, but Linus [asked for it to be removed](<https://lwn.net/Articles/326515/>). His position is that there is nothing that the caller can do about a failed barrier operation anyway, so there is no real reason to propagate that error upward. At most, the system should set a flag noting that the device doesn't support barriers. But, says Linus, filesystems should cope with what the storage device provides. 

Ric Wheeler, instead, [argues](<https://lwn.net/Articles/326517/>) that filesystems should know if barrier operations are not working and be able to respond accordingly. Says Ric: 

One thing the caller could do is to disable the write cache on the device. A second would be to stop using the transactions - skip the journal, just go back to ext2 mode or BSD like soft updates. 

Basically, it lets the file system know that its data integrity building blocks are not really there and allows it (if it cares) to try and minimize the chance of data loss. 

Alan Cox also [jumped into this discussion](<https://lwn.net/Articles/326518/>) in favor of stronger barriers: 

Throw and pray the block layer can fake it simply isn't a valid model for serious enterprise computing, and if people understood the worst cases, for a lot of non enterprise computing. 

Linus appears to be unswayed by these arguments, though. In his view, filesystems should do the best they can and accept what the underlying device is able to do. As of this writing, no patches adding barriers to `fsync()` have been merged into the mainline. 

Related to this is the concept of laptop mode. It has been [suggested](<https://lwn.net/Articles/326522/>) that, when a system is in laptop mode, an `fsync()` call should not actually flush data to disk; flushing the data would cause the drive to spin up, defeating the intent of laptop mode. The response to I/O barrier requests would presumably be similar. Some developers oppose this idea, though, seeing it as a weakening of the promises provided by the API. This looks like a topic which could go a long time without any real resolution. 

### Performance tuning

Finally, there was some talk about trying to make the virtual memory subsystem perform better in general. Part of the problem here has been recognized for some time: memory sizes have grown faster than disk speeds. So it takes a lot longer to write out a full load of dirty pages than it did in the past. That simple dynamic is part of the reason why writeout operations can stall for long periods; it just takes that long to funnel gigabytes of data onto a disk drive. It is generally expected that solid-state drives will eventually make this problem go away, but it is also expected that it will be quite some time, yet, before those drives are universal. 

In the mean time, one can try to improve performance by not allowing the system to accumulate as much data in need of writing. So, rather than letting dirty pages stay in cache for (say) 30 seconds, those pages should be flushed more frequently. Or the system could adjust the percentage of RAM which is allowed to be dirty, perhaps in response to observations about the actual bandwidth of the backing store devices. The kernel already has a "percentage dirty" limit, but some developers are now suggesting that the limit should be a fixed number of bytes instead. In particular, that limit should be set to the number of bytes which can be flushed to the backing store device in (say) one second. 

Nobody objects to the idea of a better-tuned virtual memory subsystem. But there is some real disagreement over how that tuning should be done. Some developers argue for exposing the tuning knobs to user space and letting the distributors work it out. Andrew is [a strong proponent](<https://lwn.net/Articles/326529/>) of this approach: 

I've seen you repeatedly fiddle the in-kernel defaults based on in-field experience. That could just as easily have been done in initscripts by distros, and much more effectively because it doesn't need a new kernel. That's data. 

The fact that this hasn't even been _attempted_ (afaik) is deplorable. Why does everyone just sit around waiting for the kernel to put a new value into two magic numbers which userspace scripts could have set? 

The objections to that approach follow these lines: the distributors cannot get these numbers right; in fact, they are not really even inclined to try to get them right. The proper tuning values tend to change from one kernel to the next, so it makes sense to keep them with the kernel itself. And the kernel should be able to get these things right if it is doing its job at all. Needless to say, Linus [argues](<https://lwn.net/Articles/326530/>) for this approach, saying: 

We should aim to get it right. The "user space can tweak any numbers they want" is ALWAYS THE WRONG ANSWER. It's a cop-out, but more importantly, it's a cop-out that doesn't even work, and that just results in everybody having different setups. Then nobody is happy. 

Linus has suggested (but not implemented) [one set of heuristics](<https://lwn.net/Articles/326531/>) which could help the system to tune itself. Neil Brown also [has a suggested approach](<https://lwn.net/Articles/326532/>), based on measuring the actual performance of the system's storage devices. Fixing things at this level is likely to take some time; virtual memory changes always do. But some smart people are starting to think about the problem, and that's an important first step. 

That, too, could be said for the discussion as a whole. There are clearly a lot of issues surrounding filesystems and I/O which have come to the surface and need to be discussed. The Linux kernel community as a whole needs to think through the sort of guarantees (for both robustness and performance) it will offer to its users and how those guarantees will be fulfilled. As it happens, the [2009 Linux Storage & Filesystems Workshop](<http://events.linuxfoundation.org/events/lsf-workshop>) begins on April 6\. Many of these topics are likely to be discussed there. Your editor has managed to talk his way into that room; stay tuned.
