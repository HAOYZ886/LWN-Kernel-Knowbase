---
title: Private memory for KVM guests
url: https://lwn.net/Articles/890224/
date: "April 7, 2022"
category: "Memfd; Memory management-Address-space isolation; Virtualization-KVM"
author: "By Jonathan Corbet April 7, 2022"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
April 7, 2022

Cloud computing is a wonderful thing; it allows efficient use of computing systems and makes virtual machines instantly available at the click of a mouse or API call. But cloud computing can also be problematic; the security of virtual machines is dependent on the security of the host system. In most deployed systems, a host computer can dig through its guests' memory at will; users running guest systems have to just hope that doesn't happen. There are a number of solutions to that problem under development, including [this KVM guest-private memory patch set](<https://lwn.net/ml/linux-kernel/20220310140911.50924-1-chao.p.peng@linux.intel.com/>) by Chao Peng and others, but some open questions remain. 

A KVM-based hypervisor runs as a user-space process on the host system. To provide a guest with memory, the hypervisor allocates that memory on the host, then uses various [KVM `ioctl()` calls](<https://www.kernel.org/doc/html/latest/virt/kvm/api.html>) to map it into the guest's "physical" address space. But the hypervisor retains its mapping to the memory as well, with no constraints on how the memory can be accessed. Sometimes that access is necessary for communication between the guest and the hypervisor, but the guest would likely want to keep much of that memory to itself. 

#### Providing private memory

The proposed solution to this problem makes use of the kernel's memfd mechanism. The hypervisor can set up a private memory area for its guest by calling [`memfd_create()`](<https://man7.org/linux/man-pages/man2/memfd_create.2.html>) with the new `MFD_INACCESSIBLE` flag. That creates a special type of memfd that the creator can do little with; attempts to read from or write to it will fail, as will attempts to map it into memory. The creator _can_ , though, use [`fallocate()`](<https://man7.org/linux/man-pages/man2/fallocate.2.html>) to allocate (inaccessible) pages to this memfd. If the [`MEMFD_SECRET` flag](<https://lwn.net/Articles/812325/>) is also used at creation time, then the host's direct mapping for the affected pages will be removed, meaning that the host kernel, too, will have no mapping for that memory, making it difficult to access even if the host kernel is compromised. 

The one other thing that can be done with it is to pass it to KVM to map into the guest's virtual address space. The guest will then have full access to this memory, even though the host (which set it up) does not. Enabling this functionality requires setting up callbacks in both directions between KVM and the backing store (probably shmfs) that actually provides the memory. The first set of operations is provided on the KVM side: 

```
struct memfile_notifier_ops {
    	void (*invalidate)(struct memfile_notifier *notifier,
    			   pgoff_t start, pgoff_t end);
    	void (*fallocate)(struct memfile_notifier *notifier,
    			  pgoff_t start, pgoff_t end);
        };
```

The `fallocate()` function will be called whenever memory is mapped into this memory range — when the `fallocate()` system call is used on the host side. It's worth noting that Dave Chinner [objected](<https://lwn.net/ml/linux-kernel/20220310230822.GO661808@dread.disaster.area/>) to this name, so this callback is likely to end up being named something like `notify_populate()` instead. The `invalidate()` callback, instead, is used to indicate that a range of pages has been removed and can no longer be accessed from the guest. 

The other callbacks are supplied by the backing-store implementation to provide KVM with access to the memory in this otherwise inaccessible memfd: 

```
struct memfile_pfn_ops {
    	long (*get_lock_pfn)(struct inode *inode, pgoff_t offset, int *order);
    	void (*put_unlock_pfn)(unsigned long pfn);
        };
```

KVM will call `get_lock_pfn()` to obtain the host page-frame number(s) for one or more pages. When KVM unmaps pages, it calls `put_unlock_pfn()` to inform the backing store that those pages are no longer being used. 

This mechanism, along with the requisite plumbing in KVM, is enough to provide private memory to a guest. The hypervisor will allocate that memory for the guest, but will not be able to access it in any way. 

#### Conversion

Quentin Perret [raised a relevant question](<https://lwn.net/ml/linux-kernel/YjyS6A0o4JASQK+B@google.com/>): what happens when the guest wants to share some of its private memory with the host? This happens reasonably frequently (to set up I/O buffers, for example), so most solutions in this area provide a mechanism for the "conversion" of memory pages between the private and shared states. Perret asked how that was meant to be handled with this mechanism. 

~~The answer, as[provided](<https://lwn.net/ml/linux-kernel/YkHspg+YzOsbUaCf@google.com/>) by Sean Christopherson, is that the guest indicates the desire to convert pages by exiting back into the hypervisor with a `KVM_EXIT_MEMORY_ERROR` status. That status will be passed back to the hypervisor process~~ [**Update** : Thanks to Paolo Bonzini, we have a corrected version of this explanation below. ] 

The answer, as [provided](<https://lwn.net/ml/linux-kernel/YkHspg+YzOsbUaCf@google.com/>) by Sean Christopherson, is that the guest indicates the desire to convert pages with a hypercall. The `KVM_RUN` `ioctl()` immediately returns with a `KVM_EXIT_MEMORY_ERROR` status to the user-space hypervisor process; if it concurs with the request, it responds by unmapping the relevant section of the inaccessible memfd. That, too, is done with `fallocate()`, using the "hole-punch" functionality. The hypervisor can then map ordinary memory into the newly created hole, resulting in a range that is accessible to both sides. 

An important thing to note is that sharing pages back to the host is, by design, a destructive operation; the hole-punch operation will cause the data that was stored there to go away. As Christopherson [described](<https://lwn.net/ml/linux-kernel/YkIFW25WgV2WIQHb@google.com/>), this behavior matches what is done by a number of hardware implementations; pages must be shared with the host _before_ being filled with the data to be shared. Perret, who is working on [a similar mechanism for Android](<https://lwn.net/Articles/836693/>) ("protected KVM" or pKVM), would rather have an in-place conversion mechanism available; without that, he [said](<https://lwn.net/ml/linux-kernel/YkM7eHCHEBe5NkNH@google.com/>), this solution ""might not suit pKVM all that well"". He gave [a list of reasons](<https://lwn.net/ml/linux-kernel/YksIQYdG41v3KWkr@google.com/>) why that would be useful, including: 

> One goal of pKVM is to migrate some things away from the Arm Trustzone environment (e.g. DRM and the likes) and into protected VMs instead. This will give Linux a fighting chance to defend itself against these things -- they currently have access to _all_ memory. And transitioning pages between Linux and Trustzone (donations and shares) is fast and non-destructive, so we really do not want pKVM to regress by requiring the hypervisor to memcpy things. 

Christopherson [questioned](<https://lwn.net/ml/linux-kernel/Ykslo2eo2eRXrpFR@google.com/>) the need for non-destructive conversions, suggesting that reworking pKVM to handle destructive conversions ""doesn't seem too onerous"". Andy Lutomirski was [also unclear](<https://lwn.net/ml/linux-kernel/eefc3c74-acca-419c-8947-726ce2458446@www.fastmail.com/>) on the benefits of that capability, and worried about the difficulty of implementing it correctly: 

> If we actually wanted to support transitioning the same page between shared and private, though, we have a bit of an awkward situation. Private to shared is conceptually easy -- do some bookkeeping, reconstitute the direct map entry, and it's done. The other direction is a mess: all existing uses of the page need to be torn down. If the page has been recently used for DMA, this includes IOMMU entries. 

Perret [reiterated](<https://lwn.net/ml/linux-kernel/Ykwbqv90C7+8K+Ao@google.com/>) his feeling that in-place conversion would perform better, but admitted that he (like all other participants in the discussion) has not yet collected the numbers to prove that one way or the other. He also doesn't have the details of in-place conversion worked out, though he proposed an outline for how it could work. 

As of this writing the conversation is ongoing with no clear resolution in sight. The developers involved all have an interest in creating a mechanism that will work for all use cases; there is little interest in adding several private-memory implementations. But they all want to also get the best performance they can while avoiding excess complexity. Reconciling objectives like these is at the core of system programming (and, for that matter, most other types of programming) and is something that the kernel community is usually reasonably good at — at least, if all of the interested parties are participating in the discussion. The developers have begun to talk so, with luck, a workable solution can be expected to emerge from this conversation, but it may take a while yet.
