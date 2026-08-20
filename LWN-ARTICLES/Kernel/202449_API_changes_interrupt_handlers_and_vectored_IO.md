---
title: "API changes: interrupt handlers and vectored I/O"
url: https://lwn.net/Articles/202449/
date: "October 2, 2006"
category: "Asynchronous IO; Device drivers-Support APIs; Interrupts; Vectored IO"
author: 
---

Normally, the release of 2.6.19-rc1 would be the signal that the release cycle process would begin to slow down and focus on bug fixes. Things might be just a little different this time around, however, as a large and disruptive (almost 1100 files changed) API change is likely to go in between -rc1 and -rc2. The reasoning is this: a patch which hits so many files will inevitably conflict with a number of the other patches currently flooding into the mainline. Holding this patch until the flood should make life easier all around. 

So what is this patch? Consider that interrupt handlers currently have the following prototype: 

```
irqreturn_t handler(int irq, void *data, struct pt_regs *regs);
```

The `regs` structure holds the state of the processor's registers at the time of the interrupt. It is passed to every interrupt handler, but it is almost never used; for the purposes of most handlers, the pre-interrupt register state is just a bunch of random bits. There is a cost to passing this pointer around, however. According to David Howells: 

The regs pointer is used in few places, but it potentially costs both stack space and code to pass it around. On the FRV arch, removing the regs parameter from all the genirq function results in a 20% speed up of the IRQ exit path (ie: from leaving timer_interrupt() to leaving do_IRQ()). 

So David has put together [a patch](<http://userweb.kernel.org/~akpm/irq-maintain-regs-pointer-globally-rather-than-passing-to-irq-handlers.patch>) which removes the `regs` argument to interrupt handlers. Any code which actually needs the registers - seemingly only the timer interrupt handler - can get the pointer with a call to the new `get_irq_regs()` function. Since this change obviously requires fixing every interrupt handler in the system - and there are a lot of them in the mainline kernel - the patch is large and touches a lot of files. 

This patch has just now come along, meaning that, by normal standards, it is a bit late for the 2.6.19 party. So it would normally sit in -mm for this cycle, and be merged into 2.6.20. But, [Andrew Morton says](<https://lwn.net/Articles/202453/>): 

I think the change is good. But I don't want to maintain this whopper out-of-tree for two months! If we want to do this, we should just smash it in and grit our teeth 

Nobody else seems to object to the change, though Linus did [spare a moment](<https://lwn.net/Articles/202455/>) to feel the pain of people maintaining drivers out of the mainline tree. The writing on the wall all points to a near-term inclusion, perhaps with a special defined symbol to help out-of-tree maintainers write code which works with both handler prototypes. 

Meanwhile, the `file_operations` structure can be found at the core of just about any subsystem which does I/O. Char device drivers create `file_operations` structures directly, while most other parts of the system (filesystems, network protocols and drivers, block drivers) bury them in higher-level logic. Two of the members of this structure are: 

```
ssize_t (*aio_read) (struct kiocb *iocb, char __user *buf, 
                             size_t len, loff_t pos);
        ssize_t (*aio_write) (struct kiocb *iocb, const char __user *buf, 
                              size_t len, loff_t pos);
```

These methods implement asynchronous reads and writes - operations which may be completed sometime after the original call returns to user space. One longstanding shortcoming of the Linux asynchronous I/O implementation is its lack of vectored operations; each AIO call can only operate on a single buffer. The 2.6.19 kernel will fill in that gap, at the cost of changing the above two prototypes to: 

```
ssize_t (*aio_read) (struct kiocb *iocb, const struct iovec *iov, 
                 unsigned long niov, loff_t pos);
        ssize_t (*aio_write) (struct kiocb *iocb, const struct iovec *iov, 
                 unsigned long niov, loff_t pos);
```

The single buffer has been replaced by an array of `iovec` structures: 

```
struct iovec
        {
    	void __user *iov_base;
    	__kernel_size_t iov_len;
        };
```

Single-buffer calls are now wrapped in a single `iovec` structure and passed to the new, vectorized versions of the AIO operations. All code which provides `aio_read()` and `aio_write()` will need to be updated to the new API - and the possibility of being requested to perform vectored operations. 

The changes actually go beyond that, however, in that the `readv()` and `writev()` `file_operations` methods have been removed. The associated system calls are now, instead, implemented with calls to `aio_read()` and `aio_write()`. Converting older `readv()` and `writev()` methods is not particularly difficult, since there is no requirement that `aio_read()` and `aio_write()` must be asynchronous (in fact, in this case, they will be passed a "synchronous KIOCB" which indicates that the operation must be performed synchronously). In most cases, it is simply a matter of adopting the new prototype, then looking in `iocb->ki_filp` for the `struct file` pointer, should it be needed. 

(See [this article from last February](<http://lwn.net/Articles/170954/>) for more background on this change).
