---
title: An object debugging infrastructure
url: https://lwn.net/Articles/271614/
date: "March 3, 2008"
category: "Debugging; Development tools-Kernel debugging"
author: "By Jonathan Corbet March 3, 2008"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
March 3, 2008

Thomas Gleixner has discovered that being the maintainer of a core kernel infrastructure module can bring some special challenges. Whenever somebody's kernel oopses in the timer code, for example, Thomas tends to hear about it. The only problem is that the timer code is almost never where the bug is. Instead, it's far more likely that some other kernel subsystem has corrupted an active timer, leaving a bomb that will only explode later, in the timer code, when that timer is set to expire. At that point, it can be hard to figure out where the real problem is, as the culprit will be long gone. 

In response, Thomas developed some special-purpose code aimed at finding the real source of timer-related problems, preferably before it brings down the kernel. He has now generalized that code and posted it as the [object debugging infrastructure patch](<http://lwn.net/Articles/271582/>), which was subsequently [significantly revised](<http://lwn.net/Articles/271808/>). As this code develops, it has the potential to help find whole classes of especially difficult bugs before they bring the system down. 

There's a few steps involved in adding support for object debugging to a new subsystem. The first is to create and populate a `debug_obj_descr` structure (defined in `<linux/debugobjects.h>`): 

```
struct debug_obj_descr {
    	const char		*name;
    
    	int (*fixup_init)	(void *addr, enum debug_obj_state state);
    	int (*fixup_activate)	(void *addr, enum debug_obj_state state);
    	int (*fixup_destroy)	(void *addr, enum debug_obj_state state);
    	int (*fixup_free)	(void *addr, enum debug_obj_state state);
        };
```

The `name` field is the name of the subsystem; it is used in debugging output. We will return to the other fields below. 

The next step is to call into the object debugging code whenever an action of interest involves one of the tracked objects. There is a set of functions used for this purpose: 

```
void debug_object_init      (void *addr, struct debug_obj_descr *descr);
        void debug_object_activate  (void *addr, struct debug_obj_descr *descr);
        void debug_object_deactivate(void *addr, struct debug_obj_descr *descr);
        void debug_object_destroy   (void *addr, struct debug_obj_descr *descr);
        void debug_object_free      (void *addr, struct debug_obj_descr *descr);
```

In each case, `addr` is a pointer to the object being operated on, and `descr` is a pointer to the `debug_obj_descr` structure mentioned above. The meaning of each call is: 

  * `debug_object_init()`: the object is being initialized. 

  * `debug_object_activate()`: it is being added to a subsystem list. For timer debugging, this action happens when `add_timer()` is called. 

  * `debug_object_deactivate()`: the object is being removed from a subsystem list. 

  * `debug_object_destroy()`: the object is being destroyed and is no longer referenced within the subsystem. This call is not used in the version 2 patch set. 

  * `debug_object_free()`: the object is being freed. 

The debugging code maintains a hashed set of lists for tracking objects; each object is added to the appropriate list when one of the above calls is made. As actions are performed on the objects, their state is tracked. In this way, the debugging code is able to test for a number of common mistakes, including deactivating an object which is not active, reinitializing active objects, or adding objects twice. 

When something goes wrong, a backtrace is sent to the system logs. Since this backtrace identifies where the original error is made, it is likely to be far more useful than the trace associated with the system crash which will probably come later. But this infrastructure can also help to make that crash less likely, in that each subsystem can register a set of "fixup functions." These, of course, are all the methods in the `debug_obj_descr` structure which we glossed over above. 

For example, if a call to `debug_object_init()` is made with an object which has already been activated, the debugging infrastructure will respond with a call to the `fixup_init()` callback, passing in the object in question and its current state (`ODEBUG_STATE_ACTIVE` in this case). The callback should return zero if it is able to, somehow, repair the damage. Even if things cannot be truly fixed, though, there is still use for this function; the timer code, for example, will disable an active timer if the calling code mishandles it. The kernel will almost certainly not operate as expected, but, at least, it has a smaller chance of crashing at some random time in the future. 

Most debugging checks are performed in response to calls from within the subsystem itself. There is one useful check which cannot be done that way, though: detecting the freeing of objects which are still under some sort of subsystem management. To catch that mistake, Thomas's patch inserts a hook into functions like `kfree()` and `free_hot_cold_page()`. Every time an object is freed, the code checks through the appropriate list to see if it is still seen as being active in some subsystem. Freeing an object which is still known to a subsystem is almost always a bug - one which can be hard to track down later on. 

The check on freed memory objects is clearly a useful debugging tool. It could also have a nontrivial overhead, though, since it requires searching a list every time some memory is freed. So it has its own configuration option and can be configured out of the kernel, even if the rest of the debugging code is built in. 

At this point, only the timer subsystem is covered by this infrastructure, but there are plenty of other obvious candidates. Perhaps at the top of the list would be kobjects, which are famously susceptible to all kinds of programming mistakes. So expect to see the coverage of this code grow in the near future.
