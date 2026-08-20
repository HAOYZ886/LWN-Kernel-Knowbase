---
title: Waiting for entropy
url: https://lwn.net/Articles/724643/
date: "June 6, 2017"
category: "Random numbers; Security-Random number generation"
author: "By Jonathan Corbet June 6, 2017"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
June 6, 2017

Many bytes have been expended over the years discussing the virtues of the kernel's random number generation subsystem. One of the biggest recurring concerns has to do with systems that are unable to obtain sufficient entropy during the boot process to meet early demands for random data. The latest discussion on this topic got off to a bit of a rough start, but it may lead to an incremental improvement in this area. 

Jason Donenfeld [started the thread](<https://lwn.net/Articles/724644/>) with a complaint that `/dev/urandom` will, when read from user space, return data even if the kernel's internal entropy pool has not yet been properly seeded. In such a case, it is theoretically possible for an attacker to predict the not-so-random data that will be returned. He asserted that `/dev/urandom` should simply block until the entropy pool is ready, and dismissed the reasoning behind the current behavior: ""Yes, yes, you have arguments for why you're keeping this pathological, but you're still wrong, and this api is still a bug."" 

Bug or not, as Ted Ts'o [pointed out](<https://lwn.net/Articles/724645/>), making `/dev/urandom` block causes distributions like Ubuntu and OpenWrt to fail to boot. That sort of behavioral change is typically called a "regression", and regressions of this sort are not normally allowed. So `/dev/urandom` will retain its current behavior. But that isn't the point Donenfeld was really trying to address anyway. The real issue, as it turns out, has to do with getting random data from within the kernel instead of from user space. That can be done with a call to: 

```
void get_random_bytes(void *buf, int nbytes);
```

This function will place `nbytes` of random data into the buffer pointed to by `buf`; it will do so regardless of whether the entropy pool is fully initialized. So, once again, it is possible to get data that is not truly random. Since this function is called from inside the kernel, those calls can happen early in the boot process, so the chance of encountering an insufficiently random entropy pool are relatively high. 

This problem is not unknown to the kernel development community, of course. In 2015, Stephan Mueller [proposed](<https://lwn.net/Articles/724650/>) the addition of a version of `get_random_bytes()` that would block until the entropy pool is ready, should that be necessary. That idea ran into trouble, though, when Herbert Xu [pointed out](<https://lwn.net/Articles/724651/>) that it could lead to deadlocks — just the sort of random event that tends _not_ to be of interest. So, instead, a callback interface was created. Kernel code that wants to ensure that it gets good random data starts by creating a callback function and placing a pointer to that function in a `random_ready_callback` structure: 

```
struct random_ready_callback {
    	struct list_head list;
    	void (*func)(struct random_ready_callback *rdy);
    	struct module *owner;
        };
```

That structure is then passed to `add_random_ready_callback()`: 

```
int add_random_ready_callback(struct random_ready_callback *rdy);
```

When the random-number subsystem is ready, the given callback function will be called. By adding some more structure (most likely using a completion), the calling code can create something that looks like a synchronous function to get random data. 

As Donenfeld pointed out, this interface is a little bit on the cumbersome side, which may have something to do with the fact that it has exactly one call site in the kernel. He suggested that it might make sense to add a synchronous interface that could be used in at least some situations; that would make it possible to fix some places in the kernel that are at risk of using nonrandom data. Ts'o [agreed](<https://lwn.net/Articles/724648/>) that this approach might make sense: 

Or maybe we can then help figure out what percentage of the callsites can be fixed with a synchronous interface, and fix some number of them just to demonstrate that the synchronous interface does work well. 

The end result was [a patch series](<https://lwn.net/Articles/724599/>) from Donenfeld adding a new function: 

```
int wait_for_random_bytes(bool is_interruptable, unsigned long timeout);
```

As its name might suggest, `wait_for_random_bytes()` will wait until random data is available. If `is_interruptable` is set, the function will return early (with an error code) should the calling process receive a signal. The `timeout` parameter can be used to put an upper bound on how long the call will wait. This functionality turned out to be a bit more than was needed, though; in particular, Ts'o [expressed skepticism](<https://lwn.net/Articles/724649/>) about the timeout idea, asking: ""If you are using get_random_bytes() for security reasons, does the security reason go away after 15 seconds?"" The [third version of the patch set](<https://lwn.net/Articles/724671/>) removed all of the arguments to `wait_for_random_bytes()`, making all waits interruptible with no timeout. 

The patch series then adds a set of convenience functions to combine waiting and actually getting the random data, including: 

```
static inline int get_random_bytes_wait(void *buf, int nbytes);
```

Most of the comments on the patch set at this point are about relatively minor issues. So chances are that some version of this patch set will find its way into the kernel eventually, with the result, hopefully, that there will be a reduced chance of kernel code using insufficiently random data. But there is one other aspect of this situation that seems entirely deterministic: the arguments about the quality of the kernel's random-number subsystem are far from finished. That is, after all, the fundamental problem with random numbers: it is difficult to be sure that they are truly random.
