---
title: "Shrinking the kernel with link-time garbage collection"
url: https://lwn.net/Articles/741494/
date: "December 15, 2017"
category: Build system; Embedded systems; Optimization tools
author: "December 15, 2017 This article was contributed by Nicolas Pitre"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

December 15, 2017

This article was contributed by Nicolas Pitre

One of the keys to fitting the Linux kernel into a small system is to remove any code that is not needed. The kernel's configuration system allows that to be done on a large scale, but it still results in the building of a kernel containing many smaller chunks of unused code and data. With a bit of work, though, the compiler and linker can be made to work together to garbage-collect much of that unused code and recover the wasted space for more important uses. 

This is the first article of a series discussing various methods of reducing the size of the Linux kernel to make it suitable for small environments. Several approaches will be examined, from the straightforward to the daring. 

It is a fact that Linux has conquered the high end of the computing spectrum. Since Linux started small 26 years ago, there was great emphasis on scaling up, which eventually made it the platform of choice for server rooms, data centers and the cloud infrastructure. Since November 2017, all the [top 500 supercomputers](<http://www.top500.org/lists/2017/11/>) use Linux-based operating systems. No more to say there.

On the desktop this is not as clear. Linux market share has only now surpassed the three-percent mark ([ according to Net Applications](<http://www.netmarketshare.com/operating-system-market-share.aspx?qprid=9&qpcustomb=0&qpsp=200&qpnp=25&qptimeframe=M>)) even though it has been available and working well for years. Unless we consider mobile as the new desktop, in which case the presence of at least [two-billion active Android devices](<https://blog.google/products/android/2bn-milestone/>) should make up for it and cover the middle range of the spectrum pretty well. But mobile devices, while being physically small, are far from matching the "tiny" Linux definition. Mobile is simply desktop software (more or less) that we can carry in our pocket but that only 20 years ago required a big computer tower to run.

Small often implies "embedded", and embedded Linux is often considered as being at the low end of the computing spectrum. Linux is successful in this area too; small, embedded Linux examples are plentiful, from wireless routers to smart light bulbs, from airline in-flight entertainment to car GPS systems, etc. Linux is everywhere, and most of the time its presence is unsuspected. But those systems still have a relatively generous amount of resources to work with, e.g. typically 32MB of RAM to start with. That's not _really_ tiny yet.

So what does "tiny" actually mean? Let's define it as a sub-megabyte system or thereabout. Battery-powered operation is a given and batteries are expected to last for months or years. Power consumption has to be extremely low, which implies static RAM (or SRAM). Because SRAM is expensive, it is typically deployed in minimal quantities. We're talking small, cheap, and ubiquitous [IoT](<http://en.wikipedia.org/wiki/Internet_of_things>) devices that are increasingly being connected to the Internet with all its perils. The software in this space is fragmented, and Linux has next to no presence at all.

Operating systems carrying an open-source license for the tiny space are many. A few random examples are: [Contiki](<http://www.contiki-os.org/>), [FreeRTOS](<http://www.freertos.org/>), [Mbed OS](<http://www.mbed.com/en/platform/mbed-os/>), [NuttX RTOS](<http://www.nuttx.org/>), [RIOT OS](<http://riot-os.org/>), or even [Fuzix OS](<http://www.fuzix.org/>) for the nostalgic. There is also an effort to consolidate this tiny space with the [Zephyr Project](<https://lwn.net/Articles/682723/>). Diversity is also prominent in the proprietary world. So why bother with Linux in this space?

#### The open-source gravitational field

Successful open-source projects may be compared to celestial objects. Some of them grow big, as universal gravitation works to pull them together into stars and planets. There are relatively few such objects in the end since the gravity field from large objects absorbs any other objects in their vicinity. Sometimes, however, objects that fail to consolidate with others remain numerous, roughly shaped, and sparsely distributed like asteroids.

The tiny computing space is just like an asteroid field; numerous projects exist, but they lack the required center of gravity for effective and self-sustained communities to form naturally around them. Consolidation efforts are moving slowly because of that. The end result is a highly fragmented space with relatively few developers per project and, therefore, fewer resources to rely upon when issues come up. Vulnerabilities are more likely to turn into a security nightmare.

The Linux ecosystem, instead, reached planet status a long time ago. It has a lot of knowledgeable people around it, a lot of testing infrastructure and tooling available already, etc. If a security issue turns up on Linux, it has a greater chance of being caught early, or fixed quickly, and finding people with the right knowledge is easier with Linux than it would ever be on any other tiny operating system out there. Leveraging that ecosystem could be a big plus for the tiny computing space, and it would truly mean world domination for Linux.

#### Scaling Linux down

With this short rationale in hand, let's dive into the actual technical stuff. Of course, the biggest obstacle to a tiny Linux kernel is its size. It is unrealistic to expect Linux to ever run in 64KB of RAM. Such targets are best left to the likes of Zephyr. But maybe "640K ought to be enough for anybody" as someone once said. Many modern microcontrollers capable of running Linux do have about that amount of on-chip SRAM which would make them nice single-chip Linux targets. So let's see how this can be achieved.

#### Automatic kernel size reduction

Wouldn't it be nice if computers could be leveraged to do the job automatically for us? After all, if there is one thing that computers are good at, it is finding unused code and optimizing the resulting binary. There are two ways in which currently available tools can achieve automatic size reduction: garbage collection of unused input sections and link-time optimization (LTO). This article will focus on the first of these two approaches. 

Please note that most examples presented here were produced using the ARM architecture, however the principles themselves are not architecture-dependent.

#### Linker section garbage collection

The linker is already able to omit stuff from a linked binary given extra input data. Let's think about the libraries used to produce an executable for example: if the whole of a library (say libc) were always linked into the final executable, then that executable would be rather big and inefficient. That's assuming static linking of course, but the kernel is a statically linked executable for the most part, so let's forget about dynamic linking and modules for the purpose of this demonstration.

Let's consider the following code as test.c:

```
int foo(void)  { return 1; }
    
       int bar(void)  { return foo() + 2; }
    
       int main(void) { return foo() + 4; }
```

The compiler generates the following (simplified) assembly output for that code:

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

Despite `bar()` not being called, it is still part of the compiled output because there is no way for the compiler to actually know if some other file might call it. Only the linker can know, once it gathered all the object files to be linked together, whether `bar()` is referenced from another object file or not.

And even then, the linker has no knowledge of the object file content other than the different sections it contains (.text, .data, etc.), where named things start (symbol table), and how to patch in final addresses (relocation table). So all the linker can do is to pull an object file into the final link if it contains a symbol that is referenced by another object file. The linker simply cannot carve out a piece of the `.text` section to drop the unused `bar()` code.

In the Linux kernel, this happens quite often when the core kernel API provides functions that are not called when some unwanted feature is configured out. It could be argued that the unused core function should be `#ifdef`'d in or out along with its user, but this gets hairy when multiple features sharing the same core API may be configured in and out independently. Tracking this kind of dependency can be done manually in the `Kconfig` language, but there is a point where too many configuration symbols and `#ifdef`s in the code become a maintenance burden.

Alternatively, we could move every core function into a separate source file, causing each to be compiled into its own object file. The linker could work out what is used and what is not like it does with libraries, but that wouldn't sit well with kernel developers either. Fortunately, there is a GCC flag to request the creation of a separate code section for every function, namely `-ffunction-sections`. When recompiling our test code above with `-ffunction-sections` the output becomes:

```
.section .text.foo,"ax",%progbits
            .type   foo, %function
        foo:
            mov     r0, #1
            bx      lr
    
            .section .text.bar,"ax",%progbits
            .type   bar, %function
        bar:
            push    {r3, lr}
            bl      foo
            adds    r0, r0, #2
            pop     {r3, pc}
    
            .section .text.main,"ax",%progbits
            .type   main, %function
        main:
            push    {r3, lr}
            bl      foo
            adds    r0, r0, #4
            pop     {r3, pc}
```

Now, we get three distinct sections rather than the single `.text` section we had initially, one per function, named after the function they contain plus some attributes to indicate that they contain executable code. Separate sections are just as good as separate object files to the linker, since it can drop unreferenced code on a per function granularity as long as the linker is also passed the `-gc-sections` flag.

Let's try it out, first without any special flags:

```
$ gcc -O2 -o test test.c
        $ ./test
        $ echo $?
        5
        $ nm test | grep "foo\|bar"
        00008520 T bar
        000084fc T foo
```

The code works as expected, and the bar symbol is still present as expected. Let's add our special flags now (using the special "`-Wl`" flag to pass options through to the linker):

```
$ gcc -ffunction-sections \
        >     -Wl,-gc-sections -Wl,-print-gc-sections \
        >     -O2 -o test test.c
        ld: Removing unused section '.text.bar' in file 'test.o'
        $ ./test
        $ echo $?
        5
        $ nm test | grep "foo\|bar"
        000084fc T foo
```

Now `bar()` is gone. We get an automatic size reduction and we didn't have to modify the source code at all. Instant gratification! The attentive reader would have noticed the extra `-print-gc-sections` linker flag that may be used to confirm what sections are actually removed.

In addition to `-ffunction-sections`, GCC also accepts `-fdata-sections` to perform the same split-section trick with global data variables. It is typical to see both flags used together.

Can we just add those flags to the kernel build? Unfortunately, things aren't that simple. Just adding those flags will produce a non-booting kernel. The problem comes from various sections that the kernel code creates to let the linker gather scattered pieces of information together into a table that can be used by the running kernel. For example, let's consider this macro that closely resembles what's used to implement `put_user()` on ARM:

```
#define __put_user_asm_word(x, __pu_addr, err)              \
            __asm__ __volatile__(                                   \
            "1:     strt    %1, [%2]\n"                             \
            "2:\n"                                                  \
            "       .pushsection .text.fixup,\"ax\"\n"              \
            "       .align  2\n"                                    \
            "3:     mov     %0, %3\n"                               \
            "       b       2b\n"                                   \
            "       .popsection\n"                                  \
            "       .pushsection __ex_table,\"a\"\n"                \
            "       .align  3\n"                                    \
            "       .long   1b, 3b\n"                               \
            "       .popsection"                                    \
            : "+r" (err)                                            \
            : "r" (x), "r" (__pu_addr), "i" (-EFAULT)               \
            : "cc")
```

For those who might wonder what those "`1b`", "`2b`" and "`3b`" symbols might be: those are backward references to the respective local labels without the "`b`" suffix. The "`f`" suffix is also available for forward references. See [the binutils documentation](<https://sourceware.org/binutils/docs/as/Symbol-Names.html#Local-Labels-1>) for more details.

The code above adds a single `strt` instruction in the currently active section, which is a special instruction that perform a user-space store after testing user access permissions. It also adds some code to the `.text.fixup` section, and finally records in the `__ex_table` section the location of the `strt` instruction and the `.text.fixup` code.

To better illustrate what's happening, let's consider this code:

```
int foobar(int __user *p)
        {
            return put_user(0x5a, p);
        }
```

The assembly result is:

```
.section .text.foobar,"ax"
        foobar:
            mov     r3, #0
            mov     r2, #0x5a
        1:  strt    r2, [r0]
        2:  mov     r0, r3
            bx      lr
    
            .section .text.fixup,"ax"
        3:  mov     r3, #-EFAULT
            b       2b
    
            .section __ex_table,"a"
            .long   1b, 3b
```

In the example above, the code preloads zero into `r3`, performs the user-space access and, if an exception occurs, `r3` is loaded with `-EFAULT` by the fixup code and execution is resumed past the faulting instruction.

What's important to remember here is that the linker gathers the `__ex_table` sections from every `put_user()` instance into a single section to form a table. That table is searched by the kernel exception-handling code to decide what to do if the faulting instruction matches a table entry when an exception occurs.

The problem with these `__ex_table` sections is that nothing has an actual reference to them. They are just a bunch of address values pulled together. So, when the linker is passed the `-gc-sections` flag, it is free to drop all of them because they aren't referenced by anyone since they don't define any symbol themselves anyway. So we end up with a final kernel that has an empty exception table. This is true for many such kernel tables created by the linker, such as the list of kernel command-line argument parsers or the the initcall pointer table. And a kernel without any initcalls won't boot very far.

Of course there is a linker script directive that allows for overriding the `-gc-sections` effect on a per section basis, namely the `KEEP()` directive. So [the kernel linker script](<http://elixir.free-electrons.com/linux/v4.14.6/source/include/asm-generic/vmlinux.lds.h#L521>) has gained entries that look like this:

```
__ex_table {
    	__start___ex_table = .;
    	KEEP(*(__ex_table))
    	__stop___ex_table = .;
        }
```

With the appropriate sprinkling of `KEEP()` annotations in the linker script, the kernel does eventually boot properly. Yay! So now the extent of our modifications consists of a few extra flags to the compiler/linker and a couple `KEEP()` annotations in the linker script. That is, in fact, what the mainline kernel already offers since v4.10 with the `CONFIG_LD_DEAD_CODE_DATA_ELIMINATION` configuration option.

#### The "backward reference" problem

Maybe we shouldn't celebrate just yet. Let's consider multiple functions like the previous example, each with a call to `put_user()`. We'd end up with something like the following assembly representation after the final link:

```
.section .text.foo1,"ax"
        foo1:
        	...
    	mov     r3, #0
        1:  strt    ...
        2:  ...
    
            .section .text.foo2,"ax"
        foo2:
       	...
            mov     r3, #0
        3:  strt    ...
        4:  ...
    
            .section .text.foo3,"ax"
        foo3:
    	...
            mov     r3, #0
        5:  strt    ...
        6:  ...
    
            .section .text.fixup,"ax"
        7:  mov     r3, #-EFAULT
            b       2b
        8:  mov     r3, #-EFAULT
            b       4b
        9:  mov     r3, #-EFAULT
            b       6b
    
            .section __ex_table,"a"
            .long   1b, 7b
            .long   3b, 8b
            .long   5b, 9b
```

Here we clearly see the `__ex_table` section containing a table of tuples, each with the address of a potentially exception-raising instruction and the address of the code to execute in that case. Of course our linker script has a `KEEP()` on the individual `__ex_table` entries to pull them into the final binary, otherwise the linker would discard them. But despite the fact that the `__ex_table` entries don't define symbols of their own, they _do_ reference other symbols, namely the location of the `strt` instructions, illustrated by backward references to labels 1, 3 and 5. References to the fixup code also pull in the corresponding section, which also has yet more references to those individual functions.

That means all those functions with a `put_user()` call in them, and all other functions using similar constructs that create table entries using the same mechanism, are always pulled into the final binary even if they are never referenced by anything else. And those functions will pull in all the functions they call, and so on, down to leaf functions. This makes the whole idea of dropping unused code by garbage-collecting unreferenced sections rather ineffective in this case.

What can we do about that? We could apply the same trick already applied to those original functions themselves and create separate sections for each of the exception and fixup entries so the linker can link some of them and drop the others. However, in the function case, it is the compiler that does the section splitting for us with `-ffunction-sections`. Here we're providing our own assembly stubs.

One could suggest something like `__put_user(val, ptr, __func__)`. The compiler provides the `__func__` identifier that holds the name of the current function as a string. That could be used with the `.pushsection` directive to create section names after the function where this is invoked:

```
#define __put_user_asm_word(x, __pu_addr, err)              \
            __asm__ __volatile__(                                   \
            "1:     strt    %1, [%2]\n"                             \
            "2:\n"                                                  \
            "       .pushsection .text.fixup." __func__ ",\"ax\"\n" \
            "       .align  2\n"                                    \
            "3:     mov     %0, %3\n"                               \
            "       b       2b\n"                                   \
            "       .popsection\n"                                  \
            "       .pushsection __ex_table." __func__ ",\"a\"\n"   \
            "       .align  3\n"                                    \
            "       .long   1b, 3b\n"                               \
            "       .popsection"                                    \
            : "+r" (err)                                            \
            : "r" (x), "r" (__pu_addr), "i" (-EFAULT)               \
            : "cc")
```

The problem is that `__func__` is not a string literal. It is a string pointer and therefore cannot be used to construct a string for the `asm()` statement. What about `__FUNCTION__` then? That used to work. However, [the GCC documentation](<https://gcc.gnu.org/onlinedocs/gcc-3.4.0/gcc/Function-Names.html>) says:

These identifiers are not preprocessor macros. In GCC 3.3 and earlier, in C only, __FUNCTION__ and __PRETTY_FUNCTION__ were treated as string literals; they could be used to initialize char arrays, and they could be concatenated with other string literals. GCC 3.4 and later treat them as variables, like _func_. In C++, __FUNCTION__ and __PRETTY_FUNCTION__ have always been variables.

What about `__put_user(val, ptr, __FILE__, __LINE__)`? That can work to some extent, as `__FILE__` is a string literal and `__LINE__` can be stringified. But this scheme falls flat when invoking static inline functions as the file and line information correspond to the function definition location and not to where it is inlined. That means multiple instances would end up with the same section name, which is precisely what we're trying to avoid.

The ultimate and simplest solution requires some involvement from the assembler. A [ section name substitution sequence](<http://sourceware.org/git/?p=binutils-gdb.git;a=commitdiff;h=451133cefa839104>) is possible when using binutils version 2.26 or later. With that feature, the previous `.pushsection` directives would simply become:

```
#define __put_user_asm_word(x, __pu_addr, err)              \
            __asm__ __volatile__(                                   \
            "1:     strt    %1, [%2]\n"                             \
            "2:\n"                                                  \
            "       .pushsection %S.fixup,\"ax\"\n"                 \
            "       .align  2\n"                                    \
            "3:     mov     %0, %3\n"                               \
            "       b       2b\n"                                   \
            "       .popsection\n"                                  \
            "       .pushsection __ex_table%S,\"a\"\n"              \
            "       .align  3\n"                                    \
            "       .long   1b, 3b\n"                               \
            "       .popsection"                                    \
            : "+r" (err)                                            \
            : "r" (x), "r" (__pu_addr), "i" (-EFAULT)               \
            : "cc")
```

Given the function `foobar()` invoking `put_user()`, the active section name would be `.text.foobar`. Then the fixup code would end up in section `.text.foobar.fixup` and the exception table entry in `__ex_table.text.foobar`. The linker has now the ability to include only the relevant parts of the exception table and fixup code.

Still, we didn't fix anything, did we?

#### The "missing forward reference" problem

At this point we have managed to create separate sections for functions, fixup code per function, and exception table entries per function. But we still need a `KEEP(__ex_table.*)` in the linker script or we'll still end up with all our exception sections discarded like before. Having separate sections with pretty names still doesn't create any reference to them.

What we need is some kind of explicit reference from the invoking code to the corresponding exception table entry so the linker will pull it in along the function when it is needed without having to forcefully `KEEP()` them. Something illustrated by a hypothetical `.tug` assembly directive like this:

```
.section .text.foobar,"ax"
        foobar:
            mov     r3, #0
            mov     r2, #0x5a
        1:  strt    r2, [r0]
            **.tug    4f**
        2:  mov     r0, r3
            bx      lr
    
            .section .fixup.text.foobar,"ax"
        3:  mov     r3, #-EFAULT
            b       2b
    
            .section __ex_table.text.foobar,"a"
        4:  .long   1b, 3b
```

Here, a simple call to `foobar()` from some other code will prompt the linker to pull the `foobar()` code in. That code, in turn, contains a reference to its exception entry, prompting the linker to pull that in too, and the exception entry has a reference to the fixup code which would be pulled into the link as well. Now we're going somewhere!

But how can we create such a reference? The most obvious way is to replace `.tug` with `.long` above, which would store the address of the exception table entry at that location. However this would require the code to branch over that value, which has no use other than creating a reference, wasting memory and making the code less optimal.

Turns out that the GNU assembler already has the necessary feature to create an explicit reference without allocating any space in the code, at least on ARM:

```
.section .text.foobar,"ax"
        foobar:
            mov     r3, #0
            mov     r2, #0x5a
        1:  strt    r2, [r0]
            **.reloc  ., R_ARM_NONE, 4f**
        2:  mov     r0, r3
            bx      lr
    
            .section .fixup.text.foobar,"ax"
        3:   mov     r3, #-EFAULT
            b       2b
    
            .section __ex_table.text.foobar,"a"
    4:      .long   1b, 3b
```

Yes, our `.tug` directive can be defined in terms of the `.reloc` directive with a no-op relocation type. What `.reloc` means and how relocations work is beyond the scope of this article though.

#### Conclusion

We've seen how the linker `-gc-sections` feature can be exploited to its full potential on the Linux kernel. This requires more changes to the source code than one might have hoped, involving some obscure (or exotic depending on your taste) assembler tricks. However this does not yet unveil the full extent of what the automatic size reduction current tools are capable of when using link-time optimization (LTO). So in the next article in this series we'll [look at LTO](<https://lwn.net/Articles/744507/>) instead.

Still, there is a longstanding issue with kernel code annotated with the `__exit` marker that can be fixed with what we've seen already. Because exception table entries hold references to the originating code, that code has to be pulled into the final link or unresolved symbol errors will occur. This is why the built-in `__exit` code is currently linked into the .init section to be dropped at runtime rather than being discarded upfront at link time. However, with the section name substitution feature, it is possible to create something that looks like:

```
.text
        foobar:
        1:  ...
    
            .section .init.text
        foobar_init:
        2:  ...
    
            .section .exit.text
        foobar_exit:
        3:  ...
    
            .section .text.fixup
            do_fix  1b
    
            .section .init.text.fixup
            do_fix  2b
    
            .section .exit.text.fixup
            do_fix  3b
```

This way, it would be possible to discard the `__exit` code as originally intended, along with the exception entries that reference it. Any takers?

The rest of this series is made up of: 

  * [Shrinking the kernel with link-time optimization](<https://lwn.net/Articles/744507/>)
  * [Shrinking the kernel with an axe](<https://lwn.net/Articles/746780/>)
  * [Shrinking the kernel with a hammer](<https://lwn.net/Articles/748198/>)
