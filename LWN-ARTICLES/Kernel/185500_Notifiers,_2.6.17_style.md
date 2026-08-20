---
title: "Notifiers, 2.6.17 style"
url: https://lwn.net/Articles/185500/
date: "May 29, 2006"
category: Notifiers
author: 
---

While plowing through the flood of patches early in the 2.6.17 cycle, your editor missed a significant API change: the new notifier interface. Notifiers are an internal kernel mechanism allowing code to register to be told about events of interest. There are notifiers for memory hotplug events, CPU frequency policy changes, USB hotplug events, module loading and unloading, system reboots, network device changes, and more. 

Back in November, 2005, this page [looked at a proposed notifier API change](<http://lwn.net/Articles/160953/>) motivated by the lack of locking on the notifier chains themselves. That proposal received a lukewarm reception. Many low-level data structures in the kernel explicitly avoid performing any locking, on the assumption that the higher layers will have to be concerned with their own locking in any case. So, it was asked, why should notifiers be any different? The answer seems to be that, unlike many other data structures, notifiers tend to be used across relatively wide parts of the kernel, making it hard to use any locking regime except one designed for the notifiers themselves. In any case, a version of the notifier patch was merged for 2.6.17-rc1. 

The current form of the API defines three different types of notifiers: 

  * Blocking notifiers are always called from process context. The notifier code - along with the notification routines it calls - is allowed to sleep. 

  * Atomic notifiers can be called from atomic context, no sleeping allowed. 

  * Raw notifiers have no internal locking and no associated rules; they are simply the older form of the notifier API, preserved as a historical relic. 

For 2.6.17, all notifier chains have been converted to the blocking or atomic types; there are no users of the raw interface in the mainline kernel. The notifier patch includes no threatening noises about removing the raw interface, but, sooner or later, somebody is likely to come along and want to clean it up. So avoiding raw notifiers is probably a good idea; this article will concentrate on the other two types. 

Blocking notifiers are essentially a raw notifier with an rwsem added for mutual exclusion. Any operation on a blocking notifier may, well, block on that rwsem. These notifiers can be created in the usual two ways: 

```
#include <linux/notifier.h>
    
        BLOCKING_NOTIFIER_HEAD(my_notifier);
    
        struct blocking_notifier_head my_notifier;
        BLOCKING_INIT_NOTIFIER_HEAD(my_notifier);
```

Code which wishes to hook into a blocking notifier should first fill in a `notifier_block` structure: 

```
struct notifier_block {
    	int (*notifier_call)(struct notifier_block *block, 
                                 unsigned long event, 
    			     void *data);
     	int priority;
    	/* ... */
        };
```

The `notifier_call` field should point to the function to be called when something interesting happens; the `event` and `data` parameters will be provided by the code generating the event. Notifiers are called in order of increasing `priority`; the return value from the final notifier called will be passed back to the code signalling the event. Normally, the final notifier is the one with the highest `priority` value, but any notifier can halt further processing by returning a value with the bit indicated by `NOTIFIER_STOP_MASK` set. Other than that one bit (currently `0x8000`), the return values are arbitrary (as far as the notification code is concerned), but the convenience values `NOTIFY_OK` ("so far so good"), `NOTIFY_STOP` ("all is well, but don't call any more notifiers") and `NOTIFY_BAD` ("stop calling notifiers and veto the proposed action") are available. 

Once the code has a `notifier_block` ready, it should register it with: 

```
int blocking_notifier_chain_register(struct blocking_notifier_head *chain,
                                             struct notifier_block *nb);
```

The return value is apparently intended to allow an error status to be returned if the registration fails, but the 2.6.17 version of the code cannot fail. 

A blocking notifier can be unregistered with: 

```
int blocking_notifier_chain_unregister(struct blocking_notifier_head *chain,
                                               struct notifier_block *nb);
```

This call will return `-ENOENT` if the given notifier was not actually registered. 

Code which wishes to use a blocking notifier chain to signal an event can do so with: 

```
int blocking_notifier_call_chain(struct blocking_notifier_head *chain,
    		                     unsigned long event, 
    				     void *data);
```

This function will call all notifiers in `chain` (unless one of them stops the process partway through), returning the value from the last notifier called. 

Atomic notifiers replace the rwsem with a spinlock; the API is very similar: 

```
ATOMIC_NOTIFIER_HEAD(my_notifier);
    
        struct atomic_notifier_head my_notifier;
        ATOMIC_INIT_NOTIFIER_HEAD(my_notifier);
    
        int atomic_notifier_chain_register(struct atomic_notifier_head *chain,
                                           struct notifier_block *nb);
        int atomic_notifier_chain_unregister(struct atomic_notifier_head *chain,
                                             struct notifier_block *nb);
    
        int atomic_notifier_call_chain(struct atomic_notifier_head *chain,
    		                   unsigned long event, 
    				   void *data);
```

Note that atomic notifiers use the same `notifier_block` structure as the blocking variety does. Nothing will ever sleep in the atomic notifier code, however, and notifier functions called from an atomic chain are not allowed to sleep either. 

As noted above, all notifier chains in the kernel have been changed to one of the above types; any out-of-tree code which uses a kernel chain will have to be updated accordingly. See the explanatory text for [the notifier patch](<http://permalink.gmane.org/gmane.linux.kernel.commits.head/76328>) for a summary of what type was assigned to each existing chain in the mainline kernel.
