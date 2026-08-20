---
title: Protecting control dependencies with volatile_if()
url: https://lwn.net/Articles/860037/
date: "June 18, 2021"
category: Memory barriers
author: "By Jonathan Corbet June 18, 2021"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
June 18, 2021

Memory ordering issues are, as Linus Torvalds [recently observed](<https://lwn.net/ml/linux-kernel/CAHk-%3DwgJZVjdZYO7iNb0hFz-iynrEBcxNcT8_u317J0-nzv59w%40mail.gmail.com/>), ""the rocket science of CS"". Understanding memory ordering is increasingly necessary to write scalable code, so kernel developers often find themselves having to become rocket scientists. The subtleties associated with control dependencies turn out to be an especially tricky sort of rocket. A recent discussion about how to force control dependencies to be observed shows the sorts of difficulties that arise in this area. 

#### Control dependencies

The C programming language was designed in the era of simple, uniprocessor computers. When a developer wrote a series of C statements, they could expect those statements to be executed in the order written. Decades later, though, the situation has become much more complicated; code can be extensively optimized by both compilers and CPUs to the point that it bears little resemblance to what was originally written. Code can be reordered and even eliminated if the compiler (or the processor) thinks that the end result will be the same. The effects of this reordering on single-threaded code are (in the absence of bugs) limited to making it run faster. When there are multiple threads of execution running simultaneously, though, there can be surprises in store. One thread may observe things happening in a different order than others, leading to all sorts of unfortunate confusion. 

When the visible order of operations across processors is important, developers will often use barriers to ensure that operations are not reordered in damaging ways. There are, however, cases where developers can count on things happening in the right order because there is no alternative; these are described in terms of "dependencies". There are three broad classes of dependencies, described in [this article](<https://lwn.net/Articles/850202/>) from our recent [lockless patterns series](<https://lwn.net/Articles/844224/>). Consider, for example, a simple data dependency: 

```
int x = READ_ONCE(a);
        WRITE_ONCE(b, x + 1);
```

The write to `b` simply cannot be reordered ahead of the read of `a` because neither the compiler nor the CPU knows what value should be written. The write has a data dependency on the preceding read; that dependency will prevent those two operations from being reordered. That, of course, assumes that the compiler does not conclude that it already knows what the value of `a` will be, perhaps from a previous read; that is why `READ_ONCE()` is used. The [second article](<https://lwn.net/Articles/846700/>) in the lockless patterns series describes `READ_ONCE()` and `WRITE_ONCE()` in detail. 

Control dependencies are a bit more complex. Consider code like this: 

```
if (READ_ONCE(a))
        	WRITE_ONCE(b, 1);
```

There is no data dependency linking the read of `a` and the write to `b`, but that write can only occur if `a` has a non-zero value; the read of `a` must thus occur before the write. This ordering forced by a conditional branch is a control dependency. More generally, there are three things that must be present to establish a control dependency: 

  * A read from one location (`a` in the case above) 
  * A conditional branch that depends on the value that was read 
  * A write to another location in one or more branches 

When those conditions exist, there is a control dependency from the read to the write that prevents the two operations from being reordered with respect to each other. 

#### The evil optimizing compiler

Or, at least, it would be nice if things worked that way. The problem is that, while the _hardware_ works that way, the C language does not recognize the existence of control dependencies or, as the infamous kernel [`memory-barriers.txt` document](<https://www.kernel.org/doc/Documentation/memory-barriers.txt>) puts it: ""Compilers do not understand control dependencies. It is therefore your job to ensure that they do not break your code."" While there does not appear to be much of a history of code being broken through overly aggressive optimization of code with control dependencies, it is something that developers worry about. That has led to the [proposal](<https://lwn.net/ml/linux-kernel/YLn8dzbNwvqrqqp5@hirez.programming.kicks-ass.net/>) by Peter Zijlstra of a mechanism called `volatile_if()`. 

What sort of problem is this patch trying to address? Consider an example [posted](<https://lwn.net/ml/linux-kernel/20210604214010.GD4397@paulmck-ThinkPad-P17-Gen-1/>) by Paul McKenney in the discussion: 

```
if (READ_ONCE(A)) {
    	WRITE_ONCE(B, 1);
    	do_something();
        } else {
    	WRITE_ONCE(B, 1);
    	do_something_else();
        }
```

This code has a control dependency between the read of `A` and the writes to `B`; each write is in a branch of the conditional statement and the fact that they write the same value does not affect the dependency. So one might conclude that the two operations could not be reordered. Compilers, though, might well rearrange the code to look like this instead: 

```
tmp = READ_ONCE(A);
        WRITE_ONCE(B, 1);
        if (tmp)
    	do_something();
        else
    	do_something_else();
```

This code looks equivalent, but the test on the value read from `A` no longer occurs before the write to `B`. That breaks the control dependency, freeing a sufficiently aggressive CPU to move the write ahead of the read, possibly creating a subtle and unpleasant bug. 

Since C doesn't recognize control dependencies, avoiding this kind of bug can be difficult, even in cases where the developer is aware of the problem. One sure solution is to read `A` with acquire semantics and write `B` with release semantics, as described in the lockless patterns series, but acquire and release operations can be expensive on some architectures. That expense is not usually needed in this case. 

#### volatile_if()

Zijlstra wrote in his proposal that a good solution would be to add a qualifier to the `if` statement to indicate that a dependency exists: 

```
volatile if (READ_ONCE(A)) {
        	/* ... */
```

The compiler would respond by ensuring that a conditional branch is emitted and that code from within the branches is not lifted out of those branches. That, however, requires cooperation from compiler writers; as Segher Boessenkool [noted](<https://lwn.net/ml/linux-kernel/20210604153518.GD18427@gate.crashing.org/>), that is unlikely to happen unless the standards committee gives its blessing to the idea of putting qualifiers like `volatile` on statements. Failing that, Zijlstra proposed a magic macro: 

```
volatile_if(condition) {
        	/* true case */
        } else {
            /* false case */
        }
```

He provided implementations for a number of architectures; these generally depend on hand-written assembly code to manually emit the conditional branch instruction needed to create the control dependency at the CPU level. 

The resulting discussion focused on two main topics: the implementation of `volatile_if()` and whether it is needed at all. On the implementation side, Torvalds [suggested](<https://lwn.net/ml/linux-kernel/CAHk-=wievFk29DZgFLEFpH9yuZ0jfJqppLTJnOMvhe=+tDqgrw@mail.gmail.com/>) a simpler approach: 

```
#define barrier_true() ({ barrier(); 1; })
        #define volatile_if(x) if ((x) && barrier_true())
```

The `barrier()` macro causes no code to be emitted; it is just an empty block presented to the compiler as assembly code. That keeps the compiler from reordering operations from one side of the barrier to the other; it also, Torvalds said, would force the compiler to emit the branch since it could only be evaluated on the "true" side of the branch. Life turned out to not be so simple, though; a redefinition of `barrier()` along the lines [suggested](<https://lwn.net/ml/linux-kernel/20210606185922.GF7746@tucnak/>) by Jakub Jelinek would be required to make this scheme actually work. 

But Torvalds also [wondered](<https://lwn.net/ml/linux-kernel/CAHk-=wiuLpmOGJyB385UyQioWMVKT6wN9UtyVXzt48AZittCKg@mail.gmail.com/>) why developers were worried about this problem in the first place, since he does not think it can manifest in real code: 

> Again, semantics do matter, and I don't see how the compiler could actually break the fundamental issue of "load->conditional->store is a fundamental ordering even without memory barriers because of basic causality", because you can't just arbitrarily generate speculative stores that would be visible to others. 

And, indeed, evidence of such problems actually occurring is hard to find. He did eventually [come around](<https://lwn.net/ml/linux-kernel/CAHk-=whDrTbYT6Y=9+XUuSd5EAHWtB9NBUvQLMFxooHjxtzEGA@mail.gmail.com/>) to seeing that a problem could potentially exist but also made it clear that he doesn't think there is any code in the kernel now that would be affected by it. 

The conversation (eventually) wound down without coming to any real conclusion on whether `volatile_if()` is needed or not. Experience says, though, that wariness toward compiler optimizations is usually a good idea. Even if no mechanism for explicitly marking control dependencies is merged into the mainline now, it will be waiting in the wings should future compiler releases create problems.
