---
title: wait_var_event()
url: https://lwn.net/Articles/750774/
date: "April 3, 2018"
category: Scheduler
author: "By Jonathan Corbet April 3, 2018"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
April 3, 2018

One of the trickiest aspects to concurrency in the kernel is waiting for a specific event to take place. There is a wide variety of possible events, including a process exiting, the last reference to a data structure going away, a device completing an operation, or a timeout occurring. Waiting is surprisingly hard to get right — race conditions abound to trap the unwary — so the kernel has accumulated a large set of `wait_event_*()` macros to make the task easier. An attempt to add a new one, though, has led to the generalization of specific types of waits for 4.17. 

As an example of how specialized these wait macros have become, consider `wait_on_atomic_t()`: 

```
int wait_on_atomic_t(atomic_t *val, wait_atomic_t_action_f action,
        			 unsigned mode);
```

The purpose of this function is to wait until the `atomic_t` variable pointed to by `val` drops to zero. The function that actually puts the current process to sleep is `action()` (usually [`atomic_t_wait()`](<https://elixir.bootlin.com/linux/latest/source/kernel/sched/wait_bit.c#L223>), but some callers have special needs), and the `mode` argument is the state the task should sleep in. Any code that decrements this variable should make a call to: 

```
void wake_up_atomic_t(atomic_t *val);
```

This function will check the value of `*val` and wake any waiting tasks if that value is zero. 

`wait_on_atomic_t()` is a useful function, with around twenty callers in the 4.16 kernel. But, inevitably, somebody needed to wait for an `atomic_t` variable to reach one instead of zero. That somebody was Dan Williams, who [posted a patch](<https://lwn.net/Articles/750776/>) adding a new function called `wait_on_atomic_one()` for that purpose. Peter Zijlstra, perhaps fearing the eventual addition of `wait_on_atomic_two()` and `wait_on_atomic_42()`, decided to [come up with a better solution](<https://lwn.net/Articles/749666/>) to the problem. 

#### wait_var_event()

The result is a new API designed to solve the problem of waiting for something to happen with a given variable: 

```
int wait_var_event(void *var, test);
        void wake_up_var(void *var);
```

A call to `wait_var_event()` will wait until `test` evaluates to a true value. It can be used to replace a call to `wait_on_atomic_t()` in this way: 

```
wait_var_event(&atomic_var, !atomic_read(&atomic_var));
```

On the wake side, `wake_up_var()` does not test the value of the variable as `wake_up_atomic_t()` does, so code that looks like: 

```
atomic_dec(&atomic_var);
        wake_up_atomic_t(&atomic_var);
```

needs to be changed to look like this: 

```
if (atomic_dec_and_test(&atomic_var))
            wake_up_var(&atomic_var);
```

This mechanism can be used to implement `wait_on_atomic_one()` in a fairly straightforward manner. It can also wait on any type of variable, not just `atomic_t` if the need arises. Zijlstra's patch replaces a number of `wait_on_atomic_t()` calls in the kernel; work to replace the rest has been done since this patch series was posted. 

#### Under the hood

A look at the `wait_var_event()` interface is likely to raise a couple of questions. One of those is why this macro needs a pointer to the variable involved if it is not actually checking the value of that variable or, indeed, does not even know what the type of the variable is. Developers experienced with the kernel's scheduling mechanism know that a wait requires placing an entry on a wait queue, but there is no such queue in evidence here. The answer to both of those questions lies in how `wait_var_event()` is implemented. 

`wait_var_event()` is a macro that, naturally, defers the actual work to `__wait_var_event()`. That macro supplies some defaults — the wait is done in the `TASK_UNINTERRUPTIBLE` state, using `schedule()`, in a non-exclusive mode — and then calls, inevitably, `___wait_var_event()` to do the real work. To paraphrase Randall Davis, it's one thing to have a kernel macro, and quite another to have a double-underscore macro, but a developer with a triple-underscore macro is truly blessed. 

Down in triple-underscore territory, the macro uses the kernel's [bit waitqueue](<https://elixir.bootlin.com/linux/v4.16/source/kernel/sched/wait_bit.c>) mechanism. Allocating a wait queue, making it available to the code on the wakeup side, and tracking wait-queue entries is a bit cumbersome. For a wait operation on a single variable that may never be repeated, it represents a fair amount of overhead. The bit waitqueue code implements a set of shared waitqueues intended to make life easier and more efficient for this kind of case. 

The reason that `wait_var_event()` needs a pointer to the variable is that this address is used to identify the wait queue that will be used to wait for events. The address is hashed, reduced to eight bits, and used to index into an array of 256 wait queues; the waiting process will then wait on the indicated queue. A call to `wake_up_var()` will go through the same process to find the correct wait queue, then wake any tasks there that are waiting on the same variable address. 

There is a bit of a tradeoff inherent in this mechanism: the shared wait queues will save memory and the overhead of managing a rather larger number of single-use wait queues, but it will also have to scan (and pass over) any other entries that happened to end up in the same wait queue. With luck, there will not be very many of those, so this mechanism should be much more efficient overall. 

There is, of course, the usual set of variants — `wait_var_event_timeout()`, `wait_var_event_killable()`, etc. This new functionality, along with a conversion of all `wait_on_atomic_t()` users and the removal of that function, has been merged for the 4.17 release. It may be a small change to an obscure core-kernel detail, but it is also a good example of how these APIs evolve over time.
