---
title: Rethinking fsinfo()
url: https://lwn.net/Articles/829212/
date: "August 21, 2020"
category: System calls
author: "By Jonathan Corbet August 21, 2020"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
August 21, 2020

The proposed `fsinfo()` system call, which returns extended information about mounted filesystems, was first [covered here](<https://lwn.net/Articles/792628>) just over one year ago. The form of `fsinfo()` has not changed much in that year, but the debate over merging it continues. To some, `fsinfo()` is needed to efficiently obtain information about filesystems; to others, it is an unnecessary and over-engineered mechanism. Changes will probably be necessary if this feature is ever to make it into the mainline kernel. 

Linux has long supported the `[statfs()](<https://man7.org/linux/man-pages/man2/statfs.2.html>)` system call (usually seen from user space as `[statvfs()](<https://man7.org/linux/man-pages/man3/statvfs.3.html>)`) as a way of obtaining information about mounted filesystems. As has happened so often, though, the designers of `statfs()` made a list of all the filesystem attributes they thought might be interesting and limited the call to those attributes; there is no way to extend it with new attributes. Filesystem designers, though, have stubbornly refused to stop designing new features in the decades since `statfs()` was set in stone, so there is now a lot of relevant information that cannot be obtained from `statfs()`. Such details include mount options, timestamp granularity, associated labels and UUIDs, and whether the filesystem supports features like extended attributes, access-control lists, and case-insensitive lookups. 

As it happens, the kernel does make much of that information available now by way of the `/proc/mounts` virtual file. The problem with `/proc/mounts`, beyond the fact that some information is still missing, is that it is inefficient to access. Reading the contents of that file requires the kernel to query every mounted filesystem for the relevant information; on systems with a lot of mounted filesystems, that can get expensive. Systems running containerized workloads, in particular, can have vast numbers of mounts — thousands in some cases — so reading `/proc/mounts` can be painful indeed. For extra fun, the only way to know about newly mounted filesystems with current kernels is to poll `/proc/mounts` and look for new entries. 

David Howells proposes to solve the polling problem with a new notification mechanism, but that mechanism, in turn, relies on `fsinfo()`, the 21st revision of which was [posted](<https://lwn.net/ml/linux-kernel/159646178122.1784947.11705396571718464082.stgit@warthog.procyon.org.uk/>) on August 3\. Howells requested that both [notifications](<https://lwn.net/ml/linux-kernel/1842689.1596468469@warthog.procyon.org.uk/>) and [`fsinfo()`](<https://lwn.net/ml/linux-kernel/1845353.1596469795@warthog.procyon.org.uk/>) be pulled during the 5.9 merge window, but that did not happen. Instead, the request resulted in yet another discussion about whether `fsinfo()` makes sense in its current form. 

#### fsinfo()

The API for `fsinfo()` is comprehensive and extensible; there should never be a need for an `fsinfo2()` to add new attributes in the future. But it is also complex. On the surface, the interface looks like this: 

```
int fsinfo(int dfd, const char *pathname, const struct fsinfo_params *params,
    	       size_t params_size, void *result_buffer, size_t result_buf_size);
```

Where the `params` structure is defined as: 

```
struct fsinfo_params {
    	__u64	resolve_flags;	/* RESOLVE_* flags */
    	__u32	at_flags;	/* AT_* flags */
    	__u32	flags;		/* Flags controlling fsinfo() specifically */
    	__u32	request;	/* ID of requested attribute */
    	__u32	Nth;		/* Instance of it (some may have multiple) */
    	__u32	Mth;		/* Subinstance of Nth instance */
        };
```

There are four different ways to use `dfd`, `pathname`, and `params->at_flags` to specify which filesystem should be queried; see [this patch changelog](<https://lwn.net/ml/linux-kernel/159646180259.1784947.223853053048725752.stgit@warthog.procyon.org.uk/>) for details. The rest of the `params` structure describes the actual information request; the results end up in `result_buffer`. 

There are numerous possibilities for `params->request`, including: 

  * `FSINFO_ATTR_STATFS` returns more-or-less the same information that would be obtained from `statfs()`. 
  * `FSINFO_ATTR_LIMITS` returns various limits of the filesystem, including maximum file size, inode number, user ID number, hard links to a file, file-name length, etc. These are returned in an `fsinfo_limits` structure. 
  * `FSINFO_ATTR_TIMESTAMP_INFO` yields information about timestamps on files as a set of binary structures; this information includes the maximum values and granularity of timestamps expressed in a unique (to the kernel) mantissa-and-exponent format. 
  * `FSINFO_ATTR_MOUNT_POINT` generates a string showing where the filesystem is mounted. 
  * `FSINFO_ATTR_MOUNT_CHILDREN` gives an array of structures identifying the filesystems mounted below the filesystem being queried. 

The full list of possible requests is rather longer than the above. Each returns data in a different format, usually a specific binary structure for the information requested. For some attributes, a query might return an arbitrary number of elements; in this case, the `Nth` and `Mth` fields in the `fsinfo_params` structure can be used to identify which should be returned. [This patch](<https://lwn.net/ml/linux-kernel/159646187933.1784947.10955424144341330111.stgit@warthog.procyon.org.uk/>) contains a sample program that exercises a number of `fsinfo()` features to produce a listing showing the mount topography of the current system. 

#### Complaints and alternatives

There are a couple of points of resistance to the `fsinfo()` proposal, starting with whether it is needed at all. Linus Torvalds [called it](<https://lwn.net/ml/linux-kernel/CAHk-=wgz5H-xYG4bOrHaEtY7rvFA1_6+mTSpjrgK8OsNbfF+Pw@mail.gmail.com/>) ""engineering for its own sake, rather than responding to actual user concerns"" and wondered why it was needed now after Linux has done without it for so many years. Torvalds tends to worry about adding system calls that end up being used by nobody, so it is not unusual for him to push for justification for the addition of new interfaces. It didn't take long for potential users to make their needs clear; Steven Whitehouse [described it](<https://lwn.net/ml/linux-kernel/066f9aaf-ee97-46db-022f-5d007f9e6edb@redhat.com/>) this way: 

The overall aim is to solve some issues relating to scaling to large numbers of mount in systemd and autofs, and also to provide a generically useful interface that other tools may use to monitor mounts in due course too. Currently parsing /proc/mounts is the only option, and that tends to be slow and is certainly not atomic. 

Karel Zak, maintainer of the util-linux package, [described](<https://lwn.net/ml/linux-kernel/20200813103634.ey2xxwgbn3e4lhdr@ws.net.home/>) the needs of systems with thousands of mount points. Lennart Poettering provided [a long list](<https://lwn.net/ml/linux-kernel/20200814075836.GA230635@gardel-login/>) of attributes he would like to learn about filesystems and why they would be useful. The end result of all this discussion is that the need for some sort of filesystem-information system call is not really in doubt. 

The complexity of `fsinfo()` still gives some developers something to worry about, though; to them, it looks like yet another multiplexer system call that tries to do a large number of things. But it's not entirely clear what an alternative would look like. There was a brief digression in which Torvalds [suggested](<https://lwn.net/ml/linux-kernel/CAHk-=wjzLmMRf=QG-n+1HnxWCx4KTQn9+OhVvUSJ=ZCQd6Y1WA@mail.gmail.com/>) an API where attributes of a file could be opened as if that file were actually a directory; so, for example, opening (with a special flag) `foo/max_file_size` would allow the reading of the maximum file size supported by the filesystem hosting the plain file `foo`. This idea strongly resembles the controversial [approach to metadata implemented by the reiser4 filesystem](<https://lwn.net/Articles/100148/>) back in 2004, though nobody seemed to think it was politic to point that out in the discussion. 

What _was_ pointed out was that there are numerous practical difficulties associated with implementing this sort of mechanism. Even precisely defining its semantics turns out to be hard. So this idea was put aside; it will languish until somebody else surely suggests it again several years from now. 

That leaves open the question of what a new API for obtaining filesystem information should look like. Torvalds [called](<https://lwn.net/ml/linux-kernel/CAHk-=wig0ZqWxgWtD9F1xZzE7jEmgLmXRWABhss0+er3ZRtb9g@mail.gmail.com/>) `fsinfo()` ""confusing and over-engineered"" and [asked](<https://lwn.net/ml/linux-kernel/CAHk-=wh4qaj6iFTrbHy8TPfmM3fj+msYC5X_KE0rCdStJKH2NA@mail.gmail.com/>): ""Can we just make a simple extended statfs() and be done with it, instead of this hugely complex thing that does five different things with the same interface and makes it really odd as a result?"" He further [suggested](<https://lwn.net/ml/linux-kernel/CAHk-=wh5YifP7hzKSbwJj94+DZ2czjrZsczy6GBimiogZws=rg@mail.gmail.com/>) that a number of the binary structures used by `fsinfo()` could be replaced by ASCII data. He [pointed out](<https://lwn.net/ml/linux-kernel/CAHk-=wiUcfgC1PdbS_4mfAj2+VTacOwD_uUu6krSxjpvh42T7A@mail.gmail.com/>) that a number of filesystem interfaces use ASCII for the more complex attributes already and expressed hope that a kernel interface exporting information in ASCII would make life easier for code that is parsing that information out of `/proc/mounts` now. 

So the end result of this discussion is likely to be an attempt to redesign `fsinfo()` along those lines. There is a problem here, though: the information needed is, like the systems it is representing, inherently complex. By the time a `statfs()`-like API that can represent all of this information and which can be extended in the future is designed, chances are that this design will start to look a lot like what `fsinfo()` is now. Replacing a few binary structures with ASCII seems unlikely to change the picture significantly. The end result of this whole exercise may be something that strongly resembles the current design.
