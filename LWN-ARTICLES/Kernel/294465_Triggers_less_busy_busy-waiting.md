---
title: "Triggers: less busy busy-waiting"
url: https://lwn.net/Articles/294465/
date: "August 18, 2008"
category: Triggers
author: "By Jonathan Corbet August 18, 2008"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
August 18, 2008

Kernel code must often wait for something to happen elsewhere in the system. The preferred way to wait is to use any of a number of interfaces to wait queues, allowing the processor to perform other tasks in the mean time. If the kernel code in question is running in an atomic mode, though, it cannot block, so the use of wait queues is not an option. Traditionally, in such situations, the programmer simply must code a busy wait which sits in a tight loop until the required event takes place. 

Busy waits are always undesirable, but, in some situations, they become even more so. If the wait is going to be relatively long, it would be better to put the processor into a lower power state. After all, nobody cares if it executes its empty loop at full speed, or, even, whether the loop executes at all. If the wait is running within a virtualized guest, the situation can be even worse: by looping in the processor, a busy wait can actively prevent the running of the code which will eventually provide the event which is being waited for. In a virtualized environment, it is far better to simply suspend the virtual system altogether than to let it busy wait. 

Jeremy Fitzhardinge has proposed a solution to this problem in the form of the [trigger API](<http://lwn.net/Articles/294436/>). A trigger can be thought of as a special type of continuation intended for use in a specific environment: situations where preemption is disabled and sleeping is not possible, but where it is necessary to wait for an external event. 

A trigger is set up in either of the two usual patterns: 

```
#include <linux/trigger.h>
    
        DEFINE_TRIGGER(my_trigger);   
        /* ... or ... */
        trigger_t my_trigger;
        trigger_init(&my_trigger);
```

There is a sequence of calls which must be made by code intending to wait for a trigger: 

```
trigger_reset(&my_trigger);
        while(!condition)
        	trigger_wait(&my_trigger);
        trigger_finish(&my_trigger);
```

Triggers are designed to be safe against race conditions, in that if a trigger is fired after the `trigger_reset()` call, the subsequent `trigger_wait()` call will return immediately. As with any such primitive, false "wakeups" are possible, so it is necessary to check for the condition being waited for and wait again if need be. 

Code which wishes to signal completion to a thread waiting on a trigger need only make a call to: 

```
void trigger_kick(trigger_t *trigger);
```

This code should, of course, ensure that the waiting thread will see that the resource it was waiting for is available before calling `trigger_kick()`. 

A reader of the generic implementation of triggers may be forgiven for wondering what the point is; most of the functions are empty, and `trigger_wait()` turns into a call to `cpu_relax()`. In other words, it's still a busy wait, just like before except that now it's hidden behind a set of trigger functions. The idea, of course, is that better versions of these functions can be defined in architecture-specific code. If the target architecture is actually a virtual machine environment, for example, a trigger can simply suspend the execution of the machine altogether. To that end, there is a new set of paravirt_ops allowing hypervisors to implement the trigger operations. 

Jeremy has also created an implementation for the x86 architecture which uses the relatively new `monitor` and `mwait` instructions. In this implementation, a trigger is a simple integer variable. A call to `trigger_reset()` turns into a `monitor` instruction, informing the processor that it should watch out for changes to that integer variable. The `mwait` instruction built into `trigger_wait()` halts the processor until the monitored variable is written to. No more busy waiting is required. 

There is a certain elegance to the `monitor`/`mwait` implementation, but Arjan van de Ven [worries](<https://lwn.net/Articles/294473/>) that it may prove to be too slow. So changes to the x86 implementation are possible. There have not been a lot of comments about the API itself, though, so the trigger functions may well make it into the mainline in something close to their current form.
