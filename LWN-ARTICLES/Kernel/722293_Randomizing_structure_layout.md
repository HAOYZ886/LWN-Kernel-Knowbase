---
title: Randomizing structure layout
url: https://lwn.net/Articles/722293/
date: "May 11, 2017"
category: "Security-Kernel hardening"
author: "May 11, 2017 This article was contributed by Nur Hussein"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

May 11, 2017

This article was contributed by Nur Hussein

Kees Cook is working on a series of [patches for C structure randomization](<http://www.openwall.com/lists/kernel-hardening/2017/04/06/14>) to improve security in the Linux kernel. This is an important part of obfuscating the internal binary layout of a running kernel, making kernel exploits harder. The `randstruct` plugin is a new GCC add-on that lets the compiler randomize the layout of C structures. When enabled, the plugin will scramble the layout of the kernel structures that are specifically designated for randomization.

The patches in question are part of the [Kernel Self Protection Project](<https://kernsec.org/wiki/index.php/Kernel_Self_Protection_Project>) (KSPP). The goal of KSPP is to harden the mainline Linux kernel. Currently, KSPP is primarily working on porting features from [grsecurity](<https://grsecurity.net/>)/[PaX](<https://pax.grsecurity.net/>) to mainline Linux, and sending them incrementally to the Linux kernel mailing list. The structure randomization patches are part of that effort, and is a port of grsecurity's structure layout randomization plugin.

#### Structure randomization

Fields in a C structure are laid out by the compiler in order of their declaration. There may or may not be padding between the fields, depending on architecture alignment rules and whether the "`packed`" attribute is present. One technique for attacking the kernel is using memory bounds-checking flaws to overwrite specific fields of a structure with malicious values. When the order of fields in a structure is known, it is trivial to calculate the offsets where sensitive fields reside. A useful type of field for such exploitation is the function pointer, which an attacker can overwrite with a location containing malicious code that the kernel can be tricked into executing. Other sensitive data structures include security credentials, which can result in privilege-escalation vulnerabilities when overwritten.

The `randstruct` plugin randomly rearranges fields at compile time given a randomization seed. When potential attackers do not know the layout of a structure, it becomes much harder for them to overwrite specific fields in those structures. Thus, the barrier to exploitation is raised significantly, providing extra protection to the kernel from such attacks. Naturally, compiler support is necessary to get this feature to work. Since kernel 4.8, [GCC's plugin infrastructure](<https://lwn.net/Articles/691102/>) has been used by the kernel to implement such support for KSPP features.

To get structure randomization working in the kernel, a few things need to be done to ensure that it works smoothly without breaking anything. Once enabled, the `randstruct` plugin will do its magic provided a few conditions are met. First, structures marked for randomization need to be tagged with the `__randomize_layout` annotation. However, structures consisting entirely of function pointers are automatically randomized. Structures that only contain function pointers are a big target for attackers and reordering them is unlikely to cause problems elsewhere. This behavior can be overridden with the `__no_randomize_layout` annotation, when such randomization is undesirable. Therefore, if enabled, structure randomization is opt-in, except for structures that only contain function pointers, in which case it becomes opt-out. An example of a situation where `__no_randomize_layout` is needed is [this patch](<http://www.openwall.com/lists/kernel-hardening/2017/04/06/25>) from Cook, in which some paravirtualization structures (consisting entirely of function pointers) are used outside the kernel, hence should not be auto-randomized.

Structures to be randomized need to be initialized with [designated initializers](<https://gcc.gnu.org/onlinedocs/gcc/Designated-Inits.html>). A designated initializer is a C99 feature where the members of a C structure are initialized in any order explicitly by member name instead of anonymously by order of declaration. Also, structure pointers should not be cast to other pointer types for randomized structures. Cook has sent a number of patches to convert a few sensitive structures to use designated initializers, but their use has been standard practice in the kernel for some time now, so the kernel is pretty much ready for that feature. Structures that explicitly require designated initializers can be [tagged with](<http://www.openwall.com/lists/kernel-hardening/2017/04/06/13>) `__designated_init`; that will trigger a warning if a designated initializer is not used when initializing them.

#### Randomization of `task_struct`

This is a change that affects the source code in many places, so careful consideration is required when reordering some structure's internals as there are special cases that need to be handled. The [`task_struct`](<http://elixir.free-electrons.com/linux/latest/source/include/linux/sched.h#L483>) structure is a prime example of a structure that benefits from field randomization. Inside `task_struct` are sensitive fields such as process credentials, flags for enabling or disabling process auditing, and pointers to other `task_struct` structures. Those fields, among others, are juicy targets for potential attackers to overwrite. However, we can't just randomize the entirety of `task_struct`, as some fields on the very top and very bottom of the structure need to be where they are.

The top of `task_struct` is as follows:

```
struct task_struct {
        #ifdef CONFIG_THREAD_INFO_IN_TASK
    	    /*
    	     * For reasons of header soup (see current_thread_info()), this
    	     * must be the first element of task_struct.
    	     */
    	    struct thread_info		thread_info;
        #endif
```

To make it easy for `current_thread_info()` to get to the `thread_info` structure of the current running thread, it is possible to just use a `struct thread_info` pointer to the first element of the `task_struct` without having to actually include the file that defines it. This is to avoid circular header dependencies that arise from such an inclusion. Therefore, `thread_info` needs to be at that fixed location for the pointer access to work.

The bottom of `task_struct` has a similar position-locked structure:

```
/* CPU-specific state of this task: */
    	    struct thread_struct		thread;
    
    	    /*
    	     * WARNING: on x86, 'thread_struct' contains a variable-sized
    	     * structure.  It *MUST* be at the end of 'task_struct'.
    	     *
    	     * Do not put anything below here!
    	     */
        };
```

From here we can see that the implementation of `thread_struct` is architecture-specific, can be variable in size and sensitive to alignment. Thus it needs to be placed at the very end, and cannot be shifted around.

Linus Torvalds [weighed in on `task_struct` randomization](<https://marc.info/?l=linux-kernel&m=149083181100577&w=2>): 

Making "struct task_struct" be something that contains a fixed beginning and end, and just have an unnamed randomized part in the middle might be the way to go. 

Thus, the solution in Cook's patch is to [introduce another sub-structure](<http://www.openwall.com/lists/kernel-hardening/2017/04/06/27>) that encompasses the middle of `task_struct`, just after the first fields and just before the last one, and to mark that as randomizable. Any other fields that need to be position-locked can be carved out with more sub-structures without the `__randomize_layout` annotation; two such fields are `blocked` and `saved_sigmask` (both `sigset_t`). These fields are directly copied to user space and are expected to be adjacent and in that order.

#### More caveats

There is one major caveat to structure randomization: to build third party or out-of-tree kernel modules against a kernel with randomized structures, the randomization seed is required. Therefore, those distributing kernel binaries (such as Linux distributions) will need a way to expose the randomization seed to users that install the kernel headers or other kernel development package; attackers can use that to defeat the randomization. Since the same seed will be used across all instances of that particular distribution (as the seed needs to be chosen at compile time), any successful attack on a distribution kernel would work for all installations of that distribution kernel version. Nevertheless, compile-time randomization remains useful for custom private kernel builds, where the seed need not be exposed. Cook explains: 

While less useful for distribution kernels (where the randomization seed must be exposed for third party kernel module builds), it still has some value there since now all kernel builds would need to be tracked by an attacker. It is most useful to "in-house" kernel builds where the randomization seed is not available to an attacker. 

Nevertheless, Torvalds is unimpressed by structure randomization, [calling it security theater](<https://marc.info/?l=linux-kernel&m=149083181100577&w=2>). The fact that distributions need to publish the randomization seeds for module-building meant it did not provide as big of a security feature as advertised. Torvalds however did add: ""So it's imnsho a pretty questionable security thing. It's likely most useful for one-off 'special secure installations' than mass productions. "" To which, Cook [replied](<https://marc.info/?l=linux-kernel&m=149083865202302&w=2>): ""Well, Facebook and Google don't publish their kernel builds. :)"" 

There is a good argument to be made that large production servers running custom kernels do benefit from additional security protections such as structure randomization, so it is a worthwhile addition to the mainline.
