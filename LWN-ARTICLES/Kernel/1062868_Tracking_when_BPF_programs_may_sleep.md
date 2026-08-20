---
title: Tracking when BPF programs may sleep
url: https://lwn.net/Articles/1062868/
date: "March 23, 2026"
category: BPF
author: "By Daroc Alden March 23, 2026"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Daroc Alden**  
March 23, 2026

BPF programs can run in both sleepable and non-sleepable (atomic) contexts. Currently, sleepable BPF programs are not allowed to enter an atomic context. Puranjay Mohan has a [ new patch set](<https://lwn.net/ml/all/20260226161500.775715-1-puranjay@kernel.org/>) that changes that. The patch set would let BPF programs called in sleepable contexts temporarily acquire locks that cause the programs to transition to an atomic context. BPF maintainer Alexei Starovoitov objected to parts of the implementation, however, so acceptance of the patch depends on whether Mohan is willing and able to straighten it out. 

In an atomic context, kernel code is not allowed to do anything that would delay the continued execution of the kernel, such as waiting for block I/O or faulting a page back into memory. It is usually up to the kernel programmer (assisted by the kernel's various forms of instrumentation) to make sure that they don't accidentally call a function that can sleep in such a context. BPF programs were originally only capable of running in atomic contexts, and were therefore never allowed to call functions that could sleep. In 2020, the BPF verifier [was extended](<https://lwn.net/Articles/825415/>) to handle BPF programs that could sleep (by marking the entire program with a special flag), but such programs were not permitted to call many of the existing BPF interfaces, which assumed they could transition to an atomic context. 

The main advantage of marking a BPF program as sleepable is that it is allowed to copy data from user space (which can sleep if the data needs to be faulted back into memory). Since that is generally useful, it would be nice if more BPF programs could be marked as sleepable; currently, many cannot because they need to take locks or acquire resources that are only available in an atomic context. Mohan's patch set allows for more fine-grained accounting of contexts in BPF programs by having the BPF verifier track whether the program is allowed to sleep on an instruction-by-instruction basis instead of globally for the whole program. 

The BPF verifier tracks which kernel resources a BPF program has access to at a given point by looking for kfuncs (kernel functions to which BPF programs have access) that are annotated with the `KF_ACQUIRE` and `KF_RELEASE` markers. When a program calls a kfunc with the `KF_ACQUIRE` marker, the verifier tracks the return value and ensures that the program eventually passes it back to a compatible `KF_RELEASE` kfunc. A preparatory patch in Mohan's series adds support for these flags to BPF iterators. The main patches add another marker, `KF_FORBID_FAULT`, that tells the verifier that the program is not allowed to sleep as long as it holds a reference to the acquired resource. The intention is just to prevent page faults (hence the name), but the implementation forbids all kinds of sleeps (of which page faults are a subset). `KF_FORBID_FAULT` can only be used on kfuncs that are already marked with `KF_ACQUIRE`. Once the resource is released, the program is allowed to sleep again. 

In his cover letter, Mohan gave an example of when this increased granularity might be useful. The `task_vma` iterator lets BPF programs iterate over a task's virtual memory areas (VMAs) — but doing anything with that information is difficult, because the iterator yields [`vm_area_struct`](<https://elixir.bootlin.com/linux/v6.19.8/source/include/linux/mm_types.h#L892>) structures. Those structures only remain valid as long as `mmap_lock` is held. Taking that lock creates a context in which page faults are forbidden, since page-fault handling may need to take the same lock. With his changes, BPF programs can now read from the VMA to obtain a user-space pointer, and then explicitly release the VMA structure to exit the atomic context and interact with user space. (Although, of course, the VMA in question could be unmapped by another CPU as soon as the lock is released, so the program needs to be able to cope with failure.) 

```
bpf_for_each(task_vma, vma, task, 0) {
            u64 start = vma->vm_start;
    
            /* Faulting forbidden, but VMA pointer access allowed */
            
            bpf_iter_task_vma_release(&___it);
            
            /* mmap_lock released, VMA pointer invalidated */
            /* Faulting (and sleeping) is fine here. */
    
            bpf_copy_from_user(&buf, sizeof(buf), (void *)start);
        }
```

An [ earlier version](<https://lwn.net/ml/all/20260223174659.2749964-1-puranjay@kernel.org/>) of the patch set called the new kfunc marker `KF_FORBID_SLEEP`, but Starovoitov had [ concerns about the name and semantics](<https://lwn.net/ml/all/CAADnVQ+aobVwR1m-v2w_RWFxG_9TwkYQ-K8KtC6D18HrYe8Dig@mail.gmail.com/>). `KF_ACQUIRE` is also used for things other than locks, particularly for reference-counted resources; Starovoitov suggested differentiating between `KF_ACQUIRE` (for reference counts) and `KF_ACQUIRE_LOCK` (for actual locks), and merging the semantics of `KF_FORBID_SLEEP` into the latter. 

Mohan [ was fine with that suggestion](<https://lwn.net/ml/all/CANk7y0hLknrHo+UB2HDPeen6nLxRaBdLNKiMBiMBv7Hh-QzPEw@mail.gmail.com/>), but Eduard Zingerman [ thought](<https://lwn.net/ml/all/a8fbc7a3669ad1014b2623c42219d3d074dfbc8f.camel@gmail.com/>) it might be worth exploring a more radical change. The verifier currently tracks four kinds of resources that all forbid sleeping, but with different acquire/release logic: active interrupts, active RCU locks, active preemption locks, and other active locks. The list of which kfunc corresponds to which kind of acquire/release logic is hard-coded; Zingerman suggested that, if Mohan was already going to make changes to the meaning of `KF_ACQUIRE`, it might be worthwhile to explore separate markers for these four categories to make the verifier's logic more generic. 

Mohan said that exploring the possibility is now [ next on his agenda](<https://lwn.net/ml/all/CANk7y0iR5sJkP99ztihrT6DDAp-DA4nZ2heR2kNXgUnKVURWAg@mail.gmail.com/>) after the current patch set is done. For now, the name of the new annotation was changed to `KF_FORBID_FAULT` to indicate a narrower intended use. Mohan's follow-up work will look at refactoring the kfunc flags to allow for more precisely identifying the type of lock being used. That may take some time, however, because Starovoitov [ still has problems](<https://lwn.net/ml/all/CAADnVQJjzd_r6SjXaxgz-OuJbpP7DkBM3DuspCgd8Nfy4M6iMA@mail.gmail.com/>) with the implementation of the latest version of the patch set. ""Sorry. This is no go. We have to go back to the drawing board with the whole thing."" 

Starovoitov specifically objected to the way that Mohan's code repurposed the `id` field of the structure that the verifier uses to track stack slots containing references to iterators. There are already several slightly different uses of IDs across the verifier — something Amery Hung is [ working on cleaning up](<https://lwn.net/ml/all/20260307064439.3247440-1-ameryhung@gmail.com/>) — and Starovoitov doesn't want to add another one that is specific to iterators when the patch set is supposed to add a generic mechanism. 

Mohan has not yet responded to Starovoitov's concerns with a new version of the patch set, but he has submitted [ another](<https://lwn.net/ml/all/20260311225726.808332-1-puranjay@kernel.org/>) patch set that changes the `task_vma` iterator to use per-VMA locking instead of `mmap_lock`. The iterator copies the VMA and drops the lock before providing the VMA to the BPF program, making the iterator usable in sleepable contexts. That patch set is still undergoing revision, but it could solve the specific problem of using `task_vma` iterators in a sleepable context. Hopefully the more general mechanism (and Zingerman's suggested cleanup) will still be a priority for Mohan afterward, even if the newer patch set meets his needs.
