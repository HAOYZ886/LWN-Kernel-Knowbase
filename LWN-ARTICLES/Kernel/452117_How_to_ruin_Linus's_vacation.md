---
title: "How to ruin Linus's vacation"
url: https://lwn.net/Articles/452117/
date: "July 19, 2011"
category: Dentry cache
author: "By Jonathan Corbet July 19, 2011"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
July 19, 2011

It's all Hugh's fault. 

Linus was all set to release the final 3.0 kernel when Hugh Dickins showed up on the list with [a little problem](<https://lwn.net/Articles/452118/>): occasionally a full copy of the kernel source tree fails because one of the files found therein vanishes temporarily. What followed was a determined bug-chasing exercise which demonstrates how subtle and tricky some of our core code has become. The problem has been found and squashed, but there may be more. 

A bit of background might help in understanding what was happening here. The 2.6.38 release included the [dcache scalability patches](<https://lwn.net/Articles/419811/>); this code uses a number of tricks to avoid taking locks during the process of looking up file names. For the right kind of workload, the "RCU walk" method yields impressive performance improvements. But that only works if all of the relevant directory entry ("dentry") structures are in the kernel's dentry cache and the lookup process does not race with other CPUs which may be making changes on the same path. Whenever such a situation is encountered, the lookup process will fall back to the older, slower algorithm which requires locking each dentry. 

The dentry cache (dcache) is a highly dynamic data structure, with dentries coming and going at all times. So one CPU might be removing a dentry at the same time that another is using it to look up a name. Chaos is avoided through the use of read-copy-update (RCU) to manage the removal of dentries; a dentry may be removed from the cache, but, if the thread using that dentry for lookup got a reference to it before its removal, the structure itself will continue to exist for as long as that thread needs it. The same should be true of the inode structure associated with that dentry. 

Hugh tracked the problem down to a bit of code in `walk_component()`: 

```
err = do_lookup(nd, name, path, &inode);
    	/* ... */
    	if (!inode) {
    		path_to_nameidata(path, nd);
    		terminate_walk(nd);
    		return -ENOENT;
    	}
```

If `do_lookup()` returns a null inode pointer, `walk_component()` assumes that a "negative dentry" has been encountered. Negative dentries are kept in the dentry cache to record the fact that a specific name does _not_ exist; they are an important performance-enhancing feature in the Linux virtual filesystem layer. To see an example, run any simple program under `strace` and watch how many system calls return with ENOENT; lookups on nonexistent files happen frequently. What Hugh determined was that this inode pointer was coming back null even though the file exists, leading the code to believe that a negative dentry had been found and causing the "briefly vanishing file" problem. 

Hugh must have looked at this code for some time before concluding that the kernel must be removing the dentry from the dcache at just the wrong time during the lookup process. As described above, the dentry itself continues to exist after its removal from the cache, but that does not mean that it is unchanged: the removal process sets its `d_inode` pointer to `NULL`. (It's worth noting that this behavior goes against normal RCU practice, which calls for the structure to be preserved unmodified until the last reference is known to be gone). Hugh concluded that this null pointer was being picked up later by the lookup process, causing `walk_component()` to conclude that the file does not exist when all that had happened was the removal of a dentry from the cache. His problem report included a patch causing the lookup code to check much more carefully when the inode pointer comes up null. 

Linus [acknowledged the problem](<https://lwn.net/Articles/452128/>) but didn't like the fix which, he thought, was too specific to one particular situation. He proposed an alternative: just don't set `d_inode` to `NULL`; that would keep the inode pointer from picking up that value later. Al Viro posted [an alternative fix](<https://lwn.net/Articles/452131/>) which changed dcache behavior in less subtle ways, and [worried](<https://lwn.net/Articles/452134/>) about the possibility of introducing other weird bugs: 

I'm not entirely convinced that it's a valid optimization in the first place (probably is, but I'm seriously scared by the complexity we already have there), and I'm really not fond of the idea of dealing with whatever subtle crap we might discover with Linus' patch. Again, dcache is not in a healthy shape right now; at this point dumb and straightforward is, IMO, better than subtle and risking to step on toes of very odd code out there... 

Once we are done with code audit, sure, I'm fine with ->d_inode being kept until dentry is actually freed. Any code that relies on that thing being cleared is asking for trouble and should be rewritten anyway. The only thing is, it needs to be found before we rewrite it... 

Linus didn't like Al's fix either; it threatened to force slow lookups when negative dentries are involved. The discussion of the patches went on at some length; in the process of trying to find the safest way to fix this subtle bug the participants slowly came to the realization that they did not actually know what was happening. After looking at things closely, Linus [threw up his hands](<https://lwn.net/Articles/452136/>) and admitted he didn't understand it: 

So how could Hugh's NULL inode ever happen in the first place? Even with the current sources? It all looks solid to me now that I look at all the details. 

As it happens, Linus's exposition was enough to point Hugh at the real problem. Just as the process of transiting through a specific dentry is almost complete, `do_lookup()` makes a call to `__follow_mount_rcu()`, whose job is to redirect the lookup process if it is passing through a mount point. The inode pointer is passed to `__follow_mount_rcu()` separately; Hugh noticed that this function was doing the following: 

```
*inode = path->dentry->d_inode;
```

In other words, the inode pointer is being re-fetched from the dentry structure; this assignment happens regardless of whether the dentry represents a mount point. That is the true source of the problem: if the dentry has been removed from the dcache after the lookup process gained a reference, `d_inode` will be `NULL`. So `__follow_mount_rcu()` will zero a pointer which had pointed to a valid inode, causing later code to think that the file does not exist at all. 

Linus posted [a fix for the real problem](<https://lwn.net/Articles/452138/>) along with his [now-famous Google+ posting](<https://plus.google.com/102150693225130002912/posts/2BXkWyrY4jH>) saying that he was delaying the 3.0 release for a day just in case: 

We have a patch, we understand the problem, and it looks ObviouslyCorrect(tm), but I don't think I want to release 3.0 just a couple of hours after applying it. 

Linus delayed the release despite [the inconvenient fact](<https://lwn.net/Articles/452145/>) that it will push the 3.1 merge window into his planned vacation. That was a well-placed bit of caution on his part: the ObviouslyCorrect(tm) patch had [YetAnotherSubtleBug(tm) in it](<https://lwn.net/Articles/452141/>). A fixed version of the patch exists, and this particular bug should, at this point, be history. 

There is a sobering conclusion to be drawn from this episode, though. The behavior of the dentry cache is, at this point, so subtle that even the combined brainpower of developers like Linus, Al, and Hugh has a hard time figuring out what is going on. These same developers are visibly nervous about making changes in that part of the kernel. Our once approachable and hackable kernel has, over time, become more complex and difficult to understand. Much of that is unavoidable; the environment the kernel runs in has, itself, become much more complex over the last 20 years. But if we reach a point where almost nobody can understand, review, or fix some of our core code, we may be headed for long-term trouble. 

Meanwhile, we should be able to enjoy a 3.0 release (and a 2.6.39 update) without mysteriously vanishing files. One potential short-term problem remains, though: given that the next merge window will push into Linus's vacation, there is a distinct chance that he might be more than usually grumpy with maintainers who get their pull requests in late. Wise subsystem maintainers may want to be ready to go when the merge window opens.
