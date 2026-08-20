---
title: Another try for getrandom() in the vDSO
url: https://lwn.net/Articles/980447/
date: "July 4, 2024"
category: "Random numbers; Security-Random number generation"
author: "By Jonathan Corbet July 4, 2024"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
July 4, 2024

Random numbers, it seems, can never be random enough, and they cannot be generated quickly enough. The kernel's [`getrandom()`](<https://man7.org/linux/man-pages/man2/getrandom.2.html>) system call might, after years of discussion, be seen as sufficiently secure by most users, but it is still a system call. Linux system calls are relatively fast, but they are necessarily slower than calling a function directly. In an attempt to speed the provision of secure random data to user space, Jason Donenfeld has put together [an implementation of `getrandom()`](<https://lwn.net/ml/all/20240703183115.1075219-1-Jason@zx2c4.com>) that lives in the [virtual dynamic shared object (vDSO) area](<https://man7.org/linux/man-pages/man7/vdso.7.html>). 

Random data is used in no end of applications, including modeling of natural phenomena, generation of identifiers like [UUIDs](<https://en.wikipedia.org/wiki/Universally_unique_identifier>), and game play; it's how [NetHack](<https://www.nethack.org/>) somehow summons those three balrogs all in one corner of a dark room. Security-related operations, such as the generation of nonces and keys, also make heavy use of random data — and depend heavily on that data actually being random. Some applications need so much random data that they spend a significant amount of time in the `getrandom()` system call. 

One possible solution to this problem is to generate random numbers in user space, perhaps seeded by random data from the kernel; many developers have taken that route over the years. But, as Donenfeld explains in the cover letter to his patch series, that approach is not ideal. The kernel has the best view of the amount of entropy in the system and what is needed to generate truly random data. It is also aware of events, such as virtual-machine forks, that can compromise a random-number generator and make a reseeding necessary. He concluded: 

> The simplest statement you could make is that userspace RNGs that expand a getrandom() seed at some point T1 are nearly always *worse*, in some way, than just calling getrandom() every time a random number is desired. 

Always calling `getrandom()` ensures the best random data, but the associated performance problem remains. Moving that function into the vDSO can help to address that problem. 

#### `getrandom()` in the vDSO

The vDSO is a special mechanism provided to accelerate tasks that require some kernel involvement, but which can otherwise be carried out just as well in user space. It contains code and data provided in user space directly by the kernel in a memory area mapped into every thread's address space. The classic vDSO function is [`gettimeofday()`](<https://man7.org/linux/man-pages/man2/gettimeofday.2.html>), which returns the current system time as kept by the kernel. This function can be implemented as a system call, but that will slow down applications that frequently query the time, of which there are many. So the Linux vDSO includes an implementation of `gettimeofday()`; that implementation can simply read a time variable in memory shared with the kernel and return it to the caller, avoiding the need to make a system call. 

`getrandom()` is a similar sort of function; it reads data from the kernel and returns it to user space. So a vDSO implementation of `getrandom()` might make sense. Such an implementation must be done carefully, though; it should return data that is just as random as a direct call into the kernel would, and it must be robust against the types of events (a fork, for example) that could compromise the state of a thread's random-number generation. 

In Donenfeld's implementation, user-space programs will continue to just call `getrandom()` as usual, with no changes needed. Under the hood, though, there are some significant changes needed within the C library, which provides the `getrandom()` wrapper for the system call. 

#### State-area allocation

The random-number generator works on some state data stored in memory. When random data is needed, a pseudo-random-number generator creates it from that state, mutating the state in the process. Every thread must have its own state, and care must be taken to avoid exposing that state during events like process forks, core dumps, virtual-machine forks, or checkpointing. That state should be reseeded with random data regularly, and specifically at any time when its content might have been compromised. 

The vDSO implementation of `getrandom()` requires that the memory to be used for this state be allocated by the kernel. So the first thing that the C library must do is to allocate this state storage for as many threads as it thinks are likely to run. That is done with a new system call: 

```
struct vgetrandom_alloc_args {
            u64 flags;
          	u64 num;
          	u64 size_per_each;
          	u64 bytes_allocated;
        };
    
        void *vgetrandom_alloc(struct vgetrandom_alloc_args *args, size_t args_len);
```

The structure pointed to by `args` describes the allocation request, while `args_len` is `sizeof(*args)`; that allows the structure to be extended in a compatible way if needed in the future. Within that structure, `flags` must currently be zero, and `num` is the number of thread-state areas that the kernel is being requested to allocate. On a successful return, `num` will be set to the number of areas actually allocated, `size_per_each` describes the size of a state area, and `bytes_allocated` is the total amount of memory that was allocated. The return value will point to the base of the allocated area. 

The allocated area is ordinary anonymous memory, except that it will be specially marked within the kernel using a number of virtual-memory-area flags. The `VM_WIPEONFORK` flag causes its contents to be zeroed if the process forks (so that the two processes do not generate the same stream of random numbers), `VM_DONTDUMP` keeps its contents from being written to core dumps, and `VM_NORESERVE` causes it to not be charged against the process's locked-memory limit. Donenfeld also added a new flag to use with this area: `VM_DROPPABLE` allows the memory-management subsystem to simply reclaim the memory if need be; since this is anonymous memory, accessing it after it is reclaimed will cause a new, zero-filled page to be allocated. The result is memory that should remain private, but which can be zeroed (or reclaimed, which has the same effect) by the kernel at any time. 

#### Generating random data

The kernel also shares some memory with the vDSO containing this structure: 

```
struct vdso_rng_data {
    	u64	generation;
    	u8	is_ready;
        };
```

This structure is used by the vDSO version of `getrandom()`, which has this prototype: 

```
ssize_t vgetrandom(void *buffer, size_t len, unsigned int flags,
                           void *opaque_state, size_t opaque_len);
```

The first three arguments mirror `getrandom()`, describing the amount of random data needed and whether the call should block waiting for the kernel's random-number generator to be ready. The final two, instead, describe one of the state areas allocated by `vgetrandom_alloc()`. This function's job is to provide the same behavior that `getrandom()` would. 

It starts by looking at the `is_ready` field in the shared structure; if the kernel's random-number generator is not yet ready, `vgetrandom()` will just call `getrandom()` to handle the request. Once the random-number generator has initialized, though, that fallback will no longer be necessary. So the next thing to do is to compare the `generation` count (which tracks the number of times that the kernel's random-number generator has been reseeded) with a generation count stored in the state area. If the two don't match, then the state area must be reseeded with random data obtained from the kernel. 

When the state area is first allocated, it is zeroed, so the generation number found there will be zero, which will never match the kernel's generation number; that will cause the state area to be seeded on the first call to `vgetrandom()`. The same thing will happen if this area has been cleared by the kernel, as the result of a fork (`VM_WIPEONFORK`) or the memory being reclaimed (`VM_DROPPABLE`), for example. So the kernel is able to clear that memory at any time in the knowledge that `vgetrandom()` will do the right thing. 

Once the state area is known to be in a good condition, `vgetrandom()` uses it to generate the requested random data using the same algorithm used within the kernel itself. Doing this calculation securely is a bit tricky; if the process forks or core-dumps while it is underway, any data kept on the stack could be exposed. So `vgetrandom()` has to use an implementation of the [ChaCha20 stream cipher](<https://datatracker.ietf.org/doc/html/rfc7539>) that uses no stack at all. The patch series only includes an x86 implementation of this cipher; other architectures seem certain to follow. 

As a final step before returning the generated data to the caller, `vgetrandom()` checks the generation number one more time. If, for example, the state area was wiped by the kernel while the call was executing, the generation-number check will fail. In such cases, `vgetrandom()` will throw away its work and start over. 

Donenfeld described the end result of this work as ""pretty stellar (around 15x for uint32_t generation)"" and noted happily that ""it seems to be working"". 

#### Prospects

LWN last [looked at this work](<https://lwn.net/Articles/919008/>) at the beginning of 2023. At that time, there were a number of objections, many of which were focused on the `VM_DROPPABLE` changes to the memory-management subsystem, which included some tricky, x86-specific tricks. When [version 15](<https://lwn.net/ml/linux-kernel/20240521111958.2384173-1-Jason@zx2c4.com/>) of the patch series was posted several months later, `VM_DROPPABLE` remained, but the logic had been simplified considerably in the hope of addressing those concerns, seemingly successfully. There does not appear to be anybody who is arguing against the inclusion of this series now. 

As of the current version (20), this work has been added to linux-next for wider testing; if all goes well, it could go upstream as soon as the 6.11 merge window later this month. "If all goes well", of course, includes passing muster with Linus Torvalds, who has not commented this time around; he was [not thrilled](<https://lwn.net/ml/linux-kernel/CAHk-=wg_6Uhkjy12Vq_hN6rQqGRP2nE15rkgiAo6Qay5aOeigg@mail.gmail.com/>) with previous versions, though. Should the mainline merge happen, the work to integrate the needed changes into the C libraries can begin. The end result will be a significant internal change, but the only thing that users should notice is that their programs run faster.
