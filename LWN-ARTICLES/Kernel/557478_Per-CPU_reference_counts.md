---
title: "Per-CPU reference counts"
url: https://lwn.net/Articles/557478/
date: "July 3, 2013"
category: Reference counting
author: "By Jonathan Corbet July 3, 2013"
---

By **Jonathan Corbet**  
July 3, 2013

Reference counting is used by the kernel to know when a data structure is unused and can be disposed of. Most of the time, reference counts are represented by an `atomic_t` variable, perhaps wrapped by a structure like a `kref`. If references are added and removed frequently over an object's lifetime, though, that `atomic_t` variable can become a performance bottleneck. The 3.11 kernel will include a new per-CPU reference count mechanism designed to improve scalability in such situations. 

This mechanism, created by Kent Overstreet, is defined in `<linux/percpu-refcount.h>`. Typical usage will involve embedding a `percpu_ref` structure within the data structure being tracked. The counter must be initialized with: 

```
int percpu_ref_init(struct percpu_ref *ref, percpu_ref_release *release);
```

Where `release()` is the function to be called when the reference count drops to zero: 

```
typedef void (percpu_ref_release)(struct percpu_ref *);
```

The call to `percpu_ref_init()` will initialize the reference count to one. References are added and removed with: 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

```
void percpu_ref_get(struct percpu_ref *ref);
        void percpu_ref_put(struct percpu_ref *ref);
```

These functions operate on a per-CPU array of reference counters, so they will not cause cache-line bouncing across the system. There is one potential problem, though: `percpu_ref_put()` must determine whether the reference count has dropped to zero and call the `release()` function if so. Summing an array of per-CPU counters would be expensive, to the point that it would defeat the whole purpose. This problem is avoided with a simple observation: as long as the initial reference is held, the count cannot be zero, so `percpu_ref_put()` does not bother to check. 

The implication is that the thread which calls `percpu_ref_init()` must indicate when it is dropping its reference; that is done with a call to: 

```
void percpu_ref_kill(struct percpu_ref *ref);
```

After this call, the reference count degrades to the usual model with a single shared `atomic_t` counter; that counter will be decremented and checked whenever a reference is released. 

The performance benefits of a per-CPU reference count will clearly only be realized if most of the references to an object are added or removed while the initial reference is held. In practice that is often the case. This mechanism has found an initial use in the control group code; the comments in the header file claim that it is used by the asynchronous I/O code as well, but that is not the case in the current mainline.
