---
title: "Shrinking the kernel with link-time optimization"
url: https://lwn.net/Articles/744507/
date: "January 18, 2018"
category: Build system; Optimization tools
author: "January 18, 2018 This article was contributed by Nicolas Pitre"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

January 18, 2018

This article was contributed by Nicolas Pitre

This is the second article of a series discussing various methods of reducing the size of the Linux kernel to make it suitable for small environments. The [first article](<https://lwn.net/Articles/741494/>) provided a short rationale for this topic, and covered the link-time garbage collection, also called the `ld --gc-sections` method. We've seen that, though it is pretty straightforward, link-time garbage collection has issues of its own when applied to the kernel, making achieving optimal results more difficult than it is worth. In this article we'll have a look at what the compiler itself can do using link-time optimization.

Please note that most examples presented here were produced using the ARM architecture, however the principles themselves are architecture-independent.

#### Dead-code elimination

Kernel developers often rely on a compiler feature called "dead-code elimination". This is an important optimization that results in unreachable code simply being dropped from the final binary. Unlike the linker garbage-collection feature, dead-code elimination can be performed by the compiler both within and across functions inside a compilation unit or file.

Let's reuse the example code we used previously as `test.c` to illustrate it:

```
int foo(void)  { return 1; }
    
        int bar(void)  { return foo() + 2; }
    
        int main(void) { return foo() + 4; }
```

Again, the compiler generates the following (simplified) assembly output for that code:

```
.text
    
            .type   foo, %function
        foo:
            mov     r0, #1
            bx      lr
    
            .type   bar, %function
        bar:
            push    {r3, lr}
            bl      foo
            adds    r0, r0, #2
            pop     {r3, pc}
    
            .type   main, %function
        main:
            push    {r3, lr}
            bl      foo
            adds    r0, r0, #4
            pop     {r3, pc}
```

Despite `bar()` not being called, it is still part of the compiled output because there is no way for the compiler to actually know whether code in some other file might call it. But the author of that code often knows that, and can tell the compiler about it with the `static` qualifier as follows:

```
static int foo(void) { return 1; }
    
        static int bar(void) { return foo() + 2; }
    
        int main(void)       { return foo() + 4; }
```

By marking `foo()` and `bar()` static, the developer renders them no longer reachable from other source files. The compiler is then free to perform more optimizations on the compiled code. Of course the entry point (`main()`) must remain externally accessible and therefore cannot be static.

The above compiles to this:

```
.text
    
            .type   main, %function
        main:
            mov     r0, #5
            bx      lr
```

Boom! Not only did the compiler get rid of the unused `bar()` with dead-code elimination, but it also merged `foo()` directly into `main()` due to automatic inlining. In addition, it performed the arithmetic operation up front since all of the operands are constants, so that all we have left in the compiled code is the load of the resulting value and the return instruction. Instant code-size reduction that already works better than link-time garbage collection!

As mentioned, this dead-code elimination is heavily relied upon in the Linux kernel source tree, so that large portions of the code can be optimized away at compile time. For example, let's consider the following from [`include/linux/mmzone.h`](<http://elixir.free-electrons.com/linux/v4.14.13/source/include/linux/mmzone.h#L864>):

```
static inline int is_highmem_idx(enum zone_type idx)
        {
        #ifdef CONFIG_HIGHMEM
                return (idx == ZONE_HIGHMEM ||
                        (idx == ZONE_MOVABLE && zone_movable_is_highmem()));
        #else
                return 0;
        #endif
       }
```

When `CONFIG_HIGHMEM` is not defined, `is_highmem_idx()` (and `PageHighMem()` derived from it) return zero unconditionally. Any code within functions that follows the "`if (PageHighMem(page))`" pattern will be automatically optimized away as dead code.

But this works only because `is_highmem_idx()` is marked `static`; to avoid duplication of that function everywhere `mmzone.h` is included it has to be marked `inline` too. Those optimizations only work within a single compilation unit, missing out on opportunistic dead code elimination across different compilation units. So, what can we do short of concatenating all source files into a single one and making everything static to achieve the full benefit?

As mentioned previously, the core-kernel APIs are split into different C files for ease of maintenance. Those files may provide functions that are not called when some unwanted feature is configured out. It could be argued that the unused core functions should be `#ifdef`'d in or out along with their call sites, but this gets hairy when multiple features sharing the same core API may be configured in and out independently. To complicate things further, those core functions might be called within "`if (PageHighMem(page))`" blocks showing no directly visible relationship with a configuration option. So there are limits to how much unused code can be removed by the compiler; doing a more thorough job requires a tool like link-time optimization. 

#### Link-time optimization (LTO)

What is it? LTO is a compilation mode that instructs the compiler to parse the code into an abstract internal representation as usual, and store that representation directly into the resulting object file _without_ any optimization, rather than optimizing and assembling it into final machine instructions. Then, at link time when all the different object files are gathered together, the compiler intercepts the link process to reload that internal representation from all of the object files at once; only then will it perform its optimization passes — on the whole program — before the actual link. So it is basically just like if it concatenated all source files into a single one and made everything static. Great, that is exactly what we wished for.

Let's see how this works in practice with our little example program by having each function in its own file:

```
$ gcc -O2 -flto -c foo.c
       $ gcc -O2 -flto -c bar.c
       $ gcc -O2 -flto -c main.c
       $ gcc -O2 -flto -o test foo.o bar.o main.o
    
       $ nm test | grep "foo\|bar\|main"
       000102c0 T main
       $ objdump -d test
       [...]
       000102c0 <main>:
          102c0:       e3a00005        mov     r0, #5
          102c4:       e12fff1e        bx      lr
```

As expected, the result is the same as our earlier test despite having separate source files containing non-static functions.

#### LTO and the kernel

LWN [first covered LTO for the kernel](<https://lwn.net/Articles/512548/>) more than five years ago. Since then, things have improved a lot. LTO still isn't supported in the mainline, but Andi Kleen's [kernel LTO patchset](<https://github.com/andikleen/linux-misc/tree/lto-415-2>) has become much simpler as basic code correctness issues, which LTO is pickier about, have been merged upstream, and many LTO bugs in GCC have been fixed.

One of the biggest LTO showstoppers for the kernel had to do with the fact that a special version of [binutils](<https://www.gnu.org/software/binutils/>) was required. The kernel used to rely solely on partial linking (`ld -r`) when recursively gathering subdirectory build results, however `ld -r` doesn't support objects with LTO data unless binutils is patched to do so. And it was [rather unlikely](<http://sourceware.org/ml/binutils/2011-04/msg00433.html>) that the necessary patch would ever be merged in the upstream binutils tree. Nowadays the kernel build system can use [thin archives](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=a5967db9af51a84f>) instead of `ld -r`, making LTO of the kernel possible with the upstream tools that most distributions ship.

LTO also has a big advantage over link-time garbage collection given that it does not require separate linker sections for each exception table entry and does not suffer from the "backward reference" and "missing forward reference" problems described in the previous article. When the compiler optimizes code away, those exception table entries instantiated by that code are simply dropped automatically with it.

#### Numbers please!

Let's not forget that our end goal is to fit Linux into tiny systems. So it is about time we looked at actual kernel-size numbers. Let's pick the [STM32](<https://en.wikipedia.org/wiki/STM32>) target which represents the kind of tiny systems we're aiming for. The advantage here is that mainline Linux already runs on most STM32 microcontrollers, albeit with external RAM. The baseline kernel version is v4.15-rc1 plus the LTO patches.

First, with LTO disabled:

```
$ make stm32_defconfig
        $ make vmlinux
        $ size vmlinux
           text    data     bss     dec     hex filename
        1704024  144732  117660 1966416  1e0150 vmlinux
```

And with LTO enabled:

```
$ ./scripts/config --enable CONFIG_LTO_MENU
        $ make vmlinux
        $ size vmlinux
           text    data     bss     dec     hex filename
        1281644  142492  112985 1537121  177461 vmlinux
```

This is a 22% size reduction right there. For completeness, let's see how link-time garbage collection as described in the previous article fares:

```
$ [hacks for CONFIG_LD_DEAD_CODE_DATA_ELIMINATION]
        $ make vmlinux
        $ size vmlinux
           text    data     bss     dec     hex filename
        1304516  141672  113108 1559296  17cb00 vmlinux
```

Here we get a 21% size reduction. However, this comes with a big disclaimer due to the following hacks:

  * No `KEEP()` statements were added to the ARM linker file as required. Worse: the `ASSERT()` statements about missing processor and architecture tables have been disabled for the sake of successful compilation. This means important pieces of code and data are missing from this kernel. 

  * The ARM unwinding facility needed for function backtraces has been forcefully disabled as it also contained a reference to every function, making garbage collection ineffective. So, unlike the LTO-built kernel, this one would lack an important debugging facility. 

Of course those hacks produce a non-functional kernel. Still, the size reduction is slightly lower than what LTO produces, and it would be even less if proper link-time garbage collection support was implemented. And optimal link-time garbage collection as described in the previous article is way more invasive than LTO. We therefore have a clear winner here.

One could wonder if size reduction could improve further by combining both link-time optimization and link-time garbage collection. The answer is no since, once LTO has removed every piece of dead code, there is simply nothing left to garbage-collect.

#### More numbers

So LTO seems to be the best thing since sliced bread, right? Well, it has drawbacks of its own. The most significant is build time. Let's repeat the above kernel compilation sequence to see what we get.

First with LTO disabled:

```
$ make clean
        $ make stm32_defconfig
        $ time make -j8 vmlinux
    
        real    0m36.645s
        user    3m59.252s
        sys     0m21.026s
```

And with LTO enabled:

```
$ make clean
        $ ./scripts/config --enable CONFIG_LTO_MENU
        $ time make -j8 vmlinux
    
        real    1m24.774s
        user    8m4.143s
        sys     0m31.902s
```

LTO requires 1.9x more CPU time and 2.3x more wall-clock time to build the kernel. Having code optimizations performed at the very end creates a bigger serialization point, unlike traditional builds where individual source files are compiled and optimized concurrently without LTO.

But the most annoying case, at least for a kernel developer, is partial rebuild time after some source-code modifications. Without LTO we get:

```
$ touch init/main.c
        $ time make -j8 vmlinux
    
        real    0m3.686s
        user    0m5.803s
        sys     0m1.819s
```

And with LTO enabled this becomes:

```
$ touch init/main.c
        $ time make -j8 vmlinux
    
        real    0m58.283s
        user    5m6.089s
        sys     0m12.732s
```

A partial build with LTO is about 15x longer than the non-LTO case, and not very far from the full build time. So LTO is clearly not something suitable during frequent debug/rebuild/test cycles.

And for completeness:

```
$ make clean
        $ [hacks for CONFIG_LD_DEAD_CODE_DATA_ELIMINATION]
        $ time make -j8 vmlinux
    
        real    0m37.572s
        user    3m58.826s
        sys     0m21.616s
```

More or less the same result as our initial build. Clearly link-time garbage collection is basically free in terms of build time which is its biggest (perhaps only) advantage.

Test-build environment details 
    

GCC version 6.3.1 20170404 (Linaro GCC 6.3-2017.05)  
Intel® Core™ i7-4770R CPU @ 3.20GHz  
Samsung SSD 850 EVO 500GB 

#### Conclusion

We have two approaches for automatic kernel-size reduction at our disposal, each with a different set of compromises. However the advantage is clearly on the LTO side when considering maintenance costs and intrusiveness. And build time becomes tolerable when building very small kernels anyway. But, did we manage to get a "very small kernel"? Kernels that cross the one-megabyte range cannot realistically be qualified as "very small" or even "tiny" yet. Clearly automatic size reduction alone won't be sufficient, so more assertive approaches will be required to achieve our goal. That will be the subject of the next article.

Meanwhile, anybody wanting to play with LTO with their own kernel in the short term should start with [these instructions](<https://github.com/andikleen/linux-misc/blob/lto-415-2/Documentation/lto-build>) found in Kleen's patch set. 

The next article in this series is [Shrinking the kernel with an axe](<https://lwn.net/Articles/746780/>)
