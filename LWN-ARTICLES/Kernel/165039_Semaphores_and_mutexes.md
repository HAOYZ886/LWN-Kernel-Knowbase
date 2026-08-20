---
title: Semaphores and mutexes
url: https://lwn.net/Articles/165039/
date: "December 20, 2005"
category: "Locking mechanisms-Mutexes; Semaphores"
author: 
---

[Last week's Kernel Page](<http://lwn.net/Articles/163269/>) covered the mutex patch by David Howells. The discussion did not stop at that point, however, so here's this week's episode. 

There was some fairly strong pushback against the mutex patch after last week's article was written. Linus [expressed his thoughts](<https://lwn.net/Articles/165040/>) this way: 

A patch that 

  * creates a non-counting mutex 
  * .. that is SLOWER than the current counting one 
  * .. and keeps the old "semaphore" and "up/down" naming 

is simply INCREDIBLY BROKEN. It has absolutely _zero_ redeeming features. I can't understand how there are a hundred emails in my mailbox even discussing it. 

Here is [Andrew Morton's take](<https://lwn.net/Articles/165041/>): 

I must say that my interest in this stuff is down in needs-an-electron-microscope-to-locate territory. down() and up() work just fine and they're small, efficient, well-debugged and well-understood. We need a damn good reason for taking on tree-wide churn or incompatible renames or addition of risk. What's the damn good reason here? 

Please. Go fix some bugs. We're not short of them. 

The objections should be coming into focus at this point. One problem had to do with performance; the mutex patch was supposed to be faster, but that was not the case in the posted version (which lacked architecture-specific implementations). There was a long discussion on why the semaphore code could not be improved on in this regard. It seems that, on the most popular architectures at least, the locked decrement-and-test code used by semaphores is hard to beat. 

David's patch also introduced a sort of global flag day, changing the locking primitives used by vast amounts of code all at once. But it kept the old semaphore function names and applied them to the new mutex type, creating a confusing sort of interface. There was resistance to this choice of naming, but also a great deal of resistance to the idea of making major changes throughout the kernel without a very strong idea of what was being gained for it. All told, the mutex patch set looked like it had a rough road ahead of it. 

Enter Ingo Molnar, who has posted [a mutex patch of his own](<http://lwn.net/Articles/164802/>). Ingo's mutexes are derived from the code used in the realtime preemption patch, of course, but they have been heavily modified to avoid the objections which greeted David's patch. In this version, a mutex is a separate data type, with its own API: 

```
DEFINE_MUTEX(name);
    
         mutex_init(mutex);
     
         void mutex_lock(struct mutex *lock);
         int mutex_lock_interruptible(struct mutex *lock);
         int mutex_trylock(struct mutex *lock);
         void mutex_unlock(struct mutex *lock);
         int mutex_is_locked(struct mutex *lock);
```

The existing semaphore interface is not changed in any way - at least, not in any way visible to the rest of the kernel. There is an interesting feature, however: the semaphore functions (`down()`, `up()`, and friends) have been augmented to be able to handle mutex arguments as well as semaphores. This feature is a migration tool: a subsystem which is being considered for migration over to the mutex type can have its semaphores changed to mutexes, but no other code changes are required. The various checks built into the mutex type will quickly set off alarms if a mutex is being used as a counting semaphore. In that case, the locks can be changed back to semaphores and the whole episode forgotten. If, instead, all seems well, the semaphore calls can be turned into mutex calls. Eventually, when the migration work is complete, this helper code can be removed from the kernel. 

The real point of all the above is that, unlike David's patch, this version of mutexes imposes no flag day on the kernel. It is a new primitive, with its own API, and bits of the kernel can be converted over one by one. 

Ingo claims that his mutex code is significantly faster than semaphores used as mutexes. The code itself is a bit smaller and tighter, which helps. But he also gets some impressive performance improvements on some tests: a filesystem-based test more than doubled its speed on an eight-processor system. That is the sort of improvement which can help to motivate the quick merging of a patch. 

In this case, developers started to wonder just why the semaphore code was so much slower. Some research turned up the fact that, on the x86 architecture, each cycle through a semaphore had the potential to wake up two separate waiting processes, each of which would then contend for the lock. Nobody knows why the code is this way - Linus is [mystified by it](<https://lwn.net/Articles/165045/>). It quickly became clear, though, that taking out the redundant wakeup breaks the semaphores and causes lockups. For now, it is a bit of black magic which must remain for the whole thing to work. 

Ingo quickly [seized on this revelation](<https://lwn.net/Articles/165046/>) to drive home one of his other points: 

If this really is a bug that hid for years, it shows that the semaphore code is too complex to be properly reviewed and improved. Hence even assuming that the mutex code does not bring direct code advantages (which i'm disputing :-), the mutex code is far simpler and thus easier to improve. 

Linus seems to have [heard this argument](<https://lwn.net/Articles/165047/>): 

And don't get me wrong: if it's easier to just ignore the performance bug, and introduce a new "struct mutex" that just doesn't have it, I'm all for it. 

He doesn't like the under-the-hood semaphore changes, though, and would like that part of the patch taken out. 

Ingo's [initial posting](<http://lwn.net/Articles/164802/>) contains no less than ten reasons why he thinks the mutex patch should go on; rather than try to rephrase all of those arguments, your editor suggests going straight to the source. It is worth noting that, among other things, merging this mutex patch would move another piece of the realtime preemption patch into the mainline - even though many of the realtime-specific features (priority inheritance, for example) are missing.
