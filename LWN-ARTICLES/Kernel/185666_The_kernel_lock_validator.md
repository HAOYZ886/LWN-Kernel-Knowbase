---
title: The kernel lock validator
url: https://lwn.net/Articles/185666/
date: "May 31, 2006"
category: "Development tools-Kernel debugging; Lockdep; Spinlocks"
author: 
---

Locking is a necessary evil in operating systems; without a solid locking regime, different parts of the system will collide when trying to access the same resources, leading to data corruption and general chaos. But locking has hazards of its own; carelessly implemented locking can cause system deadlocks. As a simple example, consider two locks _L 1_ and _L 2_. Any code which requires both locks must take care to acquire the locks in the right order. If one function acquires _L 1_ before _L 2_, but another function acquires them in the opposite order, eventually the system will find itself in a situation where each function has acquired one lock and is blocked waiting for the other - a deadlock. 

A race condition like the one described above may be a one-in-a-million possibility, but, with computers, it does not take too long to exercise a code path a million times. Sooner or later, a system containing this sort of bug will lock up, leaving its users wondering what is going on. To avoid this sort of situation, kernel developers try to define rules for the order in which locks should be acquired. But, in a system with many thousands of locks, defining a comprehensive set of rules is challenging at best, and enforcing them is even harder. So locking bugs creep into the kernel, lurk until some truly inconvenient time, and eventually surprise some unsuspecting user. 

Over time, the kernel developers have made increasing use of automated code analysis tools as those tools become available. The latest such is [the first version of the lock validator patch](<http://lwn.net/Articles/185605/>), posted by Ingo Molnar. This patch (a 61-part set, actually) adds a complex infrastructure to the kernel which can then be used to prove that none of the locking patterns observed in a running system could ever deadlock the kernel. 

To that end, the lock validator must track real locking patterns in the kernel. There is no point, however, in tracking every individual lock - there are thousands of them, but many of them are treated in exactly the same way by the kernel. For example, every `inode` structure contains a spinlock, as does every `file` structure. Once the kernel has seen how locking is handled for one `inode` structure, it knows how it will be handled for every `inode` structure. So, somehow, the lock validator needs to be able to recognize that all spinlocks contained within (for example) the `inode` structure are essentially the same. 

To this end, every lock in the system (including rwlocks and mutexes, now) is assigned a specific key. For locks which are declared statically (for example, `files_lock`, which protects the list of open files), the address of the lock is used as the key. Locks which are allocated dynamically (as most locks embedded within structures are) cannot be tracked that way, however; there may be vast numbers of addresses involved, and, in any case, all locks associated with a specific structure field should be mapped to a single key. This is done by recognizing that these locks are initialized at run time, so, for example, `spin_lock_init()` is redefined as: 

```
# define spin_lock_init(lock)			\
        do {						\
    	static struct lockdep_type_key __key;		\
    							\
    	__spin_lock_init((lock), #lock, &__key);	\
        } while (0)
```

Thus, for each lock initialization, this code creates a static variable (`__key`) and uses its address as the key identifying the type of the lock. Since any particular type of lock tends to be initialized in a single place, this trick associates the same key with every lock of the same type. 

Next, the validator code intercepts every locking operation and performs a number of tests: 

  * The code looks at all other locks which are already held when a new lock is taken. For all of those locks, the validator looks for a past occurrence where any of them were taken _after_ the new lock. If any such are found, it indicates a violation of locking order rules, and an eventual deadlock. 

  * A stack of currently-held locks is maintained, so any lock being released should be at the top of the stack; anything else means that something strange is going on. 

  * Any spinlock which is acquired by a hardware interrupt handler can never be held when interrupts are enabled. Consider what happens when this rule is broken. A kernel function, running in process context, acquires a specific lock. An interrupt arrives, and the associated interrupt handler runs on the same CPU; that handler then attempts to acquire the same lock. Since the lock is unavailable, the handler will spin, waiting for the lock to become free. But the handler has preempted the only code which will ever free that lock, so it will spin forever, deadlocking that processor. 

To catch problems of this type, the validator records two bits of information for every lock it knows about: (1) whether the lock has ever been acquired in hardware interrupt context, and (2) whether the lock is ever held by code which runs with hardware interrupts enabled. If both bits are set, the lock is being used erroneously and an error is signaled. 

  * Similar tests are made for software interrupts, which present the same problems. 

The interrupt tests are relatively straightforward, requiring just four bits of information for each lock (though the situation is a little more complicated for rwlocks). But the ordering tests require a bit more work. For every known lock key, the validator maintains two lists. One of them contains all locks which have ever been held when the lock of interest (call it _L_) is acquired; it thus contains the keys of all locks which might be acquired before _L_. The other list (the "after" list) holds all locks acquired while the _L_ is held. These two lists thus encapsulate the proper ordering of how those other locks should be acquired relative to _L_. 

Whenever _L_ is acquired, the validator checks whether any lock on the "after" list associated with _L_ is already held. It should not find any, since all locks on the "after" list should only be acquired after acquiring _L_. Should it find a lock which should not be held, an error is signaled. The validator code also takes the "after" list of _L_ , connects it with the "before" lists of the currently-held locks, and convinces itself that there are no ordering or interrupt violations anywhere within that chain. If all the tests pass, the validator updates the various "before" and "after" lists and the kernel continues on its way. 

Needless to say, all this checking imposes a certain amount of overhead; it is not something which one will want to enable on production kernels. It is not quite as bad as one might expect, however. As the kernel does its thing, the lock validator maintains its stack of currently-held locks. It also generates a 64-bit hash value from that series of locks. Whenever a particular combination of locks is validated, the associated hash value is stored in a table. The next time that lock sequence is encountered, the code can find the associated hash value in the table and know that the checks have already been performed. This hashing speeds the process considerably. 

Of course, there are plenty of exceptions to the locking rules as understood by the validator. As a result, a significant portion of the validator patch set is aimed at getting rid of false error reports. For example, the validator normally complains if more than one lock with the same key is held at the same time - doing so is asking for deadlocks. There are situations, however, where this pattern is legitimate. For example, the block subsystem will often lock a block device, then lock a partition within that device. Since the partition also looks like a block device, the validator signals an error. To keep that from happening, the validator implements the notion of lock "subtypes." In this case, locks on partition devices can be marked with a different subtype, allowing their usage to be validated properly. This marking is done by using new versions of the locking functions (`spin_lock_nested()`, for example) which take a subtype parameter. 

The lock validator was added to [2.6.17-rc5-mm1](<http://lwn.net/Articles/185572/>), so interested people can play with it. Waiting for another -mm release might not be a bad idea, however; there has since been a fairly long series of validator fixes posted. 

The key point behind all of this is that deadlock situations can be found without having to actually make the kernel lock up. By watching the sequences in which locks are acquired, the validator can extrapolate a much larger set of possible sequences. So, even though a particular deadlock might only happen as the result of unfortunate timing caused by a specific combination of strange hardware, a rare set of configuration options, 220V power, a slightly flaky video controller, Mars transiting through Leo, an old version of `gcc`, an application which severely stresses the system (`yum`, say), and an especially bad Darl McBride hair day, the validator has a good chance of catching it. So this code should result in a whole class of bugs being eliminated from the kernel code base; that can only be a good thing.
