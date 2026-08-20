---
title: An introduction to lockless algorithms
url: https://lwn.net/Articles/844224/
date: "February 19, 2021"
category: Lockless algorithms
author: "February 19, 2021 This article was contributed by Paolo Bonzini"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

February 19, 2021

This article was contributed by Paolo Bonzini

Lockless algorithms are of interest for the Linux kernel when traditional locking primitives either cannot be used or are not performant enough. For this reason they come up every now and then on LWN; one of the last mentions, which prompted me to write this article series, was [last July](<https://lwn.net/Articles/827180/>). Topics that arise even more frequently are read-copy-update (RCU — [these articles from 2007](<https://lwn.net/Articles/262464/>) are still highly relevant), [reference counting](<https://lwn.net/Articles/575460/>), and ways of wrapping lockless primitives into higher-level, more easily understood APIs. These articles will delve into the concepts behind lockless algorithms and how they are used in the kernel. 

**Other articles in this series** : 

  * [Relaxed access and partial memory barriers](<https://lwn.net/Articles/846700/>)
  * [Full memory barriers](<https://lwn.net/Articles/847481/>)
  * [Compare-and-swap](<https://lwn.net/Articles/847973/>)
  * [More read-modify-write operations](<https://lwn.net/Articles/849237/>)
  * [Some final topics](<https://lwn.net/Articles/850202/>) 

Low-level knowledge of the memory model is universally recognized as advanced material that can scare even the most seasoned kernel hackers; our editor wrote (in the July article) that ""it takes a special kind of mind to really understand the memory model"". It's been [said](<https://lwn.net/Articles/576642/>) that the Linux kernel memory model (and in particular [`Documentation/memory-barriers.txt`](<https://elixir.bootlin.com/linux/latest/source/Documentation/memory-barriers.txt>)) can be used to frighten small children, and the same is probably true of just the words "acquire" and "release". 

At the same time, mechanisms like RCU and [seqlocks](<https://lwn.net/Articles/22818/>) are in such widespread use in the kernel that almost every developer will sooner or later encounter fundamentally lockless programming interfaces. For this reason, it is a good idea to equip yourself with at least a basic understanding of lockless primitives. Throughout this series I will describe what acquire and release semantics are really about, and present five relatively simple patterns that alone can cover most uses of the primitives. 

#### Acquire, release, and "happens before"

In order to make the big step from the (relative) comfort of synchronization primitives to lockless programming, we shall first take a look at why locks work in the first place. Usually, this is taught in terms of mutual exclusion: locks prevent multiple threads of execution from reading or writing the same data concurrently. But what does "concurrently" really mean? And what happens when thread T is done with that data and thread U starts using it? 

In order to answer these questions, we can turn to a theoretical framework that Leslie Lamport established in his 1978 paper ["Time, Clocks and the Ordering of Events in a Distributed System"](<http://research.microsoft.com/en-us/um/people/lamport/pubs/time-clocks.pdf>). According to the paper, the events in a distributed system can be ordered according to whether an event P _happens before_ another event Q: 

  * The ordering of events is _total_ within a single thread of execution. In layman's terms, for any two events from the same thread you can always say which came first and which came second. 
  * If two events do not happen within a single thread of execution, event P happens before event Q if event P is a message send and event Q is the corresponding message receive. 
  * The relation is _transitive_. Therefore, if event P happens before event Q and event Q happens before event R, then event P happens before R. 

The "happens before" relation is a _partial_ ordering: it is possible to have two events P and Q such that neither happens before the other. When this happens, the two events are _concurrent_. Remember how a lock prevents concurrent accesses to the same data structure? That is because, when you protect a data structure with a lock, all accesses to the data structure form a total ordering, as if they came from a single thread. Lamport's framework also provides a basic idea of what happens when a lock is handed off from a thread to another: some kind of "message passing" ensures that the unlock operation of thread T "happens before" the lock operation of thread U. 

As it turns out, this is not just theory: in order to ensure that their caches are coherent, CPUs exchange messages over buses such as Intel's QPI or AMD's HyperTransport. However, this level of detail is definitely too much for our purposes. Instead, we will generalize the "happens before" definition to cover any kind of synchronization primitive. 

Lamport's fundamental insight was that synchronization happens when two threads operate with symmetric operations on the same data structure. In our generalization, we will list pairs of operations (such as sending and receiving a message through the same queue) that synchronize one thread with another. Furthermore, we will classify the operations in each pair as either _release_ or _acquire_ ; equivalently we can say that they "have release (or acquire) semantics". 

Within a pair, the release operation _synchronizes with_ the corresponding acquire operation. "Synchronizing" means that, whenever one thread performs a release operation and another thread performs the corresponding acquire operation, the "happens before" relation grows edges from the releasing thread to the acquiring thread. "Happens before" remains a partial order in the general case, but it now spans two threads, or more than two, thanks to transitivity. More formally: 

  * The ordering of operations is total within a single thread. 
  * If an operation P with release semantics synchronizes with an operation Q with acquire semantics, then operation P happens before operation Q, even if they happen in different threads. 
  * Just like before, the relation is transitive and defines a partial ordering of operations. 

The old definition follows by giving release semantics to message send and acquire semantics to message receive. A message send synchronizes the sending thread with the thread (or threads) that receives the message. We can also rephrase our previous findings according to this new definition: for locks to work, unlocking must have release semantics and must synchronize with locking--which in turn has acquire semantics. No matter if the lock is contended or not, the resulting "happens before" edges ensure a smooth hand-off from one thread to the other. 

Acquire and release semantics may seem like an abstract concept, but they truly provide simple explanations of many common multithreaded programming practices. For example, consider two user-space threads accessing a global variable `s`: 

```
thread 1                              thread 2
        --------------------------------      ------------------------
        s = "hello";
        pthread_create(&t, NULL, t2, NULL);
                                              puts(s);
                                              s = "world";
        pthread_join(t, NULL);
        puts(s);
```

Are the accesses to the variable safe? Can thread 2 assume that `s` will read as `"hello"`, and likewise can thread 1 assume that `s` will be `"world"` after `pthread_join()`? The answer is affirmative, and we can explain why in terms of acquire and release semantics: 

  * `pthread_create()` has release semantics and synchronizes with the start of thread 2 (which has acquire semantics). Therefore, anything written before a thread is created can be safely accessed from the thread. 
  * Exiting thread 2 has release semantics and synchronizes with `pthread_join()` (which has acquire semantics). Therefore, anything the thread writes before exiting can be safely accessed after `pthread_join()`. 

Note that data is flowing from one thread to the other with no lock involved: congratulations, you have made it through the first example of lockless programming. To sum up: 

  * If the programmer wants thread 2 to "see the effects of" everything that happened so far in thread 1, the two threads need to synchronize with each other: this is done with a release operation in thread 1 and an acquire operation in thread 2\. 
  * Knowing which APIs provide acquire/release semantics lets you write code that relies on the ordering provided by those APIs. 

Having understood how release and acquire semantics work for high-level synchronization primitives, we can now consider them in the context of individual memory accesses. 

#### The message-passing pattern

In the previous paragraph we have seen how the acquire and release semantics of `pthread_create()` and `pthread_join()` allow the creator of a thread to exchange information with that thread and vice versa. We will now see how this kind of communication can happen in a lockless manner while threads run. 

If the message is a simple scalar value, for example a boolean, it could be read and written directly to a memory location. However, consider what happens if the message is a pointer, as in the following example: 

```
thread 1                            thread 2
        --------------------------------    ------------------------
        a.x = 1;
        message = &a;                       datum = message;
                                            if (datum != NULL)
                                              printk("%d\n", datum->x);
```

If `message` is initially `NULL`, thread 2 will read either `NULL` or `&a`, we don't know which. The problem is that, even if 

```
datum = message;
```

were to read `&a`, that assignment is still not synchronized against the assignment in thread 1: 

```
message = &a;
```

Therefore, there is no edge in the _happens before_ relation connecting the two threads: 

```
a.x = 1;                            datum = message;
           |                                    |
           |   happens before                   |
           v                                    v
        message = &a;                       datum->x
```

Because the two threads of execution are disconnected, there is still no guarantee that `datum->x` will read as 1; we don't know that the assignment to `a.x` _happens before_ that read or not.. For this to happen, the store and load must be endowed with release and acquire semantics respectively. 

To that end, we have the "store-release" and "load-acquire" operations. A store-release operation P, in addition to writing to a memory location, _synchronizes with_ a load-acquire operation Q if Q reads the value that was written by P. Here is a fixed version of the above code, using Linux's `smp_store_release()` and `smp_load_acquire()`: 

```
thread 1                                  thread 2
        --------------------------------          ------------------------
        a.x = 1;
        smp_store_release(&message, &a);          datum = smp_load_acquire(&message);
                                                  if (datum != NULL)
                                                    printk("%x\n", datum->x);
```

With this change, if `datum` is `&a` we can affirm that the store _happened before_ the load. (I am assuming for simplicity that only one thread can write `&a` to `message`. Saying "thread 2 reads the value written by thread 1" does not refer to the specific bit pattern that goes in memory, it really means that thread 1's store is the last whose effect is visible to thread 2). The relation looks like this now: 

```
a.x = 1;
           |
           v
        smp_store_release(&message, &a);  ----->  datum = smp_load_acquire(&message);
                                                      |
                                                      v
                                                  datum->x
```

And everything works. Because of transitivity, whenever thread 2 reads the value written by thread 1, everything that thread 1 did up to the store-release will also be visible to thread 2 after the load-acquire. Note that, unlike the `pthread_join()` case, "synchronizes with" does not mean that thread 2 "waits for" thread 1 to do the write. The above drawing only applies if thread 2 happens to read the value that thread 1 has written. 

In the Linux kernel, the above code will often be written in a slightly different manner: 

```
thread 1                              thread 2
        --------------------------------      ------------------------
        a.x = 1;
        smp_wmb();
        WRITE_ONCE(message, &a);              datum = READ_ONCE(message);
                                              smp_rmb();
                                              if (datum != NULL)
                                                printk("%x\n", datum->x);
```

In this case, the release and acquire semantics are provided by the _memory barriers_ `smp_wmb()` and `smp_rmb()`. Memory barriers also have acquire and release semantics, but they are a bit more complicated to reason about than simple loads and stores. We will get back to them when we talk about seqlocks. 

Regardless of whether one uses load-acquire/store-release or `smp_rmb()`/`smp_wmb()`, this is an extremely common pattern, and one that should be understood well. Among its uses we find: 

  * All sorts of ring buffers. Each entry of the ring buffer often points to other data; usually, there are also head/tail locations that contain an index in the ring buffer. The producer side will use store-release operations, synchronizing with load-acquire operations in the consumer. 
  * RCU. As far as the compiler is concerned, the familiar `rcu_dereference()` and `rcu_assign_pointer()` APIs are similar to load-acquire and store-release operations. Thanks to some assumptions that are true for all processors except the Alpha, `rcu_dereference()` can be compiled to a regular load; still, `rcu_assign_pointer()` synchronizes with `rcu_dereference()` as if it were a load-acquire operation. 
  * Publishing pointers into an array. In this (modified) KVM snippet, if `kvm_get_vcpu()` sees the incremented `kvm->online_vcpus`, the associated entry in the array will be valid: 

```
kvm_vm_ioctl_create_vcpu()                     kvm_get_vcpu()
            -----------------------------------------      -----------------------------------------------
            kvm->vcpus[kvm->online_vcpus] = vcpu;          if (idx < smp_load_acquire(&kvm->online_vcpus))
            smp_store_release(&kvm->online_vcpus,            return kvm->vcpus[idx];
                              kvm->online_vcpus + 1);      return NULL;
```

Apart from the mechanics of the load-acquire/store-release operations, there is another aspect of the message-passing pattern that you should ponder: it is a _single producer_ algorithm. If there are multiple writers, they must be protected against each other by other means, for example with a mutex. Lockless algorithms do not exist in a void; they are but one part of the concurrent programming toolbox, and they work best when combined with other, more traditional tools. 

This is just the beginning of an extended series on lockless algorithms. The next installment will look further at how atomic memory operations can be ordered, and look into how memory barriers are at the heart of both the "seqcounts" mechanism and the Linux scheduler.
