---
title: Concurrency management in BPF
url: https://lwn.net/Articles/779120/
date: "February 7, 2019"
category: BPF
author: "By Jonathan Corbet February 7, 2019"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
February 7, 2019

In the beginning, programs run on the in-kernel BPF virtual machine had no persistent internal state and no data that was shared with any other part of the system. The arrival of eBPF and, in particular, its maps functionality, has changed that situation, though, since a map can be shared between two or more BPF programs as well as with processes running in user space. That sharing naturally leads to concurrency problems, so the BPF developers have found themselves needing to add primitives to manage concurrency (the "exchange and add" or `XADD` instruction, for example). The next step is the addition of a spinlock mechanism to protect data structures, which has also led to some wider discussions on what the BPF memory model should look like. 

A BPF map can be thought of as a sort of array or hash-table data structure. The actual data stored in a map can be of an arbitrary type, including structures. If a complex structure is read from a map while it is being modified, the result may be internally inconsistent, with surprising (and probably unwelcome) results. In an attempt to prevent such problems, Alexei Starovoitov [introduced BPF spinlocks](<https://lwn.net/ml/netdev/20190116050830.1881316-1-ast@kernel.org/>) in mid-January; after a number of quick review cycles, [version 7 of the patch set](<https://lwn.net/ml/netdev/20190131234012.3712779-1-ast@kernel.org/>) was applied on February 1\. If all goes well, this feature will be included in the 5.1 kernel. 

#### BPF spinlocks

BPF spinlocks can only be placed inside structures that, in turn, are stored in BPF maps. Such structures should contain a field like: 

```
struct bpf_spin_lock lock;
```

A BPF spinlock inside a given structure is meant to protect that structure in particular from concurrent access. There is no way to protect other data, such as an entire BPF map, with a single lock. 

From the point of view of a BPF program, this lock will behave much like an ordinary kernel spinlock. An example provided with the patch-set cover letter starts by defining a structure containing a counter: 

```
struct hash_elem {
        	int cnt;
        	struct bpf_spin_lock lock;
        };
```

Code that is compiled to BPF could then increment the counter atomically with something like the following: 

```
struct hash_elem *val = bpf_map_lookup_elem(&hash_map, &key);
        if (val) {
        	bpf_spin_lock(&val->lock);
        	val->cnt++;
        	bpf_spin_unlock(&val->lock);
        }
```

BPF programs run in a restricted environment, so there is naturally a long list of rules that regulate the use of spinlocks. Only certain types of maps (hash, array, and control-group local storage) support spinlocks at all, and only one spinlock is allowed within any given map element. BPF programs can only acquire one lock at a time (to head off deadlock worries), cannot call external functions while holding a lock, and must release a lock prior to returning. Direct access to the `struct bpf_spin_lock` field by the BPF program is disallowed. A number of other rules apply as well; see [this patch](<https://lwn.net/ml/netdev/20190131234012.3712779-2-ast@kernel.org/>) for a more complete list. 

Access to BPF spinlocks from user space is naturally different; a user-space process cannot be allowed to hold BPF spinlocks for unbounded periods of time since that would be an easy way to lock up a kernel thread. The complex [`bpf()` system call](<http://man7.org/linux/man-pages/man2/bpf.2.html>) thus does not get the ability to manipulate BPF spinlocks directly. Instead, it gains a new flag (`BPF_F_LOCK`) that can be added to the `BPF_MAP_LOOKUP_ELEM` and `BPF_MAP_UPDATE_ELEM` operations to cause the spinlock contained within the indicated element to be acquired for the duration of the operation. Reading an element does not reveal the contents of the spinlock field, and updating an element will not change that field. 

One implication of this design is that user space cannot use BPF spinlocks to protect complex changes to structures stored in BPF maps; even the simple counter-incrementing example shown above would not be possible, since the lock cannot be held over the full operation (reading the counter, incrementing it, and storing the result). The implicit assumption seems to be that such manipulations will be done on the BPF side, so the locking functionality serves mostly to keep user space from accessing a structure that a BPF program has partially modified. For example, [a test program](<https://lwn.net/ml/netdev/20190131234012.3712779-10-ast@kernel.org/>) included with the patch set includes a BPF portion that repeatedly picks a random value, then sets every element of an array to that value while holding the lock. The user-space side reads that array under lock and verifies that all elements are the same, thus showing that the element was not read in the middle of an update operation. 

The patch set has seen a number of changes as the result of review comments. One significant added restriction is that BPF spinlocks cannot be used in tracing or socket-filter programs due to preemption-related issues. Those restrictions seem likely to be lifted in the future, but other types of BPF programs (including most networking-related programs) should be able to use BPF spinlocks once the feature goes upstream. 

#### The BPF memory model

In the conversation around [version 4 of the patch set](<https://lwn.net/ml/netdev/20190124041403.2100609-1-ast@kernel.org/>), Peter Zijlstra [asked](<https://lwn.net/ml/netdev/20190124180109.GA27771@hirez.programming.kicks-ass.net/>) about the overall memory model for BPF. In contemporary systems, there is a lot more to concurrency control than spinlocks, especially when the desire is to minimize the cost of that control. Access to shared data can be complicated by the tendency of modern hardware to cache and reorder memory accesses, with the result that changes made on one CPU can appear in a different order elsewhere. Concurrency-aware code may have to make careful use of memory barriers to ensure that changes are globally visible in the right order. 

Such code tends to be tricky when written for a single architecture, but it is further complicated by the fact that, naturally, every CPU type handles these concurrency issues differently. Kernel developers have done a lot of work to hide those differences to the greatest extent possible; details of that work can be found in [Documentation/memory-barriers.txt](<https://www.kernel.org/doc/Documentation/memory-barriers.txt>) and the formal [kernel memory-model specification](<https://lwn.net/Articles/718628/>). All of that work refers to kernel code running natively on the host processor, though, not code running under the BPF virtual machine. As BPF programs run in increasingly concurrent environments, the need to specify the memory model under which they run will grow. 

Starovoitov, who remains the leader of the kernel's BPF efforts, has [proved resistant](<https://lwn.net/ml/netdev/20190126001725.roqqfrpysyljqiqx@ast-mbp.dhcp.thefacebook.com/>) to defining the memory model under which BPF programs run: 

What I want to avoid is to define the whole execution ordering model upfront. We cannot say that BPF ISA is weakly ordered like alpha. Most of the bpf progs are written and running on x86. We shouldn't twist bpf developer's arm by artificially relaxing memory model. BPF memory model is equal to memory model of underlying architecture. What we can do is to make it bpf progs a bit more portable with smp_rmb instructions, but we must not force weak execution on the developer. 

This approach concerns the developers who have gone to a lot of effort to specify what the kernel's memory model should be in general; Will Deacon [said outright](<https://lwn.net/ml/netdev/20190130181100.GA18558@fuggles.cambridge.arm.com/>) that ""I don't think this is a good approach to take for the future of eBPF"". Paul McKenney has [suggested](<https://lwn.net/ml/netdev/20190130183618.GX4240@linux.ibm.com/>) that BPF should simply follow the memory model used by the rest of the kernel. Starovoitov [doesn't want to do that](<https://lwn.net/ml/netdev/20190130195113.xyqre4sxasit6vpu@ast-mbp.dhcp.thefacebook.com/>), though, saying ""tldr: not going to sacrifice performance"". 

That part of the conversation ended without any conclusions beyond a suggestion to talk further about the issue, either in a phone call or at an upcoming conference. It's not clear whether this off-list discussion has happened as of this writing. What seems clear, though, is that these issues are better worked out soon rather than having to be managed in an after-the-fact manner later on. Concurrency issues are hard enough when the underlying rules are well understood; they become nearly impossible when different developers are assuming different rules and code accordingly.
