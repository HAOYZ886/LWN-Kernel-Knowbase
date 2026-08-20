---
title: Namespace reference counting and listns()
url: https://lwn.net/Articles/1043824/
date: "November 3, 2025"
category: "Namespaces; Releases-6.19; System calls-listns"
author: "By Jonathan Corbet November 3, 2025"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
November 3, 2025

The kernel's [namespaces feature](<https://lwn.net/Articles/531114/>) is, among other things, a key part of the implementation of containers. Like much in the kernel, though, the namespace API evolved over time; there was no design at the outset. As a result, this API has some rough edges and missing features. Christian Brauner is working to straighten out the namespace situation somewhat with [this daunting 72-part patch series](<https://lwn.net/ml/all/20251029-work-namespace-nstree-listns-v4-0-2e6f823ebdc0@kernel.org>) that, among other things, adds a new system call to allow user space to query the namespaces present on the system. 

The original namespace type, now called [mount namespaces](<https://lwn.net/Articles/689856/>), was [introduced by Al Viro](<https://lore.kernel.org/lkml/Pine.GSO.4.21.0102242253460.24312-100000@weyl.math.psu.edu/>) as just "namespaces" in 2001 (they were [briefly covered in LWN](</2001/0301/kernel.php3>) at the time). UTS namespaces (which provide a different view of the system's host name), process-ID namespaces (managing the visibility of processes), and IPC namespaces (controlling the view of the System V inter-process communication features) followed as part of the 2.6.19 release in 2006. Each namespace type was added when the need arose and somebody was moved to implement it. As the use of namespaces has grown, though, some of the problems in their implementation have become more apparent. 

#### Reference counts

For example, namespaces have complicated lifecycle requirements. A namespace must obviously continue to exist as long as there are any processes that are running within it. For many years, a namespace would automatically be deleted once the last process running within it exited. Over time, the ability to keep an empty namespace around (by opening a file descriptor referencing it or bind-mounting it into the filesystem) was added. Some namespaces ([user namespaces](<https://lwn.net/Articles/532593/>), for example) are hierarchical; if a hierarchical namespace contains children, that namespace will, once again, remain within the system. 

The kernel uses a reference count on each namespace to know when it is no longer in use. C is not an object-oriented language, so there is no class hierarchy for namespaces; each namespace type is a different structure. There is a "superclass", of sorts, in the `ns_common` structure, which all of the other namespace structure types include; it was first [added by Viro](<https://git.kernel.org/linus/435d5f4bb2cc>) to the 3.19 release in 2014. The reference count was [moved into `struct ns_common`](<https://git.kernel.org/linus/2024f91e965f>) by Brauner for 5.9 in 2020. That structure went through some significant changes in the 6.18 merge window to reach [its current form](<https://elixir.bootlin.com/linux/v6.18-rc3/source/include/linux/ns_common.h#L40>), which includes a reference count now named `__ns_ref`. 

This count tracks _all_ references to a given namespace; that includes all of the types listed above, but there are others as well. The kernel will often create internal references to namespaces that can cause them to persist for a period after they are no longer used and, in theory, no longer visible to user space. There is an interesting interaction with a different feature [added for 6.18](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=3ab378cfa793>), though: the ability to refer to namespaces with file handles. A file handle is an opaque binary cookie that can be used to open an object without locating it in the filesystem; see [the `open_by_handle_at()` man page](<https://man7.org/linux/man-pages/man2/open_by_handle_at.2.html>) for details. 

Since a file handle is not contained within a filesystem, it can outlive the object to which it refers. Or, in the case of a namespace, it can outlive the _visibility_ of the object to which it refers. A namespace may have gone completely out of use and exist only because of internal kernel references that will, presumably, go away soon; normally, user space would not be able to open this namespace. But that changes if user space has a file handle referring to the namespace; opening that file handle will result in "resurrecting" the namespace when it was otherwise on its way out. 

That is the sort of API quirk that nobody asked for and nobody intended to implement. If, however, it is allowed to continue to exist, somebody — attackers if nobody else — will surely find a way to depend on it. Eliminating this quirk is the first objective of Brauner's series. 

Normally, a reference-counted structure contains a _single_ reference count; these patches go against that practice by adding a second count, called `__ns_ref_active`, to `struct ns_common`. This count tracks the number of "active" references, which are essentially the references visible to user space. It can be thought of as a subset of `__ns_ref`, in that any change to `__ns_ref_active` should be accompanied by an equal change to `__ns_ref`. Internal references created by the kernel, though, will only increment `__ns_ref`. In the end, `__ns_ref` still manages the lifetime of the namespace, while `__ns_ref_active` manages its visibility to user space. 

So, for example, an attempt to open a namespace with `open_by_handle_at()` will fail if `__ns_ref_active` is zero, even if the namespace itself still exists within the kernel. It will no longer be possible to use file handles to bring namespaces back from the dead. 

#### Listing namespaces

As kernel developers added namespaces over the last 24 years, none of them ever quite got around to implementing a way to see which namespaces are active in the system. In current kernels, the only way to get a complete list of namespaces is to go rummaging through the `/proc/_PID_ /ns` directories for every process in the system. Needless to say, that is less than optimally efficient. It also still doesn't get a full list, since any namespaces that are empty but which are kept around with, for example, a bind mount, are not present in any process's `/proc` directory and will consequently be missed. 

The obvious answer is to add a new system call that allows iterating through the active namespaces; in this series, that is `listns()`: 

```
struct ns_id_req {
            __u32 size;         /* sizeof(struct ns_id_req) */
            __u32 spare;        /* Reserved, must be 0 */
            __u64 ns_id;        /* Last seen namespace ID (for pagination) */
            __u32 ns_type;      /* Filter by namespace type(s) */
            __u32 spare2;       /* Reserved, must be 0 */
            __u64 user_ns_id;   /* Filter by owning user namespace */
        };
    
        ssize_t listns(const struct ns_id_req *req, u64 *ns_ids,
                       size_t nr_ns_ids, unsigned int flags);
```

A caller starts by filling in an `ns_id_req` structure describing the information request. The `size` field is the size of the structure itself, allowing for expansion in the future if need be. The `ns_type` field is a bitmask of the namespace types of interest; `MNT_NS` for mount namespaces, for example, or `NET_NS` for network namespaces. If only children of a given namespace are of interest, the parent's namespace ID can be put into `user_ns_id`. The `ns_id` field should be zero for the first call. 

The actual `listns()` call takes a pointer to that structure, an array (`ns_ids`) to store the returned namespace IDs, and the length of that array (`nr_ns_ids`). The `flags` field must be zero in the current implementation. This call will fill in the `ns_ids` array with matching namespace IDs, returning the number of IDs that were put there. If the number of matching namespaces is too large to fit in the provided array, a subsequent `listns()` call can pick up where this one left off by placing the final ID returned by the previous call in the `ns_id` field of the `ns_id_req` structure. 

Needless to say, `listns()` will only return namespaces that have an `__ns_ref_active` count greater than zero. For the curious, there are several examples of how `listns()` can be used in the above-linked cover letter. It is also worth noting that, while this patch series is long, patches 22 through 72 are all tests for the new functionality; they also provide examples of how it is expected to be used. 

This series is in its fourth revision; the rate of change so far suggests that there might be another round or two in store before it is ready to go, but there do not appear to be any fundamental objections to this work. While Brauner has not indicated when he plans to send these changes upstream, it seems reasonable to expect to see them during the 6.19 merge window.
