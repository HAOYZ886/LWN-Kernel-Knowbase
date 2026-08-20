---
title: "A new API for kfifo?"
url: https://lwn.net/Articles/347619/
date: "August 19, 2009"
category: kfifo
author: "By Jake Edge August 19, 2009"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jake Edge**  
August 19, 2009

The kernel FIFO implementation, `kfifo`, is not that widely used and Stefani Seibold would like to see that change. To that end, she has [proposed](<http://lwn.net/Articles/347346/>) a new `kfifo` API that embodies many of the [design patterns for data structures](<http://lwn.net/Articles/336255/>) that were described by Neil Brown. The new interface is simpler in many ways, so it should be easier to use, which, in turn, could lead to more use throughout the kernel. 

The basic problems with the current `kfifo` interface stem from the constraints it places on its users. A spinlock is required, though it is not needed by the majority of current `kfifo` users. Also, the `kfifo` structure can only be allocated dynamically, so users cannot put FIFOs inside of other structures--only pointers to FIFOs. Moreover, according to Seibold, the current API is too simple and doesn't provide a number of important features. 

The new API has 23 separate functions, while the current has 9, but there are quite a few variants that increase the total. Those variants include copying from or to user space, supporting fixed-length records, as well as being able to use spinlocks to synchronize adding or removing data from the FIFO. It supports many different use cases in the style of Brown's "Broad Interfaces" pattern. 

A `kfifo` is declared using the `DECLARE_KFIFO()` macro which can be used inside of a struct or union declaration. FIFOs declared with with `DECLARE_KFIFO()` must be initialized using `INIT_KFIFO()`. Alternatively, `DEFINE_KFIFO()` handles both the declaration and initialization for FIFOs outside of struct/union declarations. The macros take `name` (to name the variable or struct/union element) and `size` (in bytes) parameters: 

```
DECLARE_KFIFO(name, size)
        INIT_KFIFO(name)
    
        DEFINE_KFIFO(name, size)
```

There are two functions to support dynamic buffer allocation: 

```
int kfifo_alloc(struct kfifo *fifo, unsigned int size, gfp_t gfp_mask)
```

which allocates a buffer of `size` bytes using the `gfp_mask` as flags to pass to `kmalloc()`. Or, a pre-allocated buffer can be attached to a `kfifo` using: 

```
void kfifo_init(struct kfifo *fifo, unsigned char *buffer, unsigned int size)
```

Any buffer allocated by `kfifo_alloc()` should later be freed by calling `kfifo_free()`. 

To put data into the FIFO, `kfifo_in()` is used: 

```
unsigned int kfifo_in(struct kfifo *fifo, 
            unsigned char *from, unsigned int n)
```

which returns the number of bytes stored. As mentioned above there are variants for getting the data from user space, as well as for locking: 

```
unsigned int kfifo_from_user(struct kfifo *fifo, 
            const void __user *from, unsigned int n)
    
        unsigned int kfifo_in_locked(struct kfifo *fifo,
    	const unsigned char *from, unsigned int n, spinlock_t *lock)
```

As one might guess, the calls to remove data from the FIFO look similar--if reversed: 

```
unsigned int kfifo_out(struct kfifo *fifo, 
            unsigned char *to, unsigned int n)
    
        unsigned int kfifo_to_user(struct kfifo *fifo, 
            void __user *to, unsigned int n)
    
        unsigned int kfifo_out_locked(struct kfifo *fifo,
    	unsigned char *to, unsigned int n, spinlock_t *lock)
```

In the common case, with only one reader and one writer, extra locks are not required to add or remove data from the FIFO, but for more complicated scenarios, the `*_locked()` variants allow the caller to use the appropriate lock. 

The expected tests for FIFO full and empty (`kfifo_is_full()` and `kfifo_is_empty()`) are present, as are ways to get FIFO size and available space (`kfifo_size()`, `kfifo_len()`, and `kfifo_avail()`). One can also consume some FIFO bytes without returning them using `kfifo_skip()` or clear the entire FIFO with `kfifo_reset()`. 

There is also support for fixed-length records, with either 1- or 2-byte lengths stored with each entry. Each call must pass a `recsize` parameter that specifies the size of the length field (i.e. 1 or 2, though 0 is supported for no length) stored with each entry: 

```
unsigned int kfifo_in_rec(struct kfifo *fifo,
    	void *from, unsigned int n, unsigned int recsize)
    
        unsigned int kfifo_from_user_rec(struct kfifo *fifo,
    	const void __user *from, unsigned int n, unsigned int recsize)
    
        unsigned int kfifo_out_rec(struct kfifo *fifo,
    	void *to, unsigned int n, unsigned int recsize,
    	unsigned int *total)
    
        unsigned int kfifo_to_user_rec(struct kfifo *fifo,
    	void __user *to, unsigned int n, unsigned int recsize,
    	unsigned int *total)
```

These functions return the number of bytes _not_ copied, which is a bit strange. For the functions that remove data from the FIFO, `*total` stores the number of bytes actually copied. This part of the interface seems a little dubious, and may not survive in its present form, though no comments along those lines have been seen. 

Overall, the interface has been fairly well-received. There were a few comments on an [earlier version](<http://lwn.net/Articles/347168/>) of the API, which Seibold has mostly addressed. The only comments on the most recent version (v0.4) were a disagreement between Alan Cox and Andrew Morton over naming conventions. 

Cox would rather not see the current `kfifo_put()` and `kfifo_get()` calls get removed quite yet, [noting](<https://lwn.net/Articles/347724/>) ""We are about to set fifo loose through all the USB and some other char/serial drivers all of which will use the spinlock facility."" The current calls use the spinlock, so the kind of change Seibold is proposing would need to be reflected in the USB and char/serial driver code. But Morton [thinks](<https://lwn.net/Articles/347725/>) that this is the right time to make those changes, because ""kfifo has no business assuming that the caller wants to use spin_lock() locking"". 

The basic problem is that Morton would like to see a convention followed that things like `kfifo_put()` (or, `kfifo_in()` as Seibold's API defines it) do not assume locking. He agrees with Seibold's decision to add a separate `kfifo_*_locked()` variants. Cox [points out](<https://lwn.net/Articles/347731/>) that the convention is very inconsistently followed, but Morton is [adamant](<https://lwn.net/Articles/347732/>): 

Plus, as I've said enty en times and keep getting ignored: the current naming is wrong. The generic kfifo_get() should not be assuming that the caller wants spinlock-based locking. 

After initially NAK-ing part of the patch, Cox seems to have [relented](<https://lwn.net/Articles/347733/>), so that particular barrier is gone. It would seem far too late in the 2.6.31 process for this kind of change to go in, but unless some other major issues crop up, it is quite possible that a new `kfifo` API could show up in 2.6.32.
