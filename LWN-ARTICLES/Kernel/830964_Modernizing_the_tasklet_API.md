---
title: Modernizing the tasklet API
url: https://lwn.net/Articles/830964/
date: "September 14, 2020"
category: Tasklets
author: "September 14, 2020 This article was contributed by Marta Rybczyńska"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

September 14, 2020

This article was contributed by Marta Rybczyńska

Tasklets offer a deferred-execution method in the Linux kernel; they have been available since the 2.3 development series. They allow interrupt handlers to schedule further work to be executed as soon as possible after the handler itself. The tasklet API has its shortcomings, but it has stayed in place while other deferred-execution methods, including workqueues, have been introduced. Recently, Kees Cook posted a security-inspired [patch set](<https://lwn.net/ml/kernel-hardening/20200716030847.1564131-1-keescook@chromium.org/>) (also including work from Romain Perier) to improve the tasklet API. This change is uncontroversial, but it provoked a discussion that might lead to the removal of the tasklet API in the (not so distant) future.

The need for tasklets and other deferred execution mechanisms comes from the way the kernel handles interrupts. An interrupt is (usually) caused by some hardware event; when it happens, the execution of the current task is suspended and the interrupt handler takes the CPU. Before the introduction of [threaded interrupts](<https://lwn.net/Articles/302043/>), the interrupt handler had to perform the minimum necessary operations (like accessing the hardware registers to silence the interrupt) and then call an appropriate deferred-work mechanism to take care of just about everything else that needed to be done. Threaded interrupts, yet another import from the realtime preemption work, move the handler to a kernel thread that is scheduled in the usual way; this feature was merged for the 2.6.30 kernel, by which time tasklets were well established. 

An interrupt handler will schedule a tasklet when there is some work to be done at a later time. The kernel then runs the tasklet when possible, typically when the interrupt handler finishes, or the task returns to the user space. The tasklet callback runs in atomic context, inside a software interrupt, meaning that it cannot sleep or access user-space data, so not all work can be done in a tasklet handler. Also, the kernel only allows one instance of any given tasklet to be running at any given time; multiple different tasklet callbacks can run in parallel. Those limitations of tasklets are not present in more recent deferred work mechanisms like workqueues. But still, the current kernel contains more than a hundred users of tasklets.

Cook's patch set changes the parameter type for the tasklet's callback. In current kernels, they take an `unsigned long` value that is specified when the tasklet is initialized. This is different from other kernel mechanisms with callbacks; the preferred way in current kernels is to use a pointer to a type-specific structure. The change Cook proposes goes in that direction by passing the tasklet context ([`struct tasklet_struct`](<https://elixir.bootlin.com/linux/latest/source/include/linux/interrupt.h#L606>)) to the callback. The goal behind this work is to avoid a number of problems, including a need to cast from the `unsigned int` to a different type (without proper type checking) in the callback. The change allows the removal of the (now) redundant `data` field from the tasklet structure. Finally, this change mitigates the possible buffer overflow attacks that could overwrite the callback pointer and the `data` field. This is likely one of the primary objectives, as the work was [first posted](<https://lwn.net/ml/kernel-hardening/20190929163028.9665-1-romain.perier@gmail.com/>) (in 2019) on the kernel-hardening mailing list.

#### Plotting the removal of tasklets

The patch set caused no controversies, but the discussion changed direction following [this comment](<https://lwn.net/ml/kernel-hardening/20200716075718.GM10769@hirez.programming.kicks-ass.net/>) from Peter Zijlstra, who said: ""I would _MUCH_ rather see tasklets go the way of the dodo [...] Can't we stage an extinction event here instead?"" In a response, Sebastian Andrzej Siewior [suggested](<https://lwn.net/ml/kernel-hardening/20200716081538.2sivhkj4hcyrusem@linutronix.de/>) that tasklets could be replaced with threaded interrupts, as they also run in atomic context. Dmitry Torokhov [suggested](<https://lwn.net/ml/kernel-hardening/CAKdAkRQHRobiG-RpifyrAmV9ENgENn_woPBVXpRrhKwRBf9Esw@mail.gmail.com/>) immediately expiring timers instead. Cook [replied](<https://lwn.net/ml/kernel-hardening/202007161416.9C1B8F3D26@keescook/>) that the change could not be done mechanically and gave some examples of more complicated usage of tasklets. One such case is the AMD `ccp` crypto driver, which combines tasklets with [DMA engines](<https://www.kernel.org/doc/html/latest/driver-api/dmaengine/>), while another is the Intel i915 GPU driver, which schedules GPU tasks with tasklets.

In the following messages, Thomas Gleixner ["grudgingly" acked the patch set](<https://lwn.net/ml/kernel-hardening/87h7tpa3hg.fsf@nanos.tec.linutronix.de/>), but also spoke in favor of removing tasklets: ""I'd rather see tasklets vanish from the planet completely, but that's going to be a daring feat."" The developers agreed that removing tasklets would be a logical next step, but that this is a bigger task than improving their API. The [Kernel Self-Protection Project](<https://kernsec.org/wiki/index.php/Kernel_Self_Protection_Project>) has [added a dedicated task](<https://github.com/KSPP/linux/issues/94>) for this objective.

The removal of the tasklet API has been discussed before; LWN [covered it in 2007](<https://lwn.net/Articles/239633/>). At that time, the main argument for the removal of tasklets was to limit latencies (since tasklets run in software interrupt mode, they can block even the highest-priority tasks). The argument against removing tasklets was a possible performance loss for drivers that need to react quickly to events. At that time, threaded interrupts were not yet included in the mainline.

In current kernels, tasklets can be replaced by workqueues, timers, or threaded interrupts. If threaded interrupts are used, the work may just be executed in the interrupt handler itself. Those newer mechanisms do not have the disadvantages of tasklets and should satisfy the same needs, so developers do not see a reason to keep tasklets. It seems that any migration away from tasklets will be done one driver (or subsystem) at a time. For example, Takashi Iwai [already reported](<https://lwn.net/ml/kernel-hardening/s5hpn7wz8o6.wl-tiwai@suse.de/>) having the conversion ready for sound drivers.

#### Current API changes

While the removal of tasklets remains a longer-term goal, the developers are proceeding with the API changes. The modifications in the tasklet API performed by Cook's patch set are minimal and consist of creating a new initialization macro and adding one initialization function. In current kernels, tasklets are declared with: 

```
#define DECLARE_TASKLET(name, func, data) \
              struct tasklet_struct name = { NULL, 0, ATOMIC_INIT(0), func, data }
```

To allow compatibility with existing users, all calls to the "old" `DECLARE_TASKLET()` were changed to `DECLARE_TASKLET_OLD` with the following definition:

```
#define DECLARE_TASKLET_OLD(name, _func)        \
              struct tasklet_struct name = {             \
              .count = ATOMIC_INIT(0),            	     \
              .func = _func,                    	     \
         }
```

The same modifications were done to the `DECLARE_TASKLET_DISABLED()` macro. The conversion to `DECLARE_TASKLET_OLD()` turned out to be mechanical, since all those users provided zero as the `data` parameter.

A following patch included a new version of the declaration macro that does not contain that `data` parameter:

```
#define DECLARE_TASKLET(name, _callback)        \
              struct tasklet_struct name = {             \
              .count = ATOMIC_INIT(0),            	     \
              .callback = _callback,                     \
              .use_callback = true,                	     \
         }
```

In the new API, the callback function is stored in the `callback()` field rather than `func()`; the callback itself simply takes a pointer to the `tasklet_struct` structure as its one argument: 

```
void (*callback)(struct tasklet_struct *t);
```

That structure will normally be embedded within a larger, user-specific structure, the pointer to which can be obtained with the `container_of()` macro in the usual way. The patch set also adds a function to initialize a tasklet at run time, with the following prototype:

```
void tasklet_setup(struct tasklet_struct *t,
              void (*callback)(struct tasklet_struct *));
```

The tasklet subsystem will invoke the callback in either the new or the old mode, depending on how the tasklet was initialized; beyond that, the behavior of tasklets is unchanged. 

#### Where to from here

The team working on the change submitted a number of patches to convert all tasklet initializations in the kernel to the new `tasklet_setup()` function. Another task remains to remove the tasklets from all those users. The work in some subsystems has already started. Developers are welcome to help with the conversion of all subsystems to the new API and, eventually, removing all tasklet users from the kernel. There is certainly plenty of will on the part of the kernel developers to do so, but this is likely going to take a few kernel development cycles.
