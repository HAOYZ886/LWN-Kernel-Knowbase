---
title: A vDSO implementation of getrandom()
url: https://lwn.net/Articles/919008/
date: "January 6, 2023"
category: "Random numbers; Security-Random number generation"
author: "By Jonathan Corbet January 6, 2023"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
January 6, 2023

Most developers probably do not see the generation of random numbers as being a performance bottleneck for their programs, but there are seemingly exceptions. Over the last few years, Jason Donenfeld has brought a new level of energy to the development of the kernel's random-number generator; he is now directing his efforts toward improving performance for user space with [this patch series](<https://lwn.net/ml/linux-kernel/20230101162910.710293-1-Jason@zx2c4.com/>) that provides an implementation of the [`getrandom()`](<https://man7.org/linux/man-pages/man2/getrandom.2.html>) system call in the kernel's ["virtual dynamic shared object" (vDSO)](<https://lwn.net/Articles/615809/>) area. The result is, indeed, better performance, but not all developers see this benefit as being worth the additional complexity required to achieve it. 

Traditionally, user-space processes on Linux systems have obtained random data by opening `/dev/urandom` (or `/dev/random`) and reading data from it. More recently, [the addition of `getrandom()`](<https://lwn.net/Articles/606141/>) simplified access to random data; a call to `getrandom()` will fill a user-space buffer with random data from the kernel without the need to open any files. This random data is provided with all of the guarantees that the kernel can make, including doing its best to ensure that the data is actually random and [preventing repeated data sequences](<https://lwn.net/Articles/887207/>) when, for example, a virtual machine forks. 

It's worth noting that, in the BSD world, it is more common to call the [`arc4random()`](<https://man.openbsd.org/arc4random.3>) library function. The [2.36 release](<https://lwn.net/Articles/903556/>) of the GNU C Library included an implementation of `arc4random()` that, in its pre-release form, included a fair amount of its own logic for the generation and management of random data. In July 2022, Donenfeld [questioned the need](<https://lwn.net/ml/libc-alpha/Ytwg8YEJn+76h5g9@zx2c4.com/>) for this function, noting that ""getrandom() and /dev/urandom are extremely fast"". Supporting `arc4random()` makes code more portable, though, so that function stayed in the library. The version that was eventually released [was significantly simplified by Donenfeld](<https://lwn.net/ml/libc-alpha/20220726195822.1223048-1-Jason@zx2c4.com/>), to the point that it essentially a wrapper around `getrandom()` when that system call is available. As a result, the performance of `getrandom()` also determines how fast `arc4random()` will be. 

#### The vDSO API

As ""extremely fast"" as `getrandom()` may be, some users have apparently complained that it still is not fast enough. In response, Donenfeld is now working to create a vDSO implementation. The vDSO is a special range of kernel memory that is mapped into each user-space process; it contains code that can be called to perform system-call-like functions that can be carried out without actually entering the kernel, thus avoiding the cost of a context switch. System calls implemented in this way can include [`getcpu()`](<https://man7.org/linux/man-pages/man2/getcpu.2.html>) and [`clock_gettime()`](<https://man7.org/linux/man-pages/man3/clock_gettime.3.html>) — both of which come down to simply reading some data out of the kernel. 

Moving `getrandom()` into the vDSO has the potential to improve performance for applications that need a lot of random data, but it will clearly need to involve more than just reading some kernel data if the guarantees made by that system call are to be upheld. As a result, the patch series implementing this functionality touches 59 files and adds some 1,200 lines of code. It also complicates the interface somewhat, in that there are now two (virtual) system calls involved. The first must be called at least once prior to requesting any random data: 

```
void *vgetrandom_alloc(unsigned int *num, unsigned int *size_per_each,
                               unsigned long addr, unsigned int flags);
```

The caller (likely to be the C library) should specify, in `*num`, the number of "opaque states" needed; that number is likely to correspond to the number of threads that are expected to run. This call will allocate a range of memory, returning its address. The number of states actually allocated will be stored in `*num`, and the size of each state in `*size_per_each`. The other two arguments are currently unused and must be zero. 

The library is expected to assign one of the returned states to each thread in a process; the pointer to any given state can be had by offsetting the base address by a multiple of the value returned in `*size_per_each`. The kernel uses this space to track the state of the random-number generator for the thread; user space should not access its contents. Should more state-storage space be needed, more calls can be made to `vgetrandom_alloc()`. 

With the state space in hand, random data can be obtained with: 

```
ssize_t vgetrandom(void *buffer, size_t len, unsigned int flags,
        		       void *opaque_state);
```

The first three arguments are the same as for `getrandom()`, while `opaque_state` points to one of the states returned by `vgetrandom_alloc()`. The intent is for this function to behave just as `getrandom()` would, only faster. Applications would not normally call it directly; instead, it would be invoked from the C library's `getrandom()` wrapper. Donenfeld said (in the cover letter) that ""performance is pretty stellar (around 15x for uint32_t generation), and it seems to be working"". 

#### `VM_DROPPABLE`

While some developers are clearly unconvinced about the need to optimize `getrandom()`, most of the patch series is relatively uncontroversial at this point. There is one significant exception, though: the management of the "opaque state" data. This data is used by the kernel to ensure that each caller to `vgetrandom()` gets a unique stream of random data, that each thread's random-number generator is reseeded as needed, and so on; it can be thought of as a sort of per-thread entropy storage that can be consulted without entering the kernel. For the curious, each state entry is defined (in the kernel) as: 

```
struct vgetrandom_state {
    	union {
    	    struct {
    		u8	batch[CHACHA_BLOCK_SIZE * 3 / 2];
    		u32	key[CHACHA_KEY_SIZE / sizeof(u32)];
    	    };
    	    u8		batch_key[CHACHA_BLOCK_SIZE * 2];
            };
        	vdso_kernel_ulong	generation;
        	u8			pos;
        	bool 			in_use;
        };
```

This structure occupies 256 bytes, which is not huge, but a system running a lot of threads could have quite a few of them. This is kernel-allocated memory that, in effect, must be locked into RAM; writing it out to secondary storage could expose the state of the random-number generator, potentially creating security problems. The state memory must thus be treated specially. 

Earlier versions of the patch set used [`mlock()`](<https://man7.org/linux/man-pages/man2/mlock.2.html>) to ensure that this memory would stay in place. There were some problems with that approach, though, starting with the fact that locked memory is subject to resource limits. If the kernel and C library start using some of a process's allowed locked memory for random-number generation, once-working programs could start failing. Locked memory will also become unlocked in the child process after a fork. So using `mlock()` is not an ideal solution. 

The memory used for states has another interesting characteristic, though: it can simply vanish at any time with minimal consequences. Since it is, at its core, a cache of random data, it can be replaced with more random data. This can already happen if, for example, the thread forks. In the worst case, should the state memory not be available at all, the vDSO function can just make a proper call to `getrandom()` to get its job done (albeit more slowly). So, while the state memory should be locked in the sense of not being written to swap, it does not need to be absolutely nailed down in RAM; it can be thrown away if need be. 

Donenfeld decided to take advantage of the disposable nature of this memory. Rather than using `mlock()`, the current patch set adds a new (kernel-internal) memory flag called `VM_DROPPABLE`. Memory marked with this flag will never be written to swap or a core dump, and the memory-management subsystem can, if memory is tight, just reclaim it for other uses. `VM_DROPPABLE` memory is demand-paged, in that it is not actually allocated until an attempt is made to access it; normally the failure to allocate a page in this circumstance will be fatal for the process involved. With `VM_DROPPABLE` memory, instead, failures are simply ignored and any attempted writes are simply dropped — an outcome that is effected with some low-level, architecture-specific magic that simply skips the instruction attempting the write. 

#### Dropping `VM_DROPPABLE`

Ingo Molnar [objected strongly](<https://lwn.net/ml/linux-kernel/Y7QIg%2FhAIk7eZE42@gmail.com/>) to this patch, saying that it adds complexity to a subsystem that needs a high level of trust. It would be better, he said, to just make `mlock()` work as needed or simply let the vDSO allocate a few pages outside of the existing resource limits. Donenfeld [reacted poorly](<https://lwn.net/ml/linux-kernel/Y7RDQLEvlLM0o4cp@zx2c4.com/>), questioning Molnar's motives. Molnar then [vetoed the patch](<https://lwn.net/ml/linux-kernel/Y7Rw1plb%2FpqPiWgg@gmail.com/>), leading to more [complaints](<https://lwn.net/ml/linux-kernel/Y7R5OokY7P+H2vuD@zx2c4.com/>) from Donenfeld, who did point out, though, that a process can make an arbitrary number of calls to `vgetrandom_alloc()`, so care needs to be taken to prevent it from being used to circumvent the limits on locked memory. Simply tweaking `mlock()` would not be a sufficient solution. 

Andy Lutomirski also [disliked `VM_DROPPABLE`](<https://lwn.net/ml/linux-kernel/CALCETrWdw5kxrtr4M7AkKYDOJEE1cU1wENWgmgOxn0rEJz4y3w@mail.gmail.com/>), saying that Donenfeld was ""trying to shoehorn all kinds of bizarre behavior into the core mm"". He suggested that Donenfeld could, instead, create a special type of mapping, with its own local implementation, to obtain the needed semantics; the kernel provides the infrastructure needed to do this. There would then be no need to modify the core memory-management subsystem. Donenfeld had [a relatively positive reaction to this suggestion](<https://lwn.net/ml/linux-kernel/Y7R8Zq6sIKAIprtr@zx2c4.com/>) and seemed ready to proceed in that direction. Linus Torvalds, though, [argued](<https://lwn.net/ml/linux-kernel/CAHk-=wg_6Uhkjy12Vq_hN6rQqGRP2nE15rkgiAo6Qay5aOeigg@mail.gmail.com/>) that none of this effort was worth it. Speeding up random-number generation, beyond a point, is not the kernel's job, he said; instead, that should be left to the C library. After some discussion, Torvalds [made his position clear](<https://lwn.net/ml/linux-kernel/CAHk-=wi4gshfKjbhEO_xZdVb9ztXf0iuv5kKhxtvAHf2HzTmng@mail.gmail.com/>): 

> I'm NAK'ing making invasive changes to the VM for something this specialized. I really believe that the people who have this issue are *so* few and far between that they can deal with the VM forking and reseeding issues quite well on their own. 

Donenfeld has also been clear in his disagreement with Torvalds's assessment of the need for this feature, though; he [claimed](<https://lwn.net/ml/linux-kernel/Y7SDgtXayQCy6xT6@zx2c4.com/>) that it just isn't possible to create a safe random-number generator in user space (the patch cover letter also covers that ground in depth). He later [went on to say](<https://lwn.net/ml/linux-kernel/Y7dV1lVUYjqs8fh0@zx2c4.com/>): 

> I think the moral of yesterday's poo-poo'ing all over this cool new idea is that the Linux innercircle doesn't really care for "security things" as a motivator and just takes the shortest and easiest route toward swatting it away like a gadfly, assuming that the concerns are unreal or niche or paranoid or whatever. 

It will probably come as little surprise that Torvalds [disagreed with this view](<https://lwn.net/ml/linux-kernel/CAHk-=wijEC_oDzfUajhmp=ZVnzMTXgjxHEcxAfaHiNQm4iAcqA@mail.gmail.com/>). 

In the end, though, there do appear to be valid arguments, from both performance and security points of view, for a vDSO `getrandom()` implementation. So work on the vDSO patches is likely to continue, but the `VM_DROPPABLE` flag is clearly not going to clear the bar. Donenfeld will almost certainly return with an implementation based on Lutomirski's suggestions; that should address the main concerns that have been raised about this work. At that point, its chances of getting upstream are probably reasonably good, even if some developers remain unconvinced about how necessary it really is.
