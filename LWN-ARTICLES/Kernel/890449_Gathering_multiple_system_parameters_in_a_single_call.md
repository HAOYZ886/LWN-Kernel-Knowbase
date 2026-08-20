---
title: Gathering multiple system parameters in a single call
url: https://lwn.net/Articles/890449/
date: "April 6, 2022"
category: System calls
author: "By Jake Edge April 6, 2022"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jake Edge**  
April 6, 2022

Running a command like [`lsof`](<https://man7.org/linux/man-pages/man8/lsof.8.html>), which lists the open files on the system along with information about the process that has each file open, takes a lot of system calls, mostly to read a small amount of information from many `/proc` files. Providing a new interface to collect those calls together into a single (or, at least, fewer) system calls is the target of Miklos Szeredi's [`getvalues()` RFC patch](<https://lwn.net/ml/linux-kernel/20220322192712.709170-1-mszeredi@redhat.com/>) that was posted on March 22\. While the proposal does not look like it is going far, at least in its current form, it did spark some discussion of the need—or lack thereof—for a way to reduce this kind of overhead, as well as to explore some alternative ways to get there via code that already exists in the kernel. 

#### `getvalues()`

In his post, Szeredi highlighted the performance problem: ""Calling open/read/close for many small files is inefficient"". Running `lsof` on his desktop resulted in around 60,000 calls to read small amounts of data from `/proc` files; ""90% of those are 128 bytes or less"". But another problem that `getvalues()` tries to address is the fragmentation of the interfaces for gathering system information on Linux: 

> For files we have basic stat, statx, extended attributes, file attributes (for which there are two overlapping ioctl interfaces). For mounts and superblocks we have stat*fs as well as /proc/$PID/{mountinfo,mountstats}. The latter also has the problem on not allowing queries on a specific mount. 

His proposed solution is a system call with the following prototype, which uses a new structure type: 

```
struct name_val {
    	    const char *name;		/* in */
    	    struct iovec value_in;	/* in */
    	    struct iovec value_out;	/* out */
    	    uint32_t error;		/* out */
    	    uint32_t reserved;
        };
    
        int getvalues(int dfd, const char *path, struct name_val *vec, size_t num,
    	          unsigned int flags);
```

It will look up an object (which he calls `$ORIGIN`) using `dfd` and `path`, as with [`openat()`](<https://man7.org/linux/man-pages/man2/open.2.html>); `flags` is used to modify the `path`-based lookup. `vec` is an array of `num` entries for the parameters of interest. `getvalues()` will return the number of values filled in or an error. 

The `name` field in `struct name_val` is where most of the action is. It consists of a string in a kind of new micro-language that describes the value of interest, using prefixes to identify different types of information. From the post: 

> 

```
>     mnt                    - list of mount parameters
>     mnt:mountpoint         - the mountpoint of the mount of $ORIGIN
>     mntns                  - list of mount ID's reachable from the current root
>     mntns:21:parentid      - parent ID of the mount with ID of 21
>     xattr:security.selinux - the security.selinux extended attribute
>     data:foo/bar           - the data contained in file $ORIGIN/foo/bar
>
```

The prefix can be omitted if it is the same as that of the previous entry in `vec`, so a "`mnt:mountpoint`" followed by a "`:parentid`" would imply the "`mnt`" prefix on the latter. `value_in` provides a buffer to hold the value retrieved; passing a NULL for `iov_base` in the [`struct iovec`](<https://elixir.bootlin.com/linux/latest/source/include/uapi/linux/uio.h#L17>) will reuse the previous entry's buffer. That allows a single buffer to be used for multiple retrieved values with `getvalues()` stepping through the buffer as needed. `value_out` will hold the address of where the value was stored, which is useful for shared buffers, and its length. If an error occurs, its code will be stored in `error`. 

It is a fairly straightforward interface, though it does add yet another (simple) parser into the kernel. Szeredi also posted a [sample program](<https://lwn.net/ml/linux-kernel/CAOssrKcJT=HCYW1tUwUaZbD8gi2CNh4__7Q6kqTPsdxF07dmBQ@mail.gmail.com/>) that shows how it can be used. 

#### Reaction

Casey Schaufler [pointed out](<https://lwn.net/ml/linux-kernel/f80f372b-4249-eb25-ed95-9f8615877745@schaufler-ca.com/>) that the open/read/close problem could be addressed without all of the rest of the generality with a `openandread()` system call or similar. He also had some questions and comments about the interface, some of its shortcuts, and its behavior in the presence of errors. Greg Kroah-Hartman [noted](<https://lwn.net/ml/linux-kernel/YjrJWf+XMnWVd6K0@kroah.com/>) that he had [posted](<https://lwn.net/ml/linux-kernel/20200704140250.423345-1-gregkh@linuxfoundation.org/>) a proposal for a `readfile()` system call that would address the overhead problem as well. It was the subject of an [LWN article](<https://lwn.net/Articles/813827/>) just over two years ago. But it turns out that he found little real-world performance improvement using `readfile()`, which is part of why it was never merged. ""Do you have anything real that can use this that shows a speedup?"". 

Bernd Schubert [thought](<https://lwn.net/ml/linux-kernel/d0e2573a-7736-bb3e-9f6a-5fa25e6d31a2@ddn.com/>) that network filesystems could benefit, because operations could be batched up rather than sent individually over the wire. He said that because there is no `readfile()` (or its equivalent) available, network filesystem protocols are not adding combined operations for open/read/close. But J. Bruce Fields [said](<https://lwn.net/ml/linux-kernel/20220323192958.GA18275@fieldses.org/>) that NFSv4 already has compound operations, ""so you can do OPEN+READ+CLOSE in a single round trip"". So far, at least, the NFS client does not actually use it, but the protocol support is there. 

While Christian Brauner [was in favor](<https://lwn.net/ml/linux-kernel/20220323114215.pfrxy2b6vsvqig6a@wittgenstein/>) of better ways to query filesystem information, he was concerned about the ease-of-use for `getvalues()`: 

> I would really like if we had interfaces that are really easy to use from userspace comparable to statx for example. I know having this generic as possible was the goal but I'm just a bit uneasy with such interfaces. They become cumbersome to use in userspace. 
> 
> [...] Would it be really that bad if we added multiple syscalls for different types of info? For example, querying mount information could reasonably be a more focussed separate system call allowing to retrieve detailed mount propagation info, flags, idmappings and so on. Prior approaches to solve this in a completely generic way have gotten us not very far too so I'm a bit worried about this aspect too. 

But Szeredi [thinks](<https://lwn.net/ml/linux-kernel/CAJfpegsCKEx41KA1S2QJ9gX9BEBG4_d8igA0DT66GFH2ZanspA@mail.gmail.com/>) that the generality of the interface is important for the future. A system call like [`statx()`](<https://man7.org/linux/man-pages/man2/statx.2.html>) could perhaps be added for filesystem information (e.g. `statfsx()`), but that only works for data that can be represented in a flat structure. Hierarchical data has to be represented in some other way. He would like to see some kind of unified interface to gather information from multiple different sources in the kernel, both textual and binary, that uses hierarchical namespaces (a la file paths) for data that does not have a flat structure—rather than a collection of ad hoc interfaces that get added over time. 

Kroah-Hartman pointed to two different mechanisms that might be used, [starting](<https://lwn.net/ml/linux-kernel/Yjsiv2XesJRzoeTW@kroah.com/>) with the KVM [`binary_stats.c`](<https://elixir.bootlin.com/linux/latest/source/virt/kvm/binary_stats.c>) interface, ""which tried to create a 'generic' api, but ended up just making something to work for KVM as they got tired of people ignoring their more intrusive patch sets"". But Szeredi [said](<https://lwn.net/ml/linux-kernel/CAJfpegsBmed6dchjgVeQ-OPGYBiU+2GPgsoJegjuPTrcLs6-8g@mail.gmail.com/>) that the KVM mechanism would not be easily used for things like extended attributes (xattrs) that do not have a fixed size. Kroah-Hartman followed that up with a [suggestion](<https://lwn.net/ml/linux-kernel/YjwWHk9YYGrb6i07@kroah.com/>) to look at [varlink](<https://varlink.org/>) as a possible protocol for transferring the data. 

Ted Ts'o [was not sure](<https://lwn.net/ml/linux-kernel/YjudB7XARLlRtBiR@mit.edu/>) what problem `getvalues()` was truly solving. He noted that an `lsof` on his laptop did not take an inordinate amount of time, so the performance argument does not really make sense to him. As for ease-of-use, he suggested adding user-space libraries that gather up the data from various sources ""to make life easier for application programmers"". He had other concerns as well: 

> Each new system call, especially with all of the parsing that this one is going to use, is going to be an additional attack surface, and an additional new system call that we have to maintain --- and for the first 7-10 years, userspace programs are going to have to use the existing open/read/close interface since enterprise kernels stick [around] for a L-O-N-G time, so any kind of ease-of-use argument isn't really going to help application programs until RHEL 10 becomes obsolete. 

If the open/read/close problem is real for some filesystems (e.g. network or FUSE), Christoph Hellwig [said](<https://lwn.net/ml/linux-kernel/YjwQ4dLH7BWOqZqr@infradead.org/>), a better way to address it would be with an [io_uring](<https://lwn.net/Articles/810414/>) operation. ""And even on that I need to be sold first."" The `readfile()` article linked above also has a section on a mechanism to support that use case with io_uring. 

Linus Torvalds [was skeptical](<https://lwn.net/ml/linux-kernel/CAHk-=wibqjNiLgnsL2LcTDKqnKyQ17rhHJv-qkHY1w93LJuXjA@mail.gmail.com/>) of the whole concept. Coalescing the open/read/close cycle has been shown to make little difference from a performance standpoint, and he did not think that the more general query interface was particularly compelling either: 

> With the "open-and-read" thing, the wins aren't that enormous. 
> 
> And getvalues() isn't even that. It's literally a [specialty] interface for a very special thing. Yes, I'm sure it avoids several system calls. Yes, I'm sure it avoids parsing strings etc. But I really don't think this is something we want to do unless people can show enormous and real-world examples of where it makes such a huge difference that we absolutely have to do it. 

#### Virtual xattrs?

Dave Chinner [pointed out](<https://lwn.net/ml/linux-kernel/20220323225843.GI1609613@dread.disaster.area/>) that the XFS filesystem has a somewhat similar [`ioctl()`](<https://man7.org/linux/man-pages/man2/ioctl.2.html>) command (`XFS_IOC_ATTRMULTI_BY_HANDLE`) that is used to dump and restore extended attributes in batches. He suggested that idea could be further extended: 

> I've said in the past when discussing things like statx() that maybe everything should be addressable via the xattr namespace and set/queried via xattr names regardless of how the filesystem stores the data. The VFS/filesystem simply translates the name to the storage location of the information. It might be held in xattrs, but it could just be a flag bit in an inode field. 
> 
> Then we just get named xattrs in batches from an open fd. 

He said that the values that Szeredi envisions being available via `getvalues()` could simply be mapped into an xattr namespace and retrieved using ""a new, cleaner version of xattr batch APIs that have been around for 20-odd years already"". Schaufler [cautioned](<https://lwn.net/ml/linux-kernel/4080a088-8d4a-9631-3374-ded001d35c58@schaufler-ca.com/>) that there is a ""significant and vocal set of people who dislike xattrs passionately"", but if that problem could be solved, Chinner's approach had a lot going for it. ""You could even provide getvalues() on top of it."" 

Szeredi [seemed amenable](<https://lwn.net/ml/linux-kernel/CAJfpegv6PmZ_RXipBs9UEjv_WfEUtTDE1uNZq+9fBkCzWPvXkw@mail.gmail.com/>) to the idea, though he wondered about information from elsewhere in the system. Amir Goldstein [said](<https://lwn.net/ml/linux-kernel/CAOQ4uxhbm2mtTp8PmgEq5KmwTe0n6MRRGhShXM=Ot6Bz87HXjA@mail.gmail.com/>) that there is already precedence for "virtual xattrs" in the CIFS filesystem, so that idea could be extended to mount information and statistics of various kinds: ""I don't see a problem with querying attributes of a mount/sb the same way as long as the namespace is clear about what is the object that is being queried (e.g. getxattr(path, "fsinfo.sbiostats.rchar",...)."" 

Chinner also [noted](<https://lwn.net/ml/linux-kernel/20220324203116.GJ1609613@dread.disaster.area/>) that using the xattr interface would provide ""a symmetrical API for -changing- values"". Instead of using some other mechanism (e.g. configfs) to change system parameters, they could be done with a [`setxattr()`](<https://man7.org/linux/man-pages/man2/setxattr.2.html>) call. ""That retains the simplicity of proc and sysfs attributes in that you can change them just by writing a new value to the file...."" 

The discussion more or less wound down after that. The xattrs-based idea seemed reasonably popular and much of the infrastructure to use it is already present in the kernel in various forms. So, while `getvalues()` itself does not have a path toward merging, seemingly, the idea behind it could perhaps be preserved in a somewhat different form. So far, patches for that have not appeared, but perhaps that is something we will see before too long.
