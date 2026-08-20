---
title: "A VFS deadlock post-mortem"
url: https://lwn.net/Articles/545119/
date: "April 3, 2013"
category: "Dentry cache; proc-Inode locking"
author: "By Michael Kerrisk April 3, 2013"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Michael Kerrisk**  
April 3, 2013

Dave Jones continues to exercise his [Trinity fuzz tester](<https://lwn.net/Articles/536173/>) and uncover interesting bugs in kernel code. One recent find was a long-standing bug in the implementation of network namespaces. 

The discussion of the bug began when Dave [posted](<https://lwn.net/Articles/545421/>) a note to the linux-kernel mailing list with stack traces that showed a kernel deadlock in the VFS code. Dave's report prompted Al Viro to [wonder](<https://lwn.net/Articles/545456/>) how a Trinity instance was managing to sit blocked on _two_ locks (a situation that should never be able to happen), as shown in the [lockdep](<https://lwn.net/Articles/185666/>) output posted by Dave (the output has some key pieces highlighted): 

```
Showing all locks held in the system:
        4 locks on stack by trinity-child2/7669:
         #0: **blocked** :  (sb_writers#4){.+.+.+}, 
             instance: ffff8801292d17d8, at: [<ffffffff811df134>] mnt_want_write+0x24/0x50
         #1: held:     (&type->s_vfs_rename_key){+.+.+.}, 
             instance: ffff8801292d1928, at: [<ffffffff811c6f5e>] lock_rename+0x3e/0x120
         #2: held:     (&type->**i_mutex_dir_key** #2/1){+.+.+.}, 
             instance: **ffff880110b3a858** , at: [<ffffffff811c701e>] lock_rename+0xfe/0x120
         #3: **blocked** :  (&type->**i_mutex_dir_key** #2/2){+.+.+.}, 
             instance: **ffff880110b3a858** , at: [<ffffffff811c7034>] lock_rename+0x114/0x120
```

Al also [noted](<https://lwn.net/Articles/545457/>) that the output suggested that a directory inode in the inode cache was mapped by two different dentries, since lockdep showed two `i_mutex_dir_key` locks on the same address. A dentry (directory entry) is a data structure representing a filename in the kernel directory entry cache (dcache); a brief overview of dentries and the dcache can be found in [this article](<https://lwn.net/Articles/419811/>). As will become clear shortly, it should normally never happen that a directory inode is mapped twice in the dcache. 

Some suggestions ensued regarding suitable debugging statements to add to the kernel's `lock_rename()` function to further investigate the problem. In particular, when two locks were held on the same inode address, Linus [wanted to see](<https://lwn.net/Articles/545458/>) the filenames corresponding to the inode and Al [was interested to know](<https://lwn.net/Articles/545461/>) the name of the filesystem holding the two inodes. 

Further runs of Trinity with those debugging statements in place [revealed](<https://lwn.net/Articles/545462/>) that the locks in question were occurring for various entries under the `/proc` tree. At that point Linus [refined](<https://lwn.net/Articles/545464/>) the observation to note that the entries in question were for directories under `/proc/net`, but, like Al, he was puzzled as to how that could occur. 

Here, a little background is probably in order. Once upon a time, `/proc/net` was single directory. But, with the invention of network namespaces, it is now a symbolic link to the `/proc/self/net` directory; in other words, each process now has its own network-namespace-specific view of networking information under `/proc`. 

With the output from the kernel debugging statements, the pieces started falling rapidly into place. Dave [realized](<https://lwn.net/Articles/545465/>) that he had started seeing the Trinity failure reports after he had enabled kernel namespaces support following a recent bug fix by Eric Biederman. Al began [looking more closely](<https://lwn.net/Articles/545466/>) at some of the subdirectories under the `/proc/_PID_ /net` directories, and made an unhappy discovery: 

```
al <at> duke:~/linux/trees/vfs$ **ls -lid /proc/{1,2}/net/stat**
        4026531842 dr-xr-xr-x 2 root root 0 Mar 21 19:33 /proc/1/net/stat
        4026531842 dr-xr-xr-x 2 root root 0 Mar 21 19:33 /proc/2/net/stat
```

That discovery prompted a small explosion: 

WE CAN NOT HAVE SEVERAL DENTRIES OVER THE SAME DIRECTORY INODE. […] Sigh... Namespace kinds - there should've been only one... 

Those with a long memory, or at least careful attention when reading [a recent LWN article](<https://lwn.net/Articles/531114/>), might smile with the realization that, to begin with and for many years thereafter, there was only one class of namespace--mount namespaces, as implemented by one Al Viro. 

Humor aside, Al had discovered the origin of the problem. The directory listing above shows two directory entries linked to the same inode. More generally, for all of the processes that share a network namespace, each of the corresponding entries in `/proc/_PID_ /net` is implemented as a hard link to the same (virtual) `/proc` file. 

Implementing corresponding `/proc` entries as hard links to the same inode is a technique used in various places in the implementation of namespaces. Indeed, allowing multiple hard links to a file is a normal feature of UNIX-type systems. Except in one case: Linux, like other UNIX systems, forbids multiple hard links to a directory. The reliability of various pieces of kernel and user-space code is predicated on that assumption. However, [a Linux 2.6.25 patch](<http://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=e9720acd728a46cb40daa52c99a979f7c4ff195c>) made early in the implementation of network namespaces set in train some changes that quietly broke the assumption for the directories under `/proc/_PID_ /net`. 

Having determined the cause of the problem, the developers then needed devise a suitable fix. At this point, pragmatic factors come into play, since the task is not only to fix the kernel going forward, but also going backward. In other words, the ideal solution would be one that could be applied not only to the current kernel source tree and but also to the stable and long-term kernel series. That led Linus [to speculate](<https://lwn.net/Articles/545472/>) about the possibility of allowing an exception to the rule that directory inodes are not allowed to have multiple links. Since the locks in question are placed at the inode level, why not change `lock_rename()` to replace the check on whether that function is dealing with the same dentries with a check on whether it is dealing with the same inodes? 

However, Al was quick [to point out](<https://lwn.net/Articles/545520/>) that while modifying the check would solve the particular deadlock problem found by Dave, other problems would remain. The kernel code that deals with those locks depends upon a [topological sort](<http://en.wikipedia.org/wiki/Topological_sort>) based on the hierarchical relationship between entries in the dcache; the presence of multiple directory entries that link to the same inode renders that sort unreliable. 

Al [went on](<https://lwn.net/Articles/545475/>) to describe what he considered to be the full and proper solution: creating `/proc/_PID_ /net` files as symbolic links to per-network-namespace directories of the form `/proc/_netns-ID_ /net`, where `_netns-ID_` is a per-namespace identifier. [Alternatively](<https://lwn.net/Articles/545477/>), the existing `/proc/_PID_ /net` trees could be kept, but the subdirectories could be created as duplicate subtrees rather than hard links to a single directory subtree. Al was, however, [unsure](<https://lwn.net/Articles/545478/>) about the feasibility of implementing this solution as a patch that could be backported to past stable kernel series. 

In the meantime, Linus came up with another proposal. `proc_get_inode()`, the kernel function for allocating inodes in the `/proc` filesystem, has the following form: 

```
struct inode *proc_get_inode(struct super_block *sb, struct proc_dir_entry *de)
        {
            struct inode *inode = iget_locked(sb, de->low_ino);
    
            if (inode && (inode->i_state & I_NEW)) {
    
                ...
                /* Populate fields in newly allocated cache entry pointed
                   to by 'inode' */
                   ...
    
                unlock_new_inode(inode);
            } else
                pde_put(de);
            return inode;
        }
```

The `iget_locked()` function searches the kernel's inode cache for an inode whose number corresponds to that recorded in the dentry structure `de`. It returns either a pointer to an existing entry, or, if no entry could be found, it allocates a new uninitialized cache entry that it returns to the caller. The `proc_get_inode()` function then populates the fields of the newly allocated inode cache entry using information from the dentry. 

The deadlock problem is a result of the fact that--because multiple dentries map to the same inode--multiple locks may be placed on the same entry in the inode cache. Conversely, deadlocks could be avoided if it was possible to avoid placing multiple locks on the inode entries returned from the cache. As Linus [noted](<https://lwn.net/Articles/545575/>), in the case of `/proc` files, it is not really necessary to find an existing entry in the cache, because there is no on-disk representation for the inodes under `/proc`. Instead, `proc_get_inode()` could simply always create a new cache entry via a call to `new_inode_pseudo()` and populate that cache entry. Since a new cache entry is always created, it will not be visible to any other process, so that there will be no possibility of lock conflicts and deadlocks. In other words, the logic of `proc_get_inode()` can be modified to be: 

```
struct inode *proc_get_inode(struct super_block *sb, struct proc_dir_entry *de)
        {
            struct inode *inode = new_inode_pseudo(sb);
    
            if (inode) {
                inode->i_ino = de->low_ino;
    
                ...
                /* Populate fields in newly allocated cache entry pointed
                   to by 'inode' */
                ...
    
            } else
               pde_put(de);
            return inode;
        }
```

Here, it is worth noting that the kernel uses two different allocation schemes for the inodes under `/proc`: one scheme that is generally employed for inodes under the `/proc/_PID_` directories and another for the inodes in the remainder of `/proc`. Linus's patch affects only inode allocations for entries in the second category. However, as a consequence of the implementation history, whereby `/proc/net` was migrated to `/proc/_PID_ /net`, the inodes under `/proc/_PID_ /net` are allocated in the same fashion as inodes outside `/proc/_PID_` , and so the patch also affects those inodes. 

In the subsequent [commit message](<http://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=51f0885e5415b4cc6535e9cdcc5145bfbc134353>), Linus noted that the patch could have been refined so that the new behavior was applied only to directory entries, rather than all entries, under `/proc`. However, in the interests of keeping the change simple, no such differentiation was made. 

The effect of Linus's patch is to prevent multiple locks (and thus deadlocks) on the same inode. Al [agreed](<https://lwn.net/Articles/545508/>) that the change should not be a problem from a correctness perspective. On the other hand, this change also has the effect of nullifying the benefits of inode caching for `/proc` files outside `/proc/_PID_`. Al wondered about the performance impact of that change. However, some casual instrumentation of the kernel [suggested](<https://lwn.net/Articles/545574/>) that the benefits of inode caching for `/proc` are low in any case. In addition, Dave [reported](<https://lwn.net/Articles/545506/>) that with the fix applied, Trinity was no longer hitting the deadlock problem.
