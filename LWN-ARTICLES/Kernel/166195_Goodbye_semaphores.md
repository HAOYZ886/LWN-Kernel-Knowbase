---
title: "Goodbye semaphores?"
url: https://lwn.net/Articles/166195/
date: "January 3, 2006"
category: "Locking mechanisms-Mutexes; Semaphores"
author: 
---

In [the previous episode](<http://lwn.net/Articles/165039/>), Ingo Molnar had posted his own version of the mutex patch, adding a new synchronization primitive to the kernel. Ingo has continued to refine this patch set, with frequent releases; the current version is ~~[V10](<http://lwn.net/Articles/166052/>)~~ ~~[V11](<http://lwn.net/Articles/166135/>)~~ ~~[V12](<http://lwn.net/Articles/166168/>)~~ ~~[V13](<http://lwn.net/Articles/166277/>)~~ [V14](<http://lwn.net/Articles/166280/>). This patch set has faced ongoing resistance from Andrew Morton, who didn't see the reasons for adding a new mutual exclusion mechanism to the kernel. Andrew, instead, wished that the developers would concentrate on fixing the problems with the current semaphore code. 

Perhaps the most significant development since then has been [a private conversation between Andrew and Ingo](<https://lwn.net/Articles/166198/>). There is, it seems, a plan in place which would replace the current semaphore implementation entirely. Almost all current semaphore users are implementing simple mutual exclusion areas, so they would be converted over to the new mutex type directly. An estimated 90% of current semaphore users fall into this category. Of the remaining users, about 90% employ semaphores to indicate event completion. The task of converting those users to the [completion](<http://lwn.net/Articles/23993/>) type has been ongoing for some time; replacing semaphores would require finishing this job. Finally, an estimated 1% of the semaphores in the kernel are used for their counting feature; they can be converted over to a (not yet posted) architecture-independent counter type. 

Once all that work is done, semaphores could be removed from the kernel altogether. Says Andrew: ""It's a lot of churn, but we'll end up with a better end result and a somewhat-net-simpler kernel, so I'm happy."" Linus, meanwhile, has offered some suggestions for improvements (already incorporated by Ingo) and [stated](<https://lwn.net/Articles/166199/>): ""At that point I'd like to switch to mutexes just because the code is cleaner!"" 

Since then, most of the discussion has been concerned with the details of the mutex implementation rather than whether it is fundamentally a good idea or not. The main objections would appear to have been overcome. So, unless something new comes up, it looks like this change is going to happen; the only question is "when." The next couple of weeks will determine whether the mutex code will be part of 2.6.16 or not. Then all that's left is the long task of converting all semaphore users over and, finally, removing the old semaphore code.
