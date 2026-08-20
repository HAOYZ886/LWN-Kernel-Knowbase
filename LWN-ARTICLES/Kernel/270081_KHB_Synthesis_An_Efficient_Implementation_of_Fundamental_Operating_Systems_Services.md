---
title: "KHB: Synthesis: An Efficient Implementation of Fundamental Operating Systems Services"
url: https://lwn.net/Articles/270081/
date: "February 20, 2008"
category: Kernel Hackers Bookshelf
author: "February 20, 2008 This article was contributed by Valerie Aurora"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

February 20, 2008

This article was contributed by Valerie Aurora

When I was but a wee computer science student at New Mexico Tech, a graduate student in OS handed me an inch-thick print-out and told me that if I was really interested in operating systems, I had to read this. It was something about a completely lock-free operating system optimized using run-time code generation, written from scratch in assembly running on a homemade two-CPU SMP with a two-word compare-and-swap instruction - you know, nothing fancy. The print-out I was holding was Alexia Massalin's PhD thesis, [Synthesis: An Efficient Implementation of Fundamental Operating Systems Services](<http://citeseer.ist.psu.edu/massalin92synthesi.html>) ([html version here](<http://valhenson.org/synthesis/SynthesisOS/>)). Dutifully, I read the entire 158 pages. At the end, I realized that I understood not a word of it, right up to and including the cartoon of a koala saying "QUA!" at the end. Okay, I exaggerate - lock-free algorithms had been a hobby of mine for the previous few months - but the main point I came away with was that there was a lot of cool stuff in operating systems that I had yet to learn. 

Every year or two after that, I'd pick up my now bedraggled copy of "Synthesis" and reread it, and every time I would understand a little bit more. First came the lock-free algorithms, then the run-time code generation, then quajects. The individual techniques were not always new in and of themselves, but in Synthesis they were developed, elaborated, and implemented throughout a fully functioning UNIX-style operating system. I still don't understand all of Synthesis, but I understand enough now to realize that my grad student friend was right: anyone really interested in operating systems should read this thesis. 

### Run-time code generation

The name "Synthesis" comes from run-time code generation - code synthesis - used to optimize and re-optimize kernel routines in response to changing conditions. The concept of optimizing code during run-time is by now familiar to many programmers in part from Transmeta's processor-level code optimization, used to lower power consumption (and many programmers are familiar with Transmeta as the one-time employer of Linus Torvalds.) 

Run-time code generation in Synthesis begins with some level of compile-time optimization, optimizations that will be efficient regardless of the run-time environment. The result can thought of as a template for the final code, with "holes" where the run-time data will go. The run-time code generation then takes advantage of data-dependent optimizations. For example, if the code evaluates A * B, and at run-time we discover that B is always 1, then we can generate more efficient code that skips the multiplication step and run that code instead of the original. Fully optimized versions of the code pre-computed for common data values can be simply swapped in without any further run-time computation. Another example from the thesis: 

[...] Suppose that the compiler knows, either through static control-flow analysis, or simply by the programmer telling it through some directives, that the function _f_(p1, ...) = 4 * p1 + ... will be specialized at run-time for constant p1. The compiler can deduce that the expression 4 * p1 will reduce to a constant, but it does not know what particular value that constant will have. It can capture this knowledge in a custom code generator for _f_ that computes the value 4 * p1 when p1 becomes known and stores it in the correct spot in the machine code of the specialized function _f_ , bypassing the need for analysis at run-time. 

Run-time code generation in Synthesis is a fusion of compile-time and run-time optimizations in which useful code templates are created at compile time that can later be optimized simply and cleanly at run time. 

### Quajects

Understanding run-time code generation is a prerequisite for understanding quajects, the basic unit out of which the Synthesis kernel is constructed. Quajects are almost but not quite entirely unlike objects. Like objects, quajects come in types - queue quaject, thread quaject, buffer quaject - and encapsulate all the data associated with the quaject. Unlike objects, which contain pointers to functions implementing their methods, quajects contain the code implementing their methods directly. That's right - the actual executable instructions are stored inside the data structure of the quaject, with the code nestled up against the data it will operate on. In cases where the code is too large to fit in the quaject, the code jumps out to the rest of the method located elsewhere in memory. The code implementing the methods is created by filling in pre-compiled templates and can be self-modifying as well. 

Quajects interact with other quajects via a direct and simple system of cross-quaject calls: callentries, callouts, and callbacks. The user of quaject invokes callentries in the quaject, which implement that quaject's methods. Usually the callentry returns back to the caller as normal, but in exceptional situations the quaject will invoke a method in the caller's quaject - a callback. Callouts are places where a quaject invokes some other quaject's callentries. 

Synthesis implements a basic set of quajects - thread, queue, buffer, clock, etc. - and builds higher-level structures by combining lower-level quajects. For example, a UNIX process is constructed out of a thread quaject, a memory quaject, and some I/O quajects. 

As an example, let's look at the queue quaject's interface. A queue has two callentries, `queue_put` and `queue_get`. These are invoked by another quaject wanting to add or remove entries to and from the queue. The queue quaject also has four callbacks into the caller's quaject, `queue_full`, `queue_full-1`, `queue_empty`, and `queue_empty-1`. When a caller invokes the `queue_put` method and the queue is full, the queue quaject invokes the `queue_full` callback in the caller's quaject. From the thesis: 

The idea is: instead of returning a condition code for interpretation by the invoker, the queue quaject directly calls the appropriate handling routines supplied by the invoker, speeding execution by eliminating the interpretation of return status codes. 

The `queue_full-1` method is executed when a queue has transitioned from full to not full, queue_empty when the queue doesn't contain anything, and `queue_empty-1` when the queue transitions from empty to not empty. With these six callentries and callbacks, a queue is implemented in a generic, extensible, yet incredibly efficient manner. 

Pretty cool stuff, huh? But wait, there's more! 

### Optimistic lock-free synchronization

Most modern operating systems use a combination of interrupt disabling and locks to synchronize access to shared data structures and guarantee single-threaded execution of critical sections in general. The most popular synchronization primitive in Linux is the spinlock, implemented with the nearly universal test-and-set-bit atomic operation. When one thread attempts to acquire the spinlock guarding some critical section, it busy-waits, repeatedly trying to acquire the spinlock until it succeeds. 

Synchronization based on locks works well enough but it has several problems: contention, deadlock, and priority inversion. Each of these problems can be (and is) worked around by following strict rules: keep the critical section short, always acquire locks in the same order, and implement various more-or-less complex methods of priority inheritance. Defining, implementing, and following these rules is non-trivial and a source of a lot of the pain involved in writing code for modern operating systems. 

To address these problems, Maurice Herlihy proposed a system of lock-free synchronization using an atomic compare-and-swap instruction. Compare-and-swap takes the address of a word, the previous value of the word, and the desired new value of the word. It swaps the previous and new values of the word if and only if the previous value is the same as the current value. The bare compare-and-swap instruction allows atomic updates of single pointers. To atomically switch between larger data structures, a new copy of the data structure is created, updated with the changes, and the addresses of the two data structures swapped. If the compare-and-swap fails because some other thread has updated the value, the operation is retried until it succeeds. 

Lock-free synchronization eliminates deadlocks, the need for strict lock ordering rules, and priority inversion (contention on the compare-and-swap instruction itself is still a concern, but rarely observed in the wild). The main drawback of Herlihy's algorithms is that they require a lot of data copying for anything more complex than swapping two addresses, making the total cost of the operation greater than the cost of locking algorithms in many cases. Massalin took advantage of the two-word compare-and-swap instruction available in the Motorola 68030 and expanded on Herlihy's work to implement lock-free and copy-free synchronization of queues, stacks, and linked lists. She then took a novel approach: Rather than choose a general synchronization technique (like spinlocks) and apply it to arbitrary data structures and operations, instead build the operating system out of data structures simple enough to be updated in an efficient lock-free manner. 

Synthesis is actually even cooler than lock-free: Given the system of quajects, code synthesis, and callbacks, operations on data structures can be completely synchronization-free in common situations. For example, a single-producer, single-consumer queue can be updated concurrently without any kind of synchronization as long as the queue is non-empty, since each thread operates on only one end of the queue. When the callback for queue empty happens, the code to operate on the queue is switched to use the lock-free synchronization code. When the quaject's queue-not-empty callback is invoked, the quajects switch back to the synchronization-free code. (This specific algorithm is not, to my knowledge, described in detail in the thesis, but was imparted to me some months ago by Dr. Massalin herself at one of those wild late-night kernel programmer parties, so take my description with a grain of salt.) 

The approach to synchronization in Synthesis is summarized in the following quote: 

  * Avoid synchronization whenever possible. 
  * Encode shared data into one or two machine words. 
  * Express the operation in terms of one or more fast lock-free data structure operations. 
  * Partition the work into two parts: a part that can be done lock-free, and a part that can be postponed to a time when there can be no interference. 
  * Use a server thread to serialize the operation. Communications with the server happens using concurrent, lock-free queues. 

The last two points will sound familiar if you're aware of [Paul McKenney's read-copy-update (RCU)](<http://www.rdrop.com/users/paulmck/RCU/>) algorithm. In Synthesis, thread structures to be deleted or removed from the run queue are marked as such, and then actually deleted or removed by the scheduler thread during normal traversal of the run queue. In RCU, the reference to a list entry is removed from the linked list while holding the list lock, but the removed list entry is not actually freed until it can be guaranteed that no reader is accessing that entry. In both cases, reads are synchronization-free, but deletes are separated into two phases, one that begins the operation in an efficient low-contention manner, and a second, deferred, synchronization-free phase to complete the operation. The two techniques are by no means the same, but share a similar philosophy. 

### Synthesis: Operating system of the future?

The design principles of Synthesis, while powerful and generic, still have some major drawbacks. The algorithms are difficult to understand and implement for regular human beings (or kernel programmers, for that matter). As Linux has demonstrated, making kernel development simple enough that a wide variety of people can contribute has some significant payoffs. Another drawback is that two-word compare-and-swap is, shall we say, not a common feature of modern processors. Lock-free synchronization can be achieved without this instruction, but it is far less efficient. In my opinion, reading this paper is valuable more for retraining the way your brain thinks about synchronization than for copying the exact algorithms. This thesis is especially valuable reading for people interested in low-latency or real-time response, since one of the explicit goals of Synthesis is support for real-time sound processing. 

Finally, I want to note that Synthesis contains many more elegant ideas that I couldn't cover in even the most superficial detail - quaject-based user/kernel interface, per-process exception tables, scheduling based on I/O rates, etc., etc. And while the exact implementation details are fascinating, the thesis is also peppered with delightful koan-like statements about design patterns for operating systems. Any time you're feeling bored with operating systems, sit down and read a chapter of this thesis. 

[ Valerie Henson is a [Linux file systems consultant](<http://vahconsulting.com>) and proud recipient of a piggy-back ride from Dr. Alexia Massalin. ]
