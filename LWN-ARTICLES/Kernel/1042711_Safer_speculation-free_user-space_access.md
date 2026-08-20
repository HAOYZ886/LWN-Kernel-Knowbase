---
title: "Safer speculation-free user-space access"
url: https://lwn.net/Articles/1042711/
date: "October 23, 2025"
category: "Releases-6.19; Scoped resource management; Security-Meltdown and Spectre"
author: "By Jonathan Corbet October 23, 2025"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
October 23, 2025

The Spectre class of hardware vulnerabilities truly is a gift that keeps on giving. New variants are still being discovered in current CPUs nearly eight years after the [disclosure](<https://lwn.net/Articles/742702/>) of this problem, and developers are still working to minimize the performance costs that come from defending against it. The masked user-space access mechanism is a case in point: it reduces the cost of defending against some speculative attacks, but it brought some challenges of its own that are only now being addressed. 

The Spectre vulnerabilities can be used to exfiltrate data from the kernel in a number of ways, but the attacks usually come down to exercising a kernel path that will speculatively execute with an attacker-provided address, leaving traces of the target data that can then be recovered via a side channel. One of the most common ways to defeat such attacks is to simply prevent speculative execution of some code; it is effective, but also expensive. 

#### Defending user-space access

One common target for speculative attacks is accesses to user space by the kernel, since the address in question is often controlled by user space. Since the tests for the validity of an address nearly always succeed, speculative execution tends to take the "address is valid" path, even when the address is anything but. The functions used by most of the kernel for user-space access (such as [`copy_from_user()`](<https://elixir.bootlin.com/linux/v6.17.3/source/include/linux/uaccess.h#L204>)) are well defended, but the kernel has a number of places where faster access is required for acceptable performance. This can especially be a concern when multiple accesses to user space are required. Code in such situations tends to use a pattern like [this one](<https://elixir.bootlin.com/linux/v6.10.14/source/fs/select.c#L778>) from the 6.10 implementation of the [`select()`](<https://man7.org/linux/man-pages/man2/select.2.html>) system call, which only incurs the cost for the speculation defense once but performs two reads: 

```
if (from) {
                if (!user_read_access_begin(from, sizeof(*from)))
                    return -EFAULT;
                unsafe_get_user(to->p, &from->p, Efault);
                unsafe_get_user(to->size, &from->size, Efault);
                user_read_access_end();
            }
            return 0;
        Efault:
            user_access_end();
            return -EFAULT;
```

The `user_read_access_begin()` call is implemented as a chain of macros before finally doing two things: enabling user-space access with a `STAC` instruction, and blocking speculation with an `LFENCE` instruction. The `unsafe_get_user()` macros, which include a jump to `Efault` on error, can then be used to access the relevant data. Finally, `user_read_access_end()` and `user_access_end()` both boil down to a `CLAC` instruction to re-enable [supervisor mode access prevention](<https://lwn.net/Articles/517475/>); an important step that, if forgotten, can leave the kernel open to other attacks. The `STAC`/`CLAC` pair is unavoidable, but it would be nice to do away with the costly `LFENCE` if possible. 

#### Defense without fences

The first commit in the 6.11 merge window was [this change](<https://git.kernel.org/linus/2865baf54077>) from Linus Torvalds adding a new mechanism that he called "user address masking". It uses a relatively simple trick to avoid the `LFENCE` instruction, ensuring that any attempt at kernel-space access with a supposedly user-space address will fail. There were two new macros: 

```
#define mask_user_address(x) ((typeof(x))((long)(x)|((long)(x)>>63)))
        #define masked_user_access_begin(x) ({ __uaccess_begin(); mask_user_address(x); })
```

Passing a pointer to `mask_user_address()` will perform a logical OR of the address with a version of itself right-shifted by 63 bits. The sign-extension performed by the x86 CPU means that, if the address is in kernel space (the topmost bit is one), the resulting address will be all ones, which is not valid. Any speculation involving a kernel-space address will, as a result, fail on the invalid access. Since exploitable speculation can no longer happen, there is no longer any need for the `LFENCE` instruction. 

(For the curious, the implementation of these macros [was changed](<https://git.kernel.org/linus/91309a70829d>) in 6.14, making them [quite different](<https://elixir.bootlin.com/linux/v6.17.3/source/arch/x86/include/asm/uaccess_64.h#L59>) from the original in current kernels; amusingly, they no longer involve masking. The end result is the same, though, and the "masked access" term is still used.) 

Masked access can accelerate performance-sensitive operations, but it has a small disadvantage: it is not supported by all architectures. So code that uses this feature must be prepared to fall back to the previous method on architectures where masked access is not available. The `select()` code shown above is, as a result, [in 6.17](<https://elixir.bootlin.com/linux/v6.17.3/source/fs/select.c#L774>), written as: 

```
if (from) {
                if (can_do_masked_user_access())
                    from = masked_user_access_begin(from);
                else if (!user_read_access_begin(from, sizeof(*from)))
                    return -EFAULT;
                unsafe_get_user(to->p, &from->p, Efault);
                unsafe_get_user(to->size, &from->size, Efault);
                user_read_access_end();
            }
        Efault:
            user_access_end();
            return -EFAULT;
```

The code is faster, but has also become more complex. 

#### Using scopes

As Thomas Gleixner pointed out in [this patch series](<https://lwn.net/ml/all/20250916163004.674341701@linutronix.de>), all that code to read two user-space values is just the sort of ""tedious"" boilerplate that offers numerous opportunities for security-critical mistakes. As the use of the masked-access primitives grows over time, the chances of introducing new bugs will grow as well. He set out to improve this pattern using the kernel's [scoped primitives](<https://lwn.net/Articles/934679/>) to ensure that the proper cleanup is done once the access is complete. The result in [the current version](<https://lwn.net/ml/all/20251022102427.400699796@linutronix.de>) of the series is three new macros: 

```
scoped_user_read_access(address, label)
        scoped_user_write_access(address, label)
        scoped_user_rw_access(address, label)
```

Each of these starts a new block and speculation-proofs the given `address`, inserting a jump to the specified `label` in the case of an access violation. Using these macros, the `select()` code can now look like: 

```
if (from) {
                scoped_user_read_access(from, Efault) {
                    unsafe_get_user(to->p, &from->p, Efault);
                    unsafe_get_user(to->size, &from->size, Efault);
                }
            }
        Efault:
            return -EFAULT;
```

The end result is clearly simpler and less prone to the sorts of mistakes that developers are likely to make. The need for explicit cleanup code, in particular, has been completely removed. 

This work is in its third revision; aside from some relatively minor comments, it would appear to have reached general approval. It seems to be a likely candidate for the 6.19 merge window. This work may affect a relatively obscure corner of the kernel that few developers will see directly, but it is a good example of the ongoing effort to make kernel development a bit less error prone. Moving away from C is not in the cards for a long time, so the next best thing is to make working in C safer.
