---
title: Shrinking the kernel with an axe
url: https://lwn.net/Articles/746780/
date: "February 8, 2018"
category: Embedded systems
author: "February 8, 2018 This article was contributed by Nicolas Pitre"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

February 8, 2018

This article was contributed by Nicolas Pitre

This is the third article of a series discussing various methods of reducing the size of the Linux kernel to make it suitable for small environments. The [first article](<https://lwn.net/Articles/741494/>) provided a short rationale for this topic, and covered link-time garbage collection. The [second article](<https://lwn.net/Articles/744507/>) covered link-time optimization (LTO) and compared its results to link-time garbage collection. In this article we'll explore ways to make LTO more effective at optimizing kernel code away, as well as more assertive strategies to achieve our goal. 

#### Kernel module carving

Many kernel drivers start by allocating memory and registering stuff; then they sit there waiting until something they're responsible for happens, if ever. So the first course of action to reduce our kernel size is to make things into modules. This way some code can sit on the filesystem until it is actually needed. 

Let's pick up the same kernel config and build environment as the one from the previous article, and reduce it by carving out some modules. We start with: 

```
$ make stm32_defconfig
        $ make vmlinux
        $ size vmlinux
           text    data     bss     dec     hex filename
        1704024  144732  117660 1966416  1e0150 vmlinux
```

Now let's enable module support and turn a couple things into modules: 

```
$ ./scripts/config --enable MODULES
        $ ./scripts/config --module INPUT
        $ ./scripts/config --module I2C
        $ ./scripts/config --module NLS
        $ ./scripts/config --module CRYPTO
        $ make olddefconfig
        $ make && size vmlinux
           text    data     bss     dec     hex filename
        1738586  137296  109180 1985062  1e4a26 vmlinux
```

This is bad. Despite the `.data` and `.bss` sections becoming 7,436 and 8,480 bytes smaller, respectively, the `.text` segment is now 34,562 bytes larger despite a couple drivers being removed from the core kernel. Is the module support code that big? 

```
$ size kernel/module.o
         text    data     bss     dec     hex filename
        12452     248      77   12777    31e9 kernel/module.o
```

So module support itself is only 12,452 bytes of text. Extra code for module support is expected, however there are still 22,110 additional bytes that appeared from somewhere. Could LTO help here? 

```
$ ./scripts/config --enable LTO_MENU
        $ make && size vmlinux
           text    data     bss     dec     hex filename
        1653358  137356  104648 1895362  1cebc2 vmlinux
```

LTO improved things, but produced only a 4.5% size reduction. We're far from the 22% reduction we obtained previously when modules were disabled. Why is module support so counter-productive? 

#### The tree that hides the forest

We've seen that LTO is very good at optimizing code away. It is especially good at figuring out that some functions end up never being called; their removal means that even more functions end up not being called, and so on along the call graph down to the leaf functions. It is like cutting a limb from a tree; every sub-branch and leaf on it obviously won't be connected to the tree anymore and will fall to the ground. But what LTO does is more like getting rid of branches that simply float in the air without being connected to anything or which have become loose due to optimization. Branches connected to the trunk won't be trimmed. Neither will the trunk itself (the `main()` function) for obvious reasons. 

The kernel, unlike user-space programs that typically have only one entry point, is different as it has multiple entry points. In fact it has so many entry points that we may compare it to a forest rather than a tree, with interlacing (interdependent) branches forming a dense canopy. No wonder why it is so hard to obtain a _light_ kernel. 

Among those kernel entry points we have: 

  * The `start_kernel()` function which is equivalent to the `main()` function in a user space program. 
  * Every system call (about 400 of them) is a kernel entry point. Obviously some of them must be present for the kernel to be useful. 
  * Every `EXPORT_SYMBOL()` statement is declaring an entry point to the kernel. Some of them designate data rather than code, but they create a dependency link just the same. 
  * Every `initcall()` instance, of which there are multiple variants, is yet another entry point of some sort, even though the caller remains within the kernel itself. Still, they add call dependencies of their own. 
  * And to a lesser extent in terms of code size, there are all those parameter parsing functions attached to `early_param()` statements. 

Normally, the `initcall()` and `early_param()` instantiated code is marked with the `__init` qualifier and therefore evicted from memory once the boot process is complete. However, `EXPORT_SYMBOL()` really is a problem. Just by turning `CONFIG_MODULES` on, we added a _lot_ of tree trunks to our kernel: 

```
$ wc -l Module.symvers
        3984 Module.symvers
        $ wc -l modules.order
        30 modules.order
```

In plain text, that means that there are 3,984 exported symbols in this kernel configuration for 30 configured modules. That means 3,984 additional entry points that LTO can no longer optimize away. It is unlikely that our modules need that many exported symbols. Let's find that out: 

```
$ find . -name \*.ko -exec nm {} \; |
        >      grep "^ *U" | sort | uniq | wc -l
        429
```

So 429 symbols are needed for our set of modules, including those symbols that are exported from other modules, meaning that in reality we'd need less than 429 exported symbols from the kernel core. 

This is where the [trimming of unused exported kernel symbols](<https://lwn.net/Articles/679934/>) becomes especially useful. This is activated with `CONFIG_TRIM_UNUSED_KSYMS`, available [since Linux v4.7](<https://git.kernel.org/torvalds/c/dbacb0ef670d057a>). It works by gathering all required symbols from the set of configured modules (similarly to what is done above), and storing them in `include/generated/autoksyms.h` as a list of defines. Those defines control whether their corresponding `EXPORT_SYMBOL()` instance is activated or not. If `adjust_autoksyms.sh` detects that the list of symbols doesn't match the existing list, then it updates that list and triggers a rebuild of the affected source files. The process is repeated until the list of exported symbols becomes stable. Given `autoksyms.h` is initially empty, at least one rebuild loop is needed. 

Let's see what we get from this: 

```
$ ./scripts/config --enable TRIM_UNUSED_KSYMS
        $ make
        $ wc -l Module.symvers
        429
        $ size vmlinux
           text    data     bss     dec     hex filename
        1546209  137292  109172 1792673  1b5aa1 vmlinux
```

Yes, we finally get a nice 9% size reduction. Let's activate LTO on top of that: 

```
$ ./scripts/config --enable LTO_MENU
        $ make
        $ wc -l Module.symvers
        294 Module.symvers
        $ size vmlinux
           text    data     bss     dec     hex filename
        1156851  136272  104512 1397635  155383 vmlinux
```

Not only did we get a 29% size reduction, but thanks to LTO, the compiler was able to optimize things so that in the end 135 fewer exported symbols were needed. Great progress at last! 

#### Let's cut more trees

The next source of dependency trees, and potentially unused code, is system calls. Certainly we should be able to axe a couple of them. After all, our tiny user space to go along with our tiny kernel probably won't need most of them anyway. 

The kernel configuration system already provides some options to enable or disable support for some system calls. Here are a few examples that our user space certainly can live without: 

  * `CONFIG_SYSFS_SYSCALL` for the obsolete [`sysfs()`](<http://man7.org/linux/man-pages/man2/sysfs.2.html>) system call (not to be confused with the sysfs filesystem). 
  * `CONFIG_ADVISE_SYSCALLS` for the [`madvise()`](<http://man7.org/linux/man-pages/man2/madvise.2.html>) and [`fadvise64()`](<http://man7.org/linux/man-pages/man2/fadvise64.2.html>) system calls, which are not needed by many workloads. 
  * `CONFIG_POSIX_TIMERS` to [remove most support for POSIX timers](<https://lkml.org/lkml/2016/11/11/4>) while preserving the simplest cases. 

Turning off the above options produces this: 

```
text    data     bss     dec     hex filename
        1138567  135212  102404 1376183  14ffb7 vmlinux
```

That means a 1.5% size reduction compared to our previous kernel. This is not good if we need to rely on manual kernel configuration operations to get about 1% each time. What if we could scan our user space to automatically determine the list of required system calls just like we did for exported symbols? Such a scanning tool doesn't exist yet, but a simple lookup with `objdump` and `grep` on my busybox-based user space reveals 74 system call invocations, with some of them being duplicates of others. 

Of course we care about the environment and don't want to cut healthy trees. But here we're talking about virtual trees that can be regrown with a `make` command. So let's just do some clear-cutting in the system-call forest to see how far this approach could go before investing more efforts in a proper solution. Let's apply the following hack to our kernel to get the compiler to simply remove every system call: 

```
--- a/include/linux/syscalls.h
        +++ b/include/linux/syscalls.h
        @@ -214,7 +214,9 @@
                asmlinkage long SyS##name(__MAP(x,__SC_LONG,__VA_ARGS__));      \
                asmlinkage long SyS##name(__MAP(x,__SC_LONG,__VA_ARGS__))       \
                {                                                               \
        -               long ret = SYSC##name(__MAP(x,__SC_CAST,__VA_ARGS__));  \
        +               long ret;                                               \
        +               if (1) return -ENOSYS;                                  \
        +               ret = SYSC##name(__MAP(x,__SC_CAST,__VA_ARGS__));       \
                        __MAP(x,__SC_TEST,__VA_ARGS__);                         \
                        __PROTECT(x, ret,__MAP(x,__SC_ARGS,__VA_ARGS__));       \
                        return ret;                                             \
```

Rebuilding with the above hack applied and LTO still active produces this: 

```
text    data     bss     dec     hex filename
        1050735  135208  102348 1288291  13a863 vmlinux
```

Removing **all** system calls reduces the kernel by a mere 7.8%. This is rather disappointing. It is probably not worth pursuing this angle for now. 

#### Time to get dirty

We have probably gotten to the point where the return on the investment with automatic size reduction techniques is no longer worth the effort. Let's move to more involved approaches now, using explicit kernel configuration tweaking: 

  * Tiny embedded systems rarely have the luxury of a block device anyway, since they tend to be running directly on flash memory. So `CONFIG_BLOCK=n`. 
  * The STM32 system-on-chip is a no-MMU target; therefore any attempt to isolate users from each other is rather futile. So `CONFIG_MULTIUSER=n`. 
  * Some other goodies that we won't use anyway, and which can thus be disabled, include `CONFIG_TIMERFD`, `CONFIG_MEMBARRIER`, `CONFIG_COMPAT_BRK`, `CONFIG_PROC_SYSCTL`, etc. 
  * Let's select `CONFIG_SLOB` instead of `CONFIG_SLUB`; it should be sufficient for managing a small amount of memory, and it has smaller footprint too. 
  * With few user tasks, `CONFIG_PREEMPT_NONE` is most likely good enough. This has the nice side effect of enabling `CONFIG_TINY_RCU` and that sounds nice. 
  * We don't need all the kernel decompressors under the sun. So BZIP2, LZMA, LZO, LZ4, XZ are all gone. 

And then we get: 

```
text    data     bss     dec     hex filename
         926250  129292  101172 1156714  11a66a vmlinux
```

Okay! A 17% reduction compared to our last working kernel. Still, it feels like we ought to be making much more progress at this point. We're 41% smaller than the kernel we started with, but this kernel is also less capable. 

#### Time to get nasty

There is a `tinyconfig` make target that produces the smallest kernel possible. It doesn't boot on anything though. Still, this can be used as a starting point for yet more aggressive code modularization and axing. Let's have a look: 

```
$ make tinyconfig
        $ make vmlinux && size vmlinux
           text    data     bss     dec     hex filename
         508792   93844   20956  623592   983e8 vmlinux
```

This is getting nicely small now. However, this is still big for a kernel that, by definition, has everything configured out. Let's see where the bulk of the remaining code sits, with a little scripting: 

```
$ for f in */built-in.o; do
        >     size -t $f | tail -1 | sed "s|(TOTALS)|$f|"
        > done | sort -nr
         141183   15705    9377  166265   28979 kernel/built-in.o
         111627     816    5436  117879   1cc77 fs/built-in.o
          82893    5250    3493   91636   165f4 mm/built-in.o
          79197    3173    1788   84158   148be drivers/built-in.o
          65043     124      82   65249    fee1 lib/built-in.o
           5078   13894      61   19033    4a59 init/built-in.o
           2381       0       0    2381     94d security/built-in.o
```

That's our tinyconfig kernel. This is without LTO, otherwise it is much more difficult to get a size breakdown per subsystem like the above. But we've already seen that, even with a stripped-down kernel configuration, LTO has its limits. Regardless, this still raises some questions: 

  * Why is there still 77KB of driver code when there are no drivers configured in? 
  * Is it really necessary to entertain 80KB of memory-management support on a MMU-less target? 
  * Can we get rid of some of that 110KB of filesystem infrastructure when there is no need for a full-fledged filesystem support in our tiny system? 

Digging into the `lib` directory we can spot some low-hanging fruit: 

```
$ size lib/crc32.o
           text    data     bss     dec     hex filename
          25528       0       0   25528    63b8 lib/crc32.o
```

The CRC32 checksum implementation is not small; fortunately, a smaller alternative is available: 

```
$ ./scripts/config --disable CRC32_SLICEBY8
        $ ./scripts/config --enable CRC32_BI
        $ make lib/ && size lib/crc32.o
           text    data     bss     dec     hex filename
            340       0       0     340     154 lib/crc32.o
```

Wonderful. Now could we do the same with the VFS layer with, for example, an alternative implementation of the dentry-cache code that has no hyper-parallelized algorithms that scale to thousands of concurrent users? Perhaps it could be just some dumb code that preserves the existing interface but which does the job fully serialized in the simplest way possible… providing another 75x smaller footprint reduction maybe? But there are other challenges to overcome before taking this approach. 

#### When the winter hits the forest

Consider another possible opportunity for size reduction: the TTY layer. Digging into the `drivers/` directory of our STM32 kernel, we can see this: 

```
text    data     bss     dec     hex filename
        59572    1447    2713   63732    f8f4 drivers/tty/built-in.o
```

Spending 60KB on kernel code and data to transfer bytes over a serial port — not counting the dynamically allocated memory — seems unnecessary in our tiny environment. An attempt at an alternative TTY layer to that effect was proposed (see the associated [LWN coverage](<https://lwn.net/Articles/721074/>)). Making more pieces of the kernel optional was also attempted. The scheduler is one part of the kernel where significant parts can be carved out pretty easily (also [covered by LWN](<https://lwn.net/Articles/725376/>)). 

Unfortunately, those attempts were welcomed with a cold headwind that left me alone in the woods with frozen fingers. 

#### A different angle

One thing that usually mitigates the wind-chill effect in the community is a working proof of concept. A prominent characteristic of tiny microcontrollers is an amount of on-chip flash memory that is typically a few times larger than the on-chip RAM. So to achieve our goal of running Linux solely from the on-chip resources, we should stuff as much code and data as possible into flash memory, and execute code directly from there while keeping actual RAM usage to the bare minimum. This has to happen for both the kernel _and_ user space of course. Kernel eXecute-In-Place (XIP) has been available for quite a while, but it's another story when it comes to user space. That will be the subject of [the next (and final) article](<https://lwn.net/Articles/748198/>).
