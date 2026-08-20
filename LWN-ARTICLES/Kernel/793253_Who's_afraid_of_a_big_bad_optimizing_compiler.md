---
title: "Who's afraid of a big bad optimizing compiler?"
url: https://lwn.net/Articles/793253/
date: "July 15, 2019"
category: ACCESS ONCE
author: "July 15, 2019 (Many contributors)"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

July 15, 2019

(Many contributors)

This article was contributed by Jade Alglave, Will Deacon, Boqun Feng, David Howells, Daniel Lustig, Luc Maranget, Paul E. McKenney, Andrea Parri, Nicholas Piggin, Alan Stern, Akira Yokosawa, and Peter Zijlstra. 

When compiling Linux-kernel code that does a plain C-language load or store, as in "`a=b`", the C standard grants the compiler the right to assume that the affected variables are neither accessed nor modified by any other thread at the time of that load or store. The compiler is therefore permitted to carry out a large number of transformations, a couple of which were discussed in this [`ACCESS_ONCE()` LWN article](<https://lwn.net/Articles/508991/>), and another of which is described in [Dmitry Vyukov's KTSAN wiki page](<https://github.com/google/ktsan/wiki/READ_ONCE-and-WRITE_ONCE>). However, our increasingly aggressive modern compilers produce increasingly surprising code optimizations. Some of these optimizations might be especially surprising to developers who assume that each plain C-language load or store will always result in an assembly-language load or store. Although this article is written for Linux kernel developers, many of these scenarios also apply to other concurrent code bases, keeping in mind that "concurrent code bases" also includes single-threaded code bases that use interrupts or signals. 

The ongoing trend in compilers makes us wonder: "Just how afraid should we be?". The following sections should help answer this question: 

  1. Load tearing
  2. Store tearing
  3. Load fusing
  4. Store fusing
  5. Code reordering
  6. Invented loads
  7. Invented stores

**Quick quiz 1** : But we shouldn't be afraid at all for things like on-stack or per-CPU variables, right?   
Answer

  8. Store-to-load transformations
  9. Dead-code elimination
  10. How real is all this? 

This is followed by the ineludible answers to the quick quizzes. 

#### Load tearing

Load tearing occurs when the compiler uses multiple load instructions for a single access. For example, the compiler could, in theory, compile the load from `global_ptr` on line 1 of the following code as a series of one-byte loads. 

```
1 ptr = global_ptr; /* BUGGY if load tearing possible. */
      2 if (ptr != NULL && ptr < high_address) /* BUGGY!!! */
      3         do_low(ptr);
```

If some other thread was concurrently setting `global_ptr` to `NULL`, the result might have all but one byte of the pointer set to zero, thus forming a "wild pointer". Stores using such a wild pointer could corrupt arbitrary regions of memory, resulting in rare and difficult-to-debug crashes. 

Worse yet, on (say) an eight-bit system with 16-bit pointers, the compiler might have no choice but to use a pair of eight-bit instructions to access a given pointer. And even on today's 32-bit and 64-bit systems, a misaligned or too-large access could tear. Because the C standard must support all manner of systems, the standard cannot rule out load tearing in the general case. 

**Quick quiz 2** : But there are lots of plain loads from shared variables in the Linux kernel. These cannot possibly all be buggy, can they?   
Answer

However, the remainder of this article will assume properly aligned and machine-word-sized accesses, in which case `READ_ONCE()` will prevent load tearing. (In the Linux kernel, tearing of plain C-language loads [has been observed](<http://lkml.kernel.org/r/CAHk-=wj2t+GK+DGQ7Xy6U7zMf72e7Jkxn4_-kGyfH3WFEoH+YQ@mail.gmail.com>) even given properly aligned and machine-word-sized loads.) 

#### Store tearing

Store tearing occurs when the compiler uses multiple store instructions for a single access. For example, one thread might store `0x12345678` to a four-byte integer variable at the same time as another thread stored `0xabcdef00`. If the compiler used 16-bit stores for either access, the result might well be `0x1234ef00`, which could come as quite a surprise to code loading from this integer. Nor is this a strictly theoretical issue. For example, there are CPUs that feature small immediate values, and on such CPUs, the compiler can be tempted to split a 64-bit store into two 32-bit stores in order to reduce the overhead of explicitly forming the 64-bit constant in a register, even on a 64-bit CPU. There are historical reports of this actually happening in the wild, but there is also a [recent report](<https://lore.kernel.org/lkml/20190821103200.kpufwtviqhpbuv2n@willie-the-truck/>). Note that this tearing can happen even on properly aligned and machine-word-sized accesses, and in this particular case, even for volatile stores. Some might argue that this behavior constitutes a bug in the compiler, but either way it illustrates the perceived value of store tearing from a compiler-writer viewpoint. 

Of course, the compiler simply has no choice but to tear some stores in the general case, given the possibility of code using 64-bit integers running on a 32-bit system. But for properly aligned machine-sized stores, `WRITE_ONCE()` will prevent store tearing. 

#### Load fusing

Load fusing occurs when the compiler uses the result of a prior load from a given variable instead of repeating the load. Not only is this sort of optimization just fine in single-threaded code, it is often just fine in multithreaded code. Unfortunately, the word "often" hides some truly annoying exceptions, including the one called out in the `ACCESS_ONCE()` article. 

For example, suppose that a realtime system needs to invoke a function named `do_something_quickly()` repeatedly until the variable `need_to_stop` is set, and that the compiler can see that `do_something_quickly()` does not store to `need_to_stop`. One (unsafe) way to code this might look like: 

```
1 while (!need_to_stop) /* BUGGY!!! */
      2     do_something_quickly();
```

The compiler might reasonably unroll this loop sixteen times in order to reduce the per-invocation overhead of the backwards branch at the end of the loop. Worse yet, because the compiler knows that `do_something_quickly()` does not store to `need_to_stop`, the compiler could quite reasonably decide to check this variable only once, resulting in the code shown below: 

```
1 /* Optimized code */
      2 if (!need_to_stop)
      3     for (;;) {
      4         do_something_quickly();
      5	    do_something_quickly();
      6	    do_something_quickly();
      7	    do_something_quickly();
      8	    do_something_quickly();
      9	    do_something_quickly();
     10	    do_something_quickly();
     11	    do_something_quickly();
     12	    do_something_quickly();
     13	    do_something_quickly();
     14	    do_something_quickly();
     15	    do_something_quickly();
     16	    do_something_quickly();
     17	    do_something_quickly();
     18	    do_something_quickly();
     19	    do_something_quickly();
     20     }
```

Once entered, the loop on lines 3-20 will never stop, regardless of how many times some other thread stores a non-zero value to `need_to_stop`. The result will at best be disappointment, and might also include severe physical damage. 

The compiler can fuse loads across surprisingly large spans of code. For example, in this code: 

```
1 int *gp;
      2
      3 void t0(void)
      4 {
      5     WRITE_ONCE(gp, &myvar);
      6 }
      7
      8 void t1(void)
      9 {
     10     p1 = gp; /* BUGGY!!! */
     11     do_something(p1);
     12     p2 = READ_ONCE(gp);
     13     if (p2) {
     14         do_something_else();
     15         p3 = *gp; /* BUGGY!!! */
     16     }
     17 }
```

`t0()` and `t1()` run concurrently, and `do_something()` and `do_something_else()` are inline functions. Line 1 declares the pointer `gp`, which C initializes to `NULL` by default. At some point, line 5 of `t0()` stores a non-`NULL` pointer to `gp`. Meanwhile, `t1()` loads from `gp` three times on lines 10, 12, and 15\. Given that line 13 finds that `gp` is non-`NULL`, one might hope that the dereference on line 15 would be guaranteed never to fault. 

Unfortunately, the compiler is within its rights to fuse the reads on lines 10 and 15 which means that if line 10 loads `NULL` and line 12 loads `&myvar`, line 15 could dereference `NULL`, resulting in a fault. Note that the intervening `READ_ONCE()` does not prevent the other two loads from being fused, despite the fact that all three are loading from the same variable. It might seem that no real compiler would ever do this, but Will Deacon reports that this has [actually happened in the Linux kernel](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=a7b100953aa3>). 

**Quick quiz 3** : Why does it matter whether `do_something()` and `do_something_else()` are inline functions?   
Answer

Avoid load fusing by either using `READ_ONCE()` for the other accesses to `gp` or by placing at least a Linux kernel `barrier()` between each of these three accesses. 

#### Store fusing

Store fusing can occur when the compiler notices a pair of successive stores to a given variable with no intervening loads from that variable. In this case, the compiler is within its rights to omit the first store. This is never a problem in single-threaded code, and in fact it is usually the case that it is not a problem in correctly written concurrent code. After all, if the two stores are executed in quick succession, there is very little chance that some other thread could load the value from the first store. 

However, there are exceptions, for example as shown below: 

```
1 void shut_it_down(void)
      2 {
      3     status = SHUTTING_DOWN; /* BUGGY!!! */
      4     start_shutdown();
      5     while (!other_task_ready) /* BUGGY!!! */
      6         continue;
      7     finish_shutdown();
      8     status = SHUT_DOWN; /* BUGGY!!! */
      9     do_something_else();
     10 }
     11
     12 void work_until_shut_down(void)
     13 {
     14     while (status != SHUTTING_DOWN) /* BUGGY!!! */
     15         do_more_work();
     16     other_task_ready = 1; /* BUGGY!!! */
     17 }
```

The function `shut_it_down()` stores to the shared variable `status` on lines 3 and 8\. Assuming that neither `start_shutdown()` nor `finish_shutdown()` access `status`, the compiler could reasonably remove the store to `status` on line 3\. Unfortunately, this would mean that `work_until_shut_down()` would never exit its loop spanning lines 14 and 15, and thus would never set `other_task_ready`, which would in turn mean that `shut_it_down()` would never exit its loop spanning lines 5 and 6, even if the compiler chooses not to fuse the successive loads from `other_task_ready` on line 5\. Although `WRITE_ONCE()` prevents store fusing, `smp_store_release()` (or stronger) is often preferable, to ensure that other changes made before the store will be visible to other threads that see the store. 

And there are other problems with that code, including code reordering. 

#### Code reordering

Code reordering is a common compilation technique used to combine common subexpressions, reduce register pressure, and improve utilization of the many functional units available on modern superscalar microprocessors. It is also another reason why the code above is buggy. For example, suppose that the `do_more_work()` function on line 15 does not access `other_task_ready`. Then the compiler would be within its rights to move the assignment to `other_task_ready` on line 16 to precede line 14, which might be a great disappointment for anyone hoping that the last call to `do_more_work()` on line 15 happens before the call to `finish_shutdown()` on line 7\. 

It might seem futile to prevent the compiler from changing the order of accesses in cases where the underlying hardware is free to reorder them. For example, even on a single-CPU machine, what would happen if the hardware reorders two accesses and then an interrupt occurs right in the middle? What values would the interrupt handler see? 

As it turns out, this isn't a problem. Modern machines have "exact exceptions" and "exact interrupts", meaning that any interrupt or exception will appear to have happened at a specific place in the instruction stream. Consequently, the handler will see the effect of all prior instructions, but won't see the effect of any subsequent instructions. `READ_ONCE()`, `WRITE_ONCE()`, and `barrier()` can therefore be used to control communication between interrupted code and interrupt handlers, independent of any reordering carried out by the underlying hardware. That said, should you write user-space code, the various standards committees would prefer that you use atomics or variables of type `sig_atomic_t` instead of `READ_ONCE()` and `WRITE_ONCE()`. 

However, when interacting with some other CPU, stronger primitives are required, such as `smp_load_acquire()` and `smp_store_release()`. 

#### Invented loads

Invented loads are illustrated by the code below, in which the compiler has optimized away a temporary variable from the code shown in the load-tearing example above. 

```
1 /* Optimized code */
      2 if (global_ptr != NULL &&
      3     global_ptr < high_address)
      4         do_low(global_ptr);
```

**Quick quiz 4** : But line 2 specifically checks for `NULL`. So how can `do_low()` possibly be invoked with a `NULL` pointer?   
Answer

This optimization causes `global_ptr` to be loaded three times, which could cause `do_low()` to be invoked with a `NULL` pointer. 

Invented loads can also be a performance hazard. These hazards can occur when a load of variable in a "hot" cacheline is hoisted out of an `if` statement. These hoisting optimizations are not uncommon, and can cause significant increases in cache misses, and thus significant degradation of both performance and scalability. 

Avoid invented loads by using `READ_ONCE()`. 

#### Invented stores

Invented stores can occur in a number of situations. For example, a compiler emitting code for `work_until_shut_down()` in the store-fusing example above might notice that `other_task_ready` is stored to on line 16 and is not accessed by `do_more_work()`. If `do_more_work()` was a complex inline function, it might be necessary to do a register spill, in which case one attractive place to use for temporary storage is `other_task_ready`. After all, there are no accesses to it, so what is the harm? 

Of course, a non-zero store to this variable at just the wrong time would result in the `while` loop on line 5 terminating prematurely, again allowing `finish_shutdown()` to run concurrently with `do_more_work()`. Given that the entire point of this `while` appears to be to prevent such concurrency, this is not a good thing. 

Using a stored-to variable as a temporary might seem outlandish, and we are not aware of any compilers that actually invent stores, but invented stores really are permitted by the standard. Nevertheless, readers might be justified in wanting a less outlandish example, which is duly provided below: 

```
1 if (condition)
      2     a = 1; /* BUGGY!!! */
      3 else
      4     do_a_bunch_of_stuff();
```

A compiler emitting code for this example might know that the value of `a` is initially zero, which might tempt the compiler to optimize away one branch by transforming this code to something like: 

```
1 /* Optimized code */
      2 a = 1;
      3 if (!condition) {
      4     a = 0;
      5     do_a_bunch_of_stuff();
      6 }
```

**Quick quiz 5** : Ouch! So can't the compiler invent a store to a normal variable pretty much any time it likes?   
Answer

Here, line 2 of the optimized version unconditionally stores one to `a`, then resets the value back to zero on line 4 if `condition` was not set. This transforms the if-then-else into an if-then, saving one branch. 

Pre-C11 compilers could invent stores to unrelated variables that happened to be adjacent to written-to variables (see Section 4.2 of Hans Boehm's classic [Threads cannot be implemented as a library](<https://dl.acm.org/citation.cfm?id=1065042>)). This variant of invented stores has been outlawed by the C11 prohibition against compiler optimizations that create data races. 

**Quick quiz 6** : What exactly is a "data race"?   
Answer

Unfortunately, there is an exception to this rule: if there is a later plain store without some sort of ordering directive beforehand, then a data race involving an invented store necessarily implies that there was already a data race involving that later plain store. In this case, the compiler believes that it is not introducing a data race, but rather expanding on an already-existing data race. And the compiler is OK with this, even if your code is not. For example: 

```
1 struct foo {
      2     short a;
      3     char b;
      4     char c;
      5 };
      6
      7 void do_something(struct foo *fp)
      8 {
      9     fp->a = 0x1234;
     10     fp->b = 0x56;
     11     do_something_else();
     12     fp->c = 0x42;
     13 }
```

If the definition of `do_something_else()` is visible to the compiler, and if it contains no ordering directives like `barrier()` or stronger, then the developer's write to `fp->c` tells the compiler that there are no concurrent reads or writes to that field, whether that was the developer's intention or not. The compiler would then be within its rights to do the following optimization (assuming a big-endian system): 

```
1 struct foo {
      2     short a;
      3     char b;
      4     char c;
      5 };
      6
      7 void do_something(struct foo *fp)
      8 {
      9     *(long *)fp = 0x123456ff;
     10     do_something_else();
     11     fp->c = 0x42;
     12 }
```

The momentary appearance of `0xff` might come as quite a surprise to any concurrent loads from `fp->c`. Please note that this is not a theoretical transformation: A later store to a variable is taken as permission to clobber that variable. In addition, older compilers can and do invent stores to unrelated variables, even without the provocation of a later plain C-language store to such an unrelated variable. Use `barrier()` or `WRITE_ONCE()` to avoid all of these types of invented stores. 

####  Store-to-load transformations

Store-to-load transformations can occur when the compiler notices that a plain C-language store might not actually change the value in memory. For example, consider this code: 

```
1 int r1, x, y;
      2
      3 void cpu1(void)
      4 {
      5     WRITE_ONCE(y, 1);
      6     smp_mb();
      7     WRITE_ONCE(x, 1);
      8 }
      9
     10 void cpu2(void)
     11 {
     12     r1 = READ_ONCE(x);
     13     if (r1 == 1)
     14         y = 0; // BUGGY!!!
     15 }
```

Here CPU 1 executes `cpu1()`, which uses `WRITE_ONCE()` to store the value one to each of `y` and then `x`, separated by a full memory barrier. CPU 2 executes `cpu2()`, which uses `READ_ONCE()` to load `x`, and only if the result is 1, line 14 does a plain store of zero to `y`. One would expect that if `r1` ends up with the value one, that the final value of `y` must necessarily be zero. 

Unfortunately, the compiler is within its rights to transform line 14 into the load-compare-store sequence shown on lines 14 and 15 below: 

```
1 int r1, x, y;
      2
      3 void cpu1(void)
      4 {
      5     WRITE_ONCE(y, 1);
      6     smp_mb();
      7     WRITE_ONCE(x, 1);
      8 }
      9
     10 void cpu2(void)
     11 {
     12     r1 = READ_ONCE(x);
     13     if (r1 == 1)
     14         if (y != 0)
     15             y = 0;
     16 }
```

Given this code, CPU 2 may reorder the load of `y` on line 14 before the `READ_ONCE`. If it does so, it might observe the original zero value of `y` and therefore skip the store on line 15\. Thus `y` could indeed end up containing one. 

Why would the compiler do such a thing? Please understand that to the best of our knowledge, this transformation is strictly theoretical. However, it does not take too much imagination to see how this might occur given feedback-driven optimization. So if you want your store to remain a store, for current and any future compilers, use `WRITE_ONCE()` or stronger. 

#### Dead-code elimination

Dead-code elimination can occur when the compiler notices that the value from a load is never used, or when a variable is stored to, but never loaded from. This can of course eliminate an access to a shared variable, which can in turn defeat a memory-ordering primitive, which could cause your concurrent code to act in surprising ways. Experience thus far indicates that relatively few such surprises will be at all pleasant. Elimination of store-only variables is especially dangerous in cases where external code locates the variable via symbol tables; the compiler is necessarily ignorant of such external-code accesses, and might thus eliminate a variable that the external code relies on. 

Reliable concurrent code clearly needs a way to cause the compiler to preserve the number, order, and type of important accesses to shared memory, which is why the Linux kernel provides `READ_ONCE()`, `WRITE_ONCE()`, `barrier()`, and a wide variety of memory barriers and atomic read-modify-write operations. 

#### How real is all this?

Some of the transformations called out in the preceding sections are more likely to actually occur than are others.  | Occurs in the Wild?  
---|---  
Transformation | (Properly Aligned, Machine-Word Sized)  
Load Tearing | [Yes](<http://lkml.kernel.org/r/CAHk-=wj2t+GK+DGQ7Xy6U7zMf72e7Jkxn4_-kGyfH3WFEoH+YQ@mail.gmail.com>)  
Store Tearing | Yes, for constants (to be fixed?)  
Load Fusing | Yes  
Store Fusing | Yes  
Code Reordering | Yes  
Invented Loads | Yes  
Invented Stores | In some cases  
Store-to-Load Transformations | Unknown  
Dead-Code Elimination | Yes  
  
So what is a Linux kernel developer to do? There is a range of possibilities, each of which applies `READ_ONCE()` and `WRITE_ONCE()` in different situations: 

  * Never. 
  * For any access to any shared variable for which there is a possibility of a data race, and for which it can be clearly shown that specific compiler optimizations could result in bugs. 
  * For any access to a shared variable for which there is a possibility of a data race for at least one of those accesses. 
  * For all accesses to all shared variables. 

There is without doubt some code somewhere in the Linux kernel corresponding to each of these possibilities. However, developers and maintainers opting for one of the first two possibilities are taking on the responsibility of ensuring that new releases of the compiler won't break their code. For these developers and maintainers, a significant level of fear of the big bad optimizing compiler is a very healthy thing, and the rest of us should hope that they continue to maintain an appropriate level of fear. 

**Quick quiz 7** : This paper has covered all of the transformations that an optimizing compiler can carry out, right?   
Answer

**Quick quiz 8** : Given the risk, why not simply require that all accesses to shared variables use `READ_ONCE()` and `WRITE_ONCE()`?   
Answer

On the other hand, developers and maintainers who instead opt for one of the last two possibilities need not fear the big bad optimizing compiler, or at least they need not fear it quite so much. However, they could benefit from a tool that determines when `READ_ONCE()` and `WRITE_ONCE()` (or stronger) are needed to defend not just against present-day optimizing compilers, but also the bigger and badder optimizing compilers that the future will bring. The next article in this series describes a recent change to the Linux kernel memory model that does just that. 

#### Acknowledgments

We owe thanks to a surprisingly large number of compiler writers and members of the C and C++ standards committees who introduced us to some of the things a big bad optimizing compiler can do, and to Junchang Wang, SeongJae Park, and Slavomir Kaslev for their help making an earlier draft of this material human-readable. We are also grateful to Mark Figley and Kara Todd for their support of this effort. 

####  Answers to quick quizzes

**Quick quiz 1** : But we shouldn't be afraid at all for things like on-stack or per-CPU variables, right? 

**Answer** : Although on-stack and per-CPU variables are often guaranteed to be untouched by other CPUs and tasks, the kernel really does allow them to be concurrently accessed in many cases. You do have to go out of your way to make this happen, say by explicitly passing the address of such a variable to another thread, but it's certainly not impossible. 

For example, the [`_wait_rcu_gp()`](<https://elixir.bootlin.com/linux/latest/source/include/linux/rcupdate_wait.h#L24>) macro uses an on-stack `__rs_array[]` array of `rcu_synchronize` structures that, in turn, contain `rcu_head` and `completion` structures. The address of the `rcu_head` structure is passed to `call_rcu()`, which results in concurrent accesses to this structure, and eventually also to the `completion` structure. 

Similar access patterns may be found for per-CPU variables. 

**Back to quick quiz 1**.

**Quick quiz 2** : But there are lots of plain loads from shared variables in the Linux kernel. These cannot possibly all be buggy, can they? 

**Answer** : This turns out to be a matter of the context in which these plain loads execute and just how vigilant the developers and maintainers wish to be. 

Starting with context, if a given variable is only ever accessed under the protection of a given exclusive lock or mutex, then use of plain loads (and stores, for that matter) is perfectly safe. 

Less restrictive contexts also suffice. If stores to a given variable can never execute concurrently with any other accesses to that variable, then use of plain loads and stores is again perfectly safe. For example, if all loads from a given variable are under the protection of a reader-writer lock or mutex, and if all stores to that same variable are under the write-side protection of that same reader-writer lock or mutex, use of plain loads and stores is perfectly safe. Alternatively, if all stores to a given variable are carried out by a given kernel thread, and that same variable is only ever loaded by subsequently spawned child threads, plain loads and stores are yet again perfectly safe. Similarly, if all stores to a given structure happen before it is made visible to readers via `rcu_assign_pointer()`, and the readers, having gained access via `rcu_dereference()`, only ever load from that structure, plain loads and stores are once more perfectly safe. There are numerous additional variations on this theme. 

But there is no shortage of plain loads in the kernel that really can execute concurrently with stores to that same variable, which brings us to vigilance. For example, if the variable only ever transitions from zero to one, no matter how the compiler [dices and slices](<http://lkml.kernel.org/r/20190606061438.nyzaeppdbqjt3jbp@gondor.apana.org.au>) the load, the result will be either a zero or a one. Give or take the possibility of invented loads, which could get the effect of both a zero and a one being loaded, though if the value loaded is only used once, one would hope that this confusing possibility would be avoided. 

In other words, when using plain loads from shared variables, it is the developers' and maintainers' responsibility to either prevent concurrent stores to that same variable on the one hand or to ensure that the compiler cannot optimize their algorithms out of existence on the other. 

So are the Linux kernel's plain loads from shared variables buggy? If the relevant developers and maintainers are either carefully controlling the contexts from which those variables are accessed on the one hand or vigilantly considering what optimizing compilers can do to their code on the other, perhaps not! 

**Back to quick quiz 2**.

**Quick quiz 3** : Why does it matter whether `do_something()` and `do_something_else()` are inline functions? 

**Answer** : Because `gp` is not a static variable, if either `do_something()` or `do_something_else()` were separately compiled, the compiler would have to assume that either or both of these two functions might change the value of `gp`. This possibility would force the compiler to reload `gp` on line 15, thus avoiding the `NULL`-pointer dereference. 

In the absence of [link-time](<https://en.wikipedia.org/wiki/Interprocedural_optimization>) [optimizations](<https://gcc.gnu.org/wiki/LinkTimeOptimization>) [(LTO)](<https://llvm.org/docs/LinkTimeOptimization.html>), that is. As optimizing compilers become more aggressive, developers and maintainers must become aggressive about disabling destructive optimizations, whether that be via command-line arguments to the compiler or via source-code decorations such as `barrier()`, `READ_ONCE()`, and `WRITE_ONCE()`. 

**Back to quick quiz 3**.

**Quick quiz 4** : But line 2 specifically checks for `NULL`. So how can `do_low()` possibly be invoked with a `NULL` pointer? 

**Answer** : Imagine the following sequence of events: 

  1. Line 2 loads a non-`NULL` pointer from `global_ptr`. 
  2. Some other CPU stores `NULL` to `global_ptr`. 
  3. Line 3 loads the newly stored `NULL` from `global_ptr`, and this compares less than `high_address`. 
  4. Surprise! There is now a call to `do_low()` with a `NULL` pointer. 

**Back to quick quiz 4**.

**Quick quiz 5** : Ouch! So can't the compiler invent a store to a normal variable pretty much any time it likes? 

**Answer** : Thankfully, the answer is no. This is because the compiler is forbidden from introducing data races. The case of inventing a store just before a normal store is quite special: It is not possible for some other entity, be it CPU, thread, signal handler, or interrupt handler, to be able to see the invented store unless the code already has a data race, even without the invented store. And if the code already has a data race, it already invokes the dreaded specter of undefined behavior, which allows the compiler to generate pretty much whatever code it wants, regardless of the wishes of the developer. 

But if the original store is volatile, as in `WRITE_ONCE()`, for all the compiler knows, there might be a side effect associated with the store that could signal some other thread, allowing data-race-free access to the variable. By inventing the store, the compiler might be introducing a data race, which it is not permitted to do. And this is one reason why [`memory-barriers.txt`](<https://github.com/torvalds/linux/blob/master/Documentation/memory-barriers.txt>) requires `WRITE_ONCE()` for stores that are to be ordered by control dependencies. Another reason may be gleaned from the Store-to-Load Transformations section. 

In the case of `volatile` and atomic variables, the compiler is specifically forbidden from inventing writes. 

**Back to quick quiz 5**.

**Quick quiz 6** : What exactly is a "data race"? 

**Answer** : A _data race_ occurs when there are multiple concurrent accesses to a given variable, at least one of which is a plain C-language access and at least one of which is a store. 

**Back to quick quiz 6**.

**Quick quiz 7** : This paper has covered all of the transformations that an optimizing compiler can carry out, right? 

**Answer** : Wrong. 

There are a great many more, which should not be a surprise given the large number of situations where the C standard specifies undefined behavior, each of which potentially points the way to interesting compiler optimizations. There are some efforts under way to rein in compiler optimizations to at least some extent (for example, [here](<http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2019/p1797r0.html>), [here [PDF]](<http://www.open-std.org/jtc1/sc22/wg14/www/docs/n2369.pdf>), and [here [PDF]](<http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2019/p1756r0.pdf>)), but compiler developers and standards-committee members are not necessarily as supportive of such efforts as might be hoped by maintainers and developers working with concurrent code. 

**Back to quick quiz 7**.

**Quick quiz 8** : Given the risk, why not simply require that all accesses to shared variables use `READ_ONCE()` and `WRITE_ONCE()`? 

**Answer** : One can certainly argue that they should be used more heavily than they currently are, but it is not all that hard to get too much of a good thing. For example, as mentioned in the answer to an earlier quick quiz, any number of in-kernel mechanisms, perhaps most notably locking, can provide mutual exclusion so that `READ_ONCE()` and `WRITE_ONCE()` are not needed. 

In addition, although `READ_ONCE()` and `WRITE_ONCE()` are low cost, they are not free due to the fact that they constrain compiler optimizations. For example, the compiler is required to emit the accesses for a pair of consecutive `READ_ONCE()` invocations in order, and it might well be just fine (and perhaps also cheaper) for those to invocations to be reordered. Some fast paths might therefore need plain C-language accesses, though one would hope that the developers and maintainers would see fit to take pity on people reading their code by providing appropriate comments. 

And there are guarantees that the Linux kernel relies on that are provided by usage restrictions rather than by compiler directives. Examples include address and data dependencies, for which the usage restrictions are documented in [rcu_dereference.txt](<https://github.com/torvalds/linux/blob/master/Documentation/RCU/rcu_dereference.txt>) as well as control dependencies, for which the usage restrictions are documented in the `CONTROL DEPENDENCIES` section of [memory-barriers.txt](<https://github.com/torvalds/linux/blob/master/Documentation/memory-barriers.txt>). 

However, we should continue to expect increasingly aggressive compiler optimizations over time. This will likely increase the development and maintenance burden incurred by those making use of plain C-language loads and stores to shared variables in cases where data races exist. This prospect might help explain why the use of things like `READ_ONCE()` and `WRITE_ONCE()` has been increasing steadily within the Linux kernel. 

**Back to quick quiz 8**.
