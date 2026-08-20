---
title: Integrating and Validating dynticks and Preemptable RCU
url: https://lwn.net/Articles/279077/
date: "April 22, 2008"
category: "Read-copy-update"
author: "April 22, 2008 This article was contributed by Paul McKenney"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

April 22, 2008

This article was contributed by Paul McKenney

### Introduction

Read-copy update (RCU) is a synchronization mechanism that was added to the Linux kernel in October of 2002. RCU is most frequently described as a replacement for reader-writer locking, but it has also been used in a number of other ways. RCU is notable in that RCU readers do not directly synchronize with RCU updaters, which makes RCU read paths extremely fast, and also permits RCU readers to accomplish useful work even when running concurrently with RCU updaters. 

In early 2008, a preemptable variant of RCU was accepted into mainline Linux in support of real-time workloads, a variant similar to the RCU implementations in [ the -rt patchset](<http://www.kernel.org/pub/linux/kernel/projects/rt/>) since August 2005. Preemptable RCU is needed for real-time workloads because older RCU implementations disable preemption across RCU read-side critical sections, resulting in excessive real-time latencies. 

However, one disadvantage of the -rt implementation was that each grace period required work to be done on each CPU, even if that CPU is in a low-power "dynticks-idle" state, and thus incapable of executing RCU read-side critical sections. The idea behind the dynticks-idle state is that idle CPUs should be physically powered down in order to conserve energy. In short, preemptable RCU can disable a valuable energy-conservation feature of recent Linux kernels. Although Josh Triplett and Paul McKenney had discussed some approaches for allowing CPUs to remain in low-power state throughout an RCU grace period (thus preserving the Linux kernel's ability to conserve energy), matters did not come to a head until Steve Rostedt integrated a new dyntick implementation with preemptable RCU in the -rt patchset. 

This combination caused one of Steve's systems to hang on boot, so in October, Paul coded up a dynticks-friendly modification to preemptable RCU's grace-period processing. Steve coded up `rcu_irq_enter()` and `rcu_irq_exit()` interfaces called from the `irq_enter()` and `irq_exit()` interrupt entry/exit functions. These `rcu_irq_enter()` and `rcu_irq_exit()` functions are needed to allow RCU to reliably handle situations where a dynticks-idle CPUs is momentarily powered up for an interrupt handler containing RCU read-side critical sections. With these changes in place, Steve's system booted reliably, but Paul continued inspecting the code periodically on the assumption that we could not possibly have gotten the code right on the first try. 

Paul reviewed the code repeatedly from October 2007 to February 2008, and almost always found at least one bug. In one case, Paul even coded and tested a fix before realizing that the bug was illusory, but in all cases, the "bug" was in fact illusory. 

Near the end of February, Paul grew tired of this game. He therefore decided to enlist the aid of [Promela and spin](<http://spinroot.com/spin/whatispin.html>), as described in the LWN article [ Using Promela and Spin to verify parallel algorithms](<http://lwn.net/Articles/243851/>). This article presents a series of seven increasingly realistic Promela models, the last of which passes, consuming about 40GB of main memory for the state space. 

**Quick Quiz 1** : Yeah, that's great!!! Now, just what am I supposed to do if I don't happen to have a machine with 40GB of main memory??? 

More important, Promela and Spin did find a very subtle bug for me!!! 

This article is organized as follows: 

  1. Introduction to Preemptable RCU and dynticks
     1. Task Interface
     2. Interrupt Interface
     3. Grace-Period Interface
  2. Validating Preemptable RCU and dynticks
     1. Basic Model
     2. Validating Safety
     3. Validating Liveness
     4. Interrupts
     5. Validating Interrupt Handlers
     6. Validating Nested Interrupt Handlers
     7. Validating NMI Handlers

These sections are followed by conclusions and answers to the Quick Quizzes. 

###  Introduction to Preemptable RCU and dynticks

The per-CPU `dynticks_progress_counter` variable is central to the interface between dynticks and preemptable RCU. This variable has an even value whenever the corresponding CPU is in dynticks-idle mode, and an odd value otherwise. A CPU exits dynticks-idle mode for the following three reasons: 

  1. to start running a task,
  2. when entering the outermost of a possibly nested set of interrupt handlers, and
  3. when entering an NMI handler. 

Preemptable RCU's grace-period machinery samples the value of the `dynticks_progress_counter` variable in order to determine when a dynticks-idle CPU may safely be ignored. 

The following three sections give an overview of the task interface, the interrupt/NMI interface, and the use of the `dynticks_progress_counter` variable by the grace-period machinery. 

####  Task Interface

When a given CPU enters dynticks-idle mode because it has no more tasks to run, it invokes `rcu_enter_nohz()`: 

```
1 static inline void rcu_enter_nohz(void)
      2 {
      3   mb();
      4   __get_cpu_var(dynticks_progress_counter)++;
      5   WARN_ON(__get_cpu_var(dynticks_progress_counter) & 0x1);
      6 }
```

This function simply increments `dynticks_progress_counter` and checks that the result is even, but first executing a memory barrier to ensure that any other CPU that sees the new value of `dynticks_progress_counter` will also see the completion of any prior RCU read-side critical sections. 

Similarly, when a CPU that is in dynticks-idle mode prepares to start executing a newly runnable task, it invokes `rcu_exit_nohz`: 

```
1 static inline void rcu_exit_nohz(void)
      2 {
      3   __get_cpu_var(dynticks_progress_counter)++;
      4   mb();
      5   WARN_ON(!(__get_cpu_var(dynticks_progress_counter) & 0x1));
      6 }
```

This function again increments `dynticks_progress_counter`, but follows it with a memory barrier to ensure that if any other CPU sees the result of any subsequent RCU read-side critical section, then that other CPU will also see the incremented value of `dynticks_progress_counter`. Finally, `rcu_exit_nohz()` checks that the result of the increment is an odd value. 

The `rcu_enter_nohz()` and `rcu_exit_nohz` functions handle the case where a CPU enters and exits dynticks-idle mode due to task execution, but does not handle interrupts, which are covered in the following section. 

####  Interrupt Interface

The `rcu_irq_enter()` and `rcu_irq_exit()` functions handle interrupt/NMI entry and exit, respectively. Of course, nested interrupts must also be properly accounted for. The possibility of nested interrupts is handled by a second per-CPU variable, `rcu_update_flag`, which is incremented upon entry to an interrupt or NMI handler (in `rcu_irq_enter()`) and is decremented upon exit (in `rcu_irq_exit()`). In addition, the pre-existing `in_interrupt()` primitive is used to distinguish between an outermost or a nested interrupt/NMI. 

Interrupt entry is handled by the `rcu_irq_enter` shown below: 

```
1 void rcu_irq_enter(void)
      2 {
      3   int cpu = smp_processor_id();
      4 
      5   if (per_cpu(rcu_update_flag, cpu))
      6     per_cpu(rcu_update_flag, cpu)++;
      7   if (!in_interrupt() &&
      8       (per_cpu(dynticks_progress_counter, cpu) & 0x1) == 0) {
      9     per_cpu(dynticks_progress_counter, cpu)++;
     10     smp_mb();
     11     per_cpu(rcu_update_flag, cpu)++;
     12   }
     13 }
```

**Quick Quiz 2** : Why not simply increment `rcu_update_flag`, and then only increment `dynticks_progress_counter` if the old value of `rcu_update_flag` was zero??? 

**Quick Quiz 3** : But if line 7 finds that we are the outermost interrupt, wouldn't we _always_ need to increment `dynticks_progress_counter`? 

Line 3 fetches the current CPU's number, while lines 4 and 5 increment the `rcu_update_flag` nesting counter if it is already non-zero. Lines 6 and 7 check to see whether we are the outermost level of interrupt, and, if so, whether `dynticks_progress_counter` needs to be incremented. If so, line 9 increments `dynticks_progress_counter`, line 10 executes a memory barrier, and line 11 increments `rcu_update_flag`. As with `rcu_exit_nohz()`, the memory barrier ensures that any other CPU that sees the effects of an RCU read-side critical section in the interrupt handler (following the `rcu_irq_enter()` invocation) will also see the increment of `dynticks_progress_counter`. 

Interrupt entry is handled similarly by `rcu_irq_exit()`: 

```
1 void rcu_irq_exit(void)
      2 {
      3   int cpu = smp_processor_id();
      4 
      5   if (per_cpu(rcu_update_flag, cpu)) {
      6     if (--per_cpu(rcu_update_flag, cpu))
      7       return;
      8     WARN_ON(in_interrupt());
      9     smp_mb();
     10     per_cpu(dynticks_progress_counter, cpu)++;
     11     WARN_ON(per_cpu(dynticks_progress_counter, cpu) & 0x1);
     12   }
     13 }
```

Line 3 fetches the current CPU's number, as before. Line 5 checks to see if the `rcu_update_flag` is non-zero, returning immediately (via falling off the end of the function) if not. Otherwise, lines 6 through 11 come into play. Line 6 decrements `rcu_update_flag`, returning if the result is not zero. Line 8 verifies that we are indeed leaving the outermost level of nested interrupts, line 9 executes a memory barrier, line 10 increments `dynticks_progress_counter`, and line 11 verifies that this variable is now even. As with `rcu_enter_nohz()`, the memory barrier ensures that any other CPU that sees the increment of `dynticks_progress_counter` will also see the effects of an RCU read-side critical section in the interrupt handler (preceding the `rcu_irq_enter()` invocation). 

These two sections have described how the `dynticks_progress_counter` variable is maintained during entry to and exit from dynticks-idle mode, both by tasks and by interrupts and NMIs. The following section describes how this variable is used by preemptable RCU's grace-period machinery. 

####  Grace-Period Interface

Of the four preemptable RCU grace-period states shown below (taken from [ The Design of Preemptable Read-Copy Update](<http://lwn.net/Articles/253651/>)), only the `rcu_try_flip_waitack_state()` and `rcu_try_flip_waitmb_state()` states need to wait for other CPUs to respond. 

> ![Preemptable RCU State Diagram](https://static.lwn.net/images/ns/kernel/RCUpreemptStates.png)

Of course, if a given CPU is in dynticks-idle state, we shouldn't wait for it. Therefore, just before entering one of these two states, the preceding state takes a snapshot of each CPU's `dynticks_progress_counter` variable, placing the snapshot in another per-CPU variable, `rcu_dyntick_snapshot`. This is accomplished by invoking `dyntick_save_progress_counter`, shown below: 

```
1 static void dyntick_save_progress_counter(int cpu)
      2 {
      3   per_cpu(rcu_dyntick_snapshot, cpu) =
      4     per_cpu(dynticks_progress_counter, cpu);
      5 }
```

The `rcu_try_flip_waitack_state()` state invokes `rcu_try_flip_waitack_needed()`, shown below: 

```
1 static inline int
      2 rcu_try_flip_waitack_needed(int cpu)
      3 {
      4   long curr;
      5   long snap;
      6 
      7   curr = per_cpu(dynticks_progress_counter, cpu);
      8   snap = per_cpu(rcu_dyntick_snapshot, cpu);
      9   smp_mb(); /* force ordering with cpu entering/leaving dynticks. */
     10   if ((curr == snap) && ((curr & 0x1) == 0))
     11     return 0;
     12   if ((curr - snap) > 2 || (snap & 0x1) == 0)
     13     return 0;
     14   return 1;
     15 }
```

Lines 7 and 8 pick up current and snapshot versions of `dynticks_progress_counter`, respectively. The memory barrier on line ensures that the counter checks in the later `rcu_try_flip_waitzero_state` follow the fetches of these counters. Lines 10 and 11 return zero (meaning no communication with the specified CPU is required) if that CPU has remained in dynticks-idle state since the time that the snapshot was taken. Similarly, lines 12 and 13 return zero if that CPU was initially in dynticks-idle state or if it has completely passed through a dynticks-idle state. In both these cases, there is no way that that CPU could have retained the old value of the grace-period counter. If neither of these conditions hold, line 14 returns one, meaning that the CPU needs to explicitly respond. 

For its part, the `rcu_try_flip_waitmb_state` state invokes `rcu_try_flip_waitmb_needed()`, shown below: 

```
1 static inline int
      2 rcu_try_flip_waitmb_needed(int cpu)
      3 {
      4   long curr;
      5   long snap;
      6 
      7   curr = per_cpu(dynticks_progress_counter, cpu);
      8   snap = per_cpu(rcu_dyntick_snapshot, cpu);
      9   smp_mb(); /* force ordering with cpu entering/leaving dynticks. */
     10   if ((curr == snap) && ((curr & 0x1) == 0))
     11     return 0;
     12   if (curr != snap)
     13     return 0;
     14   return 1;
     15 }
```

This is quite similar to `rcu_try_flip_waitack_needed`, the difference being in lines 12 and 13, because any transition either to or from dynticks-idle state executes the memory barrier needed by the `rcu_try_flip_waitmb_state()` state. 

**Quick Quiz 4** : Can you spot any bugs in any of the code in this section? 

We now have seen all the code involved in the interface between RCU and the dynticks-idle state. The next section builds up the Promela model used to validate this code. 

###  Validating Preemptable RCU and dynticks

This section develops a Promela model for the interface between dynticks and RCU step by step, with each of the following sections illustrating one step, starting with the process-level code, adding assertions, interrupts, and finally NMIs. 

####  Basic Model

This section translates the process-level dynticks entry/exit code and the grace-period processing into Promela. We start with `rcu_exit_nohz()` and `rcu_enter_nohz()` from the 2.6.25-rc4 kernel, placing these in a single Promela process that models exiting and entering dynticks-idle mode in a loop as follows: 

```
1 proctype dyntick_nohz()
      2 {
      3   byte tmp;
      4   byte i = 0;
      5 
      6   do
      7   :: i >= MAX_DYNTICK_LOOP_NOHZ -> break;
      8   :: i < MAX_DYNTICK_LOOP_NOHZ ->
      9     tmp = dynticks_progress_counter;
     10     atomic {
     11       dynticks_progress_counter = tmp + 1;
     12       assert((dynticks_progress_counter & 1) == 1);
     13     }
     14     tmp = dynticks_progress_counter;
     15     atomic {
     16       dynticks_progress_counter = tmp + 1;
     17       assert((dynticks_progress_counter & 1) == 0);
     18     }
     19     i++;
     20   od;
     21 }
```

Lines 6 and 20 define a loop. Line 7 exits the loop once the loop counter `i` has exceeded the limit `MAX_DYNTICK_LOOP_NOHZ`. Line 8 tells the loop construct to execute lines 9-19 for each pass through the loop. Because the conditionals on lines 7 and 8 are exclusive of each other, the normal Promela random selection of true conditions is disabled. Lines 9 and 11 model `rcu_exit_nohz()`'s non-atomic increment of `dynticks_progress_counter`, while line 12 models the `WARN_ON()`. The `atomic` construct simply reduces the Promela state space, given that the `WARN_ON()` is not strictly speaking part of the algorithm. Lines 14-18 similarly models the increment and `WARN_ON()` for `rcu_enter_nohz()`. Finally, line 19 increments the loop counter. 

**Quick Quiz 5** : Why isn't the memory barrier in `rcu_exit_nohz()` and `rcu_enter_nohz()` modeled in Promela? 

**Quick Quiz 6** : Isn't it a bit strange to model `rcu_exit_nohz()` followed by `rcu_enter_nohz()`? Wouldn't it be more natural to instead model entry before exit? 

Each pass through the loop therefore models a CPU exiting dynticks-idle mode (for example, starting to execute a task), then re-entering dynticks-idle mode (for example, that same task blocking). 

The next step is to model the interface to RCU's grace-period processing. For this, we need to model `dyntick_save_progress_counter()`, `rcu_try_flip_waitack_needed()`, `rcu_try_flip_waitmb_needed()`, as well as portions of `rcu_try_flip_waitack()` and `rcu_try_flip_waitmb()`, all from the 2.6.25-rc4 kernel. The following `grace_period()` Promela process models these functions as they would be invoked during a single pass through preemptable RCU's grace-period processing. 

```
1 proctype grace_period()
      2 {
      3   byte curr;
      4   byte snap;
      5 
      6   atomic {
      7     printf("MAX_DYNTICK_LOOP_NOHZ = %d\n", MAX_DYNTICK_LOOP_NOHZ);
      8     snap = dynticks_progress_counter;
      9   }
     10   do
     11   :: 1 ->
     12     atomic {
     13       curr = dynticks_progress_counter;
     14       if
     15       :: (curr == snap) && ((curr & 1) == 0) ->
     16         break;
     17       :: (curr - snap) > 2 || (snap & 1) == 0 ->
     18         break;
     19       :: 1 -> skip;
     20       fi;
     21     }
     22   od;
     23   snap = dynticks_progress_counter;
     24   do
     25   :: 1 ->
     26     atomic {
     27       curr = dynticks_progress_counter;
     28       if
     29       :: (curr == snap) && ((curr & 1) == 0) ->
     30         break;
     31       :: (curr != snap) ->
     32         break;
     33       :: 1 -> skip;
     34       fi;
     35     }
     36   od;
     37 }
```

Lines 6-9 print out the loop limit (but only into the .trail file in case of error) and model a line of code from `rcu_try_flip_idle()` and its call to `dyntick_save_progress_counter()`, which takes a snapshot of the current CPU's `dynticks_progress_counter` variable. These two lines are executed atomically to reduce state space. 

Lines 10-22 model the relevant code in `rcu_try_flip_waitack()` and its call to `rcu_try_flip_waitack_needed()`. This loop is modeling the grace-period state machine waiting for a counter-flip acknowledgment from each CPU, but only that part that interacts with dynticks-idle CPUs. 

Line 23 models a line from `rcu_try_flip_waitzero()` and its call to `dyntick_save_progress_counter()`, again taking a snapshot of the CPU's `dynticks_progress_counter` variable. 

Finally, lines 24-36 model the relevant code in `rcu_try_flip_waitack()` and its call to `rcu_try_flip_waitack_needed()`. This loop is modeling the grace-period state-machine waiting for each CPU to execute a memory barrier, but again only that part that interacts with dynticks-idle CPUs. 

**Quick Quiz 7** : Wait a minute! In the Linux kernel, both `dynticks_progress_counter` and `rcu_dyntick_snapshot` are per-CPU variables. So why are they instead being modeled as single global variables? 

The resulting [model](<https://lwn.net/Articles/279091/>), when run with the [runspin.sh](<https://lwn.net/Articles/279092/>) script, generates 691 states and passes without errors, which is not at all surprising given that it completely lacks the assertions that could find failures. The next section therefore adds safety assertions. 

####  Validating Safety

A safe RCU implementation must never permit a grace period to complete before the completion of any RCU readers that started before the start of the grace period. This is modeled by a `grace_period_state` variable that can take on three states as follows: 

```
1 #define GP_IDLE    0
      2 #define GP_WAITING  1
      3 #define GP_DONE    2
      4 byte grace_period_state = GP_DONE;
```

The `grace_period()` process sets this variable as it progresses through the grace-period phases, as shown below: 

```
1 proctype grace_period()
      2 {
      3   byte curr;
      4   byte snap;
      5 
      6   grace_period_state = GP_IDLE;
      7   atomic {
      8     printf("MAX_DYNTICK_LOOP_NOHZ = %d\n", MAX_DYNTICK_LOOP_NOHZ);
      9     snap = dynticks_progress_counter;
     10     grace_period_state = GP_WAITING;
     11   }
     12   do
     13   :: 1 ->
     14     atomic {
     15       curr = dynticks_progress_counter;
     16       if
     17       :: (curr == snap) && ((curr & 1) == 0) ->
     18         break;
     19       :: (curr - snap) > 2 || (snap & 1) == 0 ->
     20         break;
     21       :: 1 -> skip;
     22       fi;
     23     }
     24   od;
     25   grace_period_state = GP_DONE;
     26   grace_period_state = GP_IDLE;
     27   atomic {
     28     snap = dynticks_progress_counter;
     29     grace_period_state = GP_WAITING;
     30   }
     31   do
     32   :: 1 ->
     33     atomic {
     34       curr = dynticks_progress_counter;
     35       if
     36       :: (curr == snap) && ((curr & 1) == 0) ->
     37         break;
     38       :: (curr != snap) ->
     39         break;
     40       :: 1 -> skip;
     41       fi;
     42     }
     43   od;
     44   grace_period_state = GP_DONE;
     45 }
```

**Quick Quiz 8** : Given there are a pair of back-to-back changes to `grace_period_state` on lines 25 and 26, how can we be sure that line 25's changes won't be lost? 

Lines 6, 10, 25, 26, 29, and 44 update this variable (combining atomically with algorithmic operations where feasible) to allow the `dyntick_nohz()` process to validate the basic RCU safety property. The form of this validation is to assert that the value of the `grace_period_state` variable cannot jump from `GP_IDLE` to `GP_DONE` during a time period over which RCU readers could plausibly persist. 

The `dyntick_nohz()` Promela process implements this validation as shown below: 

```
1 proctype dyntick_nohz()
      2 {
      3   byte tmp;
      4   byte i = 0;
      5   bit old_gp_idle;
      6 
      7   do
      8   :: i >= MAX_DYNTICK_LOOP_NOHZ -> break;
      9   :: i < MAX_DYNTICK_LOOP_NOHZ ->
     10     tmp = dynticks_progress_counter;
     11     atomic {
     12       dynticks_progress_counter = tmp + 1;
     13       old_gp_idle = (grace_period_state == GP_IDLE);
     14       assert((dynticks_progress_counter & 1) == 1);
     15     }
     16     atomic {
     17       tmp = dynticks_progress_counter;
     18       assert(!old_gp_idle || grace_period_state != GP_DONE);
     19     }
     20     atomic {
     21       dynticks_progress_counter = tmp + 1;
     22       assert((dynticks_progress_counter & 1) == 0);
     23     }
     24     i++;
     25   od;
     26 }
```

Line 13 sets a new `old_gp_idle` flag if the value of the `grace_period_state` variable is `GP_IDLE` at the beginning of task execution, and the assertion at line 18 fires if the `grace_period_state` variable has advanced to `GP_DONE` during task execution, which would be illegal given that a single RCU read-side critical section could span the entire intervening time period. 

The resulting [model](<https://lwn.net/Articles/279094/>), when run with the runspin.sh script, generates 964 states and passes without errors, which is reassuring. That said, although safety is critically important, it is also quite important to avoid indefinitely stalling grace periods. The next section therefore covers validating liveness. 

####  Validating Liveness

Although liveness can be difficult to prove, there is a simple trick that applies here. The first step is to make `dyntick_nohz()` indicate that it is done via a `dyntick_nohz_done` variable, as shown on line 26 of the following: 

```
1 proctype dyntick_nohz()
      2 {
      3   byte tmp;
      4   byte i = 0;
      5   bit old_gp_idle;
      6 
      7   do
      8   :: i >= MAX_DYNTICK_LOOP_NOHZ -> break;
      9   :: i < MAX_DYNTICK_LOOP_NOHZ ->
     10     tmp = dynticks_progress_counter;
     11     atomic {
     12       dynticks_progress_counter = tmp + 1;
     13       old_gp_idle = (grace_period_state == GP_IDLE);
     14       assert((dynticks_progress_counter & 1) == 1);
     15     }
     16     atomic {
     17       tmp = dynticks_progress_counter;
     18       assert(!old_gp_idle || grace_period_state != GP_DONE);
     19     }
     20     atomic {
     21       dynticks_progress_counter = tmp + 1;
     22       assert((dynticks_progress_counter & 1) == 0);
     23     }
     24     i++;
     25   od;
     26   dyntick_nohz_done = 1;
     27 }
```

With this variable in place, we can add assertions to `grace_period()` to check for unnecessary blockage as follows: 

```
1 proctype grace_period()
      2 {
      3   byte curr;
      4   byte snap;
      5   bit shouldexit;
      6 
      7   grace_period_state = GP_IDLE;
      8   atomic {
      9     printf("MAX_DYNTICK_LOOP_NOHZ = %d\n", MAX_DYNTICK_LOOP_NOHZ);
     10     shouldexit = 0;
     11     snap = dynticks_progress_counter;
     12     grace_period_state = GP_WAITING;
     13   }
     14   do
     15   :: 1 ->
     16     atomic {
     17       assert(!shouldexit);
     18       shouldexit = dyntick_nohz_done;
     19       curr = dynticks_progress_counter;
     20       if
     21       :: (curr == snap) && ((curr & 1) == 0) ->
     22         break;
     23       :: (curr - snap) > 2 || (snap & 1) == 0 ->
     24         break;
     25       :: else -> skip;
     26       fi;
     27     }
     28   od;
     29   grace_period_state = GP_DONE;
     30   grace_period_state = GP_IDLE;
     31   atomic {
     32     shouldexit = 0;
     33     snap = dynticks_progress_counter;
     34     grace_period_state = GP_WAITING;
     35   }
     36   do
     37   :: 1 ->
     38     atomic {
     39       assert(!shouldexit);
     40       shouldexit = dyntick_nohz_done;
     41       curr = dynticks_progress_counter;
     42       if
     43       :: (curr == snap) && ((curr & 1) == 0) ->
     44         break;
     45       :: (curr != snap) ->
     46         break;
     47       :: else -> skip;
     48       fi;
     49     }
     50   od;
     51   grace_period_state = GP_DONE;
     52 }
```

We have added the `shouldexit` variable on line 5, which we initialize to zero on line 10\. Line 17 asserts that `shouldexit` is not set, while line 18 sets `shouldexit` to the `dyntick_nohz_done` variable maintained by `dyntick_nohz()`. This assertion will therefore trigger if we attempt to take more than one pass through the wait-for-counter-flip-acknowledgment loop after `dyntick_nohz()` has completed execution. After all, if `dyntick_nohz()` is done, then there cannot be any more state changes to force us out of the loop, so going through twice in this state means an infinite loop, which in turn means no end to the grace period. 

Lines 32, 39, and 40 operate in a similar manner for the second (memory-barrier) loop. 

However, running this [model](<https://lwn.net/Articles/279096/>) results in failure, as line 23 is checking that the wrong variable is even. Upon failure, `spin` writes out a ["trail"](<https://lwn.net/Articles/279097/>) file, which records the sequence of states that lead to the failure. Use the `spin -t -p -g -l dyntickRCU-base-sl-busted.spin` command to cause `spin` to retrace this sequence of states, [ printing the statements executed and the values of variables](<https://lwn.net/Articles/279098/>). Note that the line numbers do not match the listing above due to the fact that spin takes both functions in a single file. However, the line numbers _do_ match the full model. 

We see that the `dyntick_nohz()` process completed at step 34 (search for "34:"), but that the `grace_period()` process nonetheless failed to exit the loop. The value of `curr` is `6` (see step 35) and that the value of `snap` is `5` (see step 17). Therefore the first condition on line 21 above does not hold because `curr != snap`, and the second condition on line 23 does not hold either because `snap` is odd and because `curr` is only one greater than `snap`. 

So one of these two conditions has to be incorrect. Referring to the comment block in `rcu_try_flip_waitack_needed()` for the first condition: 

```
/*
    	 * If the CPU remained in dynticks mode for the entire time
    	 * and didn't take any interrupts, NMIs, SMIs, or whatever,
    	 * then it cannot be in the middle of an rcu_read_lock(), so
    	 * the next rcu_read_lock() it executes must use the new value
    	 * of the counter.  So we can safely pretend that this CPU
    	 * already acknowledged the counter.
    	 */
```

The first condition does match this, because if `curr == snap` and if `curr` is even, then the corresponding CPU has been in dynticks-idle mode the entire time, as required. So let's look at the comment block for the second condition: 

```
/*
    	 * If the CPU passed through or entered a dynticks idle phase with
    	 * no active irq handlers, then, as above, we can safely pretend
    	 * that this CPU already acknowledged the counter.
    	 */
```

The first part of the condition is correct, because if `curr` and `snap` differ by two, there will be at least one even number in between, corresponding to having passed completely through a dynticks-idle phase. However, the second part of the condition corresponds to having _started_ in dynticks-idle mode, not having _finished_ in this mode. We therefore need to be testing `curr` rather than `snap` for being an even number. 

The corrected C code is as follows: 

```
1 static inline int
      2 rcu_try_flip_waitack_needed(int cpu)
      3 {
      4   long curr;
      5   long snap;
      6 
      7   curr = per_cpu(dynticks_progress_counter, cpu);
      8   snap = per_cpu(rcu_dyntick_snapshot, cpu);
      9   smp_mb(); /* force ordering with cpu entering/leaving dynticks. */
     10   if ((curr == snap) && ((curr & 0x1) == 0))
     11     return 0;
     12   if ((curr - snap) > 2 || (curr & 0x1) == 0)
     13     return 0;
     14   return 1;
     15 }
```

Making the corresponding correction in the [model](<https://lwn.net/Articles/279100/>) results in a correct validation with 661 states that passes without errors. However, it is worth noting that the first version of the liveness validation failed to catch this bug, due to a bug in the liveness validation itself. This liveness-validation bug was located by inserting an infinite loop in the `grace_period()` process, and noting that the liveness-validation code failed to detect this problem! 

We have now successfully validated both safety and liveness conditions, but only for processes running and blocking. We also need to handle interrupts, a task taken up in the next section. 

####  Interrupts

There are a couple of ways to model interrupts in Promela: 

  1. using C-preprocessor tricks to insert the interrupt handler between each and every statement of the `dynticks_nohz()` process, or 
  2. modeling the interrupt handler with a separate process. 

A bit of thought indicated that the second approach would have a smaller state space, though it requires that the interrupt handler somehow run atomically with respect to the `dynticks_nohz()` process, but not with respect to the `grace_period()` process. 

Fortunately, it turns out that Promela permits you to branch out of atomic statements. This trick allows us to have the interrupt handler set a flag, and recode `dynticks_nohz()` to atomically check this flag and execute only when the flag is not set. This can be accomplished with a C-preprocessor macro that takes a label and a Promela statement as follows: 

```
1 #define EXECUTE_MAINLINE(label, stmt) \
      2 label: skip; \
      3     atomic { \
      4       if \
      5       :: in_dyntick_irq -> goto label; \
      6       :: else -> stmt; \
      7       fi; \
      8     } \
```

One might use this macro as follows: 

```
EXECUTE_MAINLINE(stmt1, tmp = dynticks_progress_counter)
```

Line 2 of the macro creates the specified statement label. Lines 3-8 are an atomic block that tests the `in_dyntick_irq` variable, and if this variable is set (indicating that the interrupt handler is active), branches out of the atomic block back to the label. Otherwise, line 6 executes the specified statement. The overall effect is that mainline execution stalls any time an interrupt is active, as required. 

####  Validating Interrupt Handlers

The first step is to convert `dyntick_nohz()` to `EXECUTE_MAINLINE()` form, as follows: 

```
1 proctype dyntick_nohz()
      2 {
      3   byte tmp;
      4   byte i = 0;
      5   bit old_gp_idle;
      6 
      7   do
      8   :: i >= MAX_DYNTICK_LOOP_NOHZ -> break;
      9   :: i < MAX_DYNTICK_LOOP_NOHZ ->
     10     EXECUTE_MAINLINE(stmt1, tmp = dynticks_progress_counter)
     11     EXECUTE_MAINLINE(stmt2,
     12          dynticks_progress_counter = tmp + 1;
     13          old_gp_idle = (grace_period_state == GP_IDLE);
     14          assert((dynticks_progress_counter & 1) == 1))
     15     EXECUTE_MAINLINE(stmt3,
     16       tmp = dynticks_progress_counter;
     17       assert(!old_gp_idle || grace_period_state != GP_DONE))
     18     EXECUTE_MAINLINE(stmt4,
     19       dynticks_progress_counter = tmp + 1;
     20       assert((dynticks_progress_counter & 1) == 0))
     21     i++;
     22   od;
     23   dyntick_nohz_done = 1;
     24 }
```

**Quick Quiz 9** : But what would you do if you needed the statements in a single `EXECUTE_MAINLINE()` group to execute non-atomically? 

**Quick Quiz 10** : But what if the `dynticks_nohz()` process had "if" or "do" statements with conditions, where the statement bodies of these constructs needed to execute non-atomically? 

It is important to note that when a group of statements is passed to `EXECUTE_MAINLINE()`, as in lines 11-14, all statements in that group execute atomically. 

The next step is to write a `dyntick_irq()` process to model an interrupt handler: 

```
1 proctype dyntick_irq()
      2 {
      3   byte tmp;
      4   byte i = 0;
      5   bit old_gp_idle;
      6 
      7   do
      8   :: i >= MAX_DYNTICK_LOOP_IRQ -> break;
      9   :: i < MAX_DYNTICK_LOOP_IRQ ->
     10     in_dyntick_irq = 1;
     11     if
     12     :: rcu_update_flag > 0 ->
     13       tmp = rcu_update_flag;
     14       rcu_update_flag = tmp + 1;
     15     :: else -> skip;
     16     fi;
     17     if
     18     :: !in_interrupt && (dynticks_progress_counter & 1) == 0 ->
     19       tmp = dynticks_progress_counter;
     20       dynticks_progress_counter = tmp + 1;
     21       tmp = rcu_update_flag;
     22       rcu_update_flag = tmp + 1;
     23     :: else -> skip;
     24     fi;
     25     tmp = in_interrupt;
     26     in_interrupt = tmp + 1;
     27     old_gp_idle = (grace_period_state == GP_IDLE);
     28     assert(!old_gp_idle || grace_period_state != GP_DONE);
     29     tmp = in_interrupt;
     30     in_interrupt = tmp - 1;
     31     if
     32     :: rcu_update_flag != 0 ->
     33       tmp = rcu_update_flag;
     34       rcu_update_flag = tmp - 1;
     35       if
     36       :: rcu_update_flag == 0 ->
     37         tmp = dynticks_progress_counter;
     38         dynticks_progress_counter = tmp + 1;
     39       :: else -> skip;
     40       fi;
     41     :: else -> skip;
     42     fi;
     43     atomic {
     44       in_dyntick_irq = 0;
     45       i++;
     46     }
     47   od;
     48   dyntick_irq_done = 1;
     49 }
```

**Quick Quiz 11** : Why are lines 44 and 45 (the `in_dyntick_irq = 0;` and the `i++;`) executed atomically? 

**Quick Quiz 12** : What property of interrupts is this `dynticks_irq()` process unable to model? 

The loop from line 7-47 models up to `MAX_DYNTICK_LOOP_IRQ` interrupts, with lines 8 and 9 forming the loop condition and line 45 incrementing the control variable. Line 10 tells `dyntick_nohz()` that an interrupt handler is running, and line 44 tells `dyntick_nohz()` that this handler has completed. Line 48 is used for liveness validation, much as is the corresponding line of `dyntick_nohz()`. 

Lines 11-24 model `rcu_irq_enter()`, and lines 25 and 26 model the relevant snippet of `__irq_enter()`. Lines 27 and 28 validate safety in much the same manner as do the corresponding lines of `dynticks_nohz()`. Lines 29 and 30 model the relevant snippet of `__irq_exit()`, and finally lines 31-42 model `rcu_irq_exit()`. 

```
1 proctype grace_period()
      2 {
      3   byte curr;
      4   byte snap;
      5   bit shouldexit;
      6 
      7   grace_period_state = GP_IDLE;
      8   atomic {
      9     printf("MAX_DYNTICK_LOOP_NOHZ = %d\n", MAX_DYNTICK_LOOP_NOHZ);
     10     printf("MAX_DYNTICK_LOOP_IRQ = %d\n", MAX_DYNTICK_LOOP_IRQ);
     11     shouldexit = 0;
     12     snap = dynticks_progress_counter;
     13     grace_period_state = GP_WAITING;
     14   }
     15   do
     16   :: 1 ->
     17     atomic {
     18       assert(!shouldexit);
     19       shouldexit = dyntick_nohz_done && dyntick_irq_done;
     20       curr = dynticks_progress_counter;
     21       if
     22       :: (curr == snap) && ((curr & 1) == 0) ->
     23         break;
     24       :: (curr - snap) > 2 || (curr & 1) == 0 ->
     25         break;
     26       :: else -> skip;
     27       fi;
     28     }
     29   od;
     30   grace_period_state = GP_DONE;
     31   grace_period_state = GP_IDLE;
     32   atomic {
     33     shouldexit = 0;
     34     snap = dynticks_progress_counter;
     35     grace_period_state = GP_WAITING;
     36   }
     37   do
     38   :: 1 ->
     39     atomic {
     40       assert(!shouldexit);
     41       shouldexit = dyntick_nohz_done && dyntick_irq_done;
     42       curr = dynticks_progress_counter;
     43       if
     44       :: (curr == snap) && ((curr & 1) == 0) ->
     45         break;
     46       :: (curr != snap) ->
     47         break;
     48       :: else -> skip;
     49       fi;
     50     }
     51   od;
     52   grace_period_state = GP_DONE;
     53 }
```

The implementation of `grace_period()` is very similar to the earlier one. The only changes are the addition of line 10 to add the new interrupt-count parameter and changes to lines 19 and 41 to add the new `dyntick_irq_done` variable to the liveness checks. 

This [model](<https://lwn.net/Articles/279103/>) results in a correct validation with roughly half a million states, passing without errors. However, this version of the model does not handle nested interrupts. This topic is taken up in the next section. 

####  Validating Nested Interrupt Handlers

Nested interrupt handlers may be modeled by splitting the body of the loop in `dyntick_irq()` as follows: 

```
1 proctype dyntick_irq()
      2 {
      3   byte tmp;
      4   byte i = 0;
      5   byte j = 0;
      6   bit old_gp_idle;
      7   bit outermost;
      8 
      9   do
     10   :: i >= MAX_DYNTICK_LOOP_IRQ && j >= MAX_DYNTICK_LOOP_IRQ -> break;
     11   :: i < MAX_DYNTICK_LOOP_IRQ ->
     12     atomic {
     13       outermost = (in_dyntick_irq == 0);
     14       in_dyntick_irq = 1;
     15     }
     16     if
     17     :: rcu_update_flag > 0 ->
     18       tmp = rcu_update_flag;
     19       rcu_update_flag = tmp + 1;
     20     :: else -> skip;
     21     fi;
     22     if
     23     :: !in_interrupt && (dynticks_progress_counter & 1) == 0 ->
     24       tmp = dynticks_progress_counter;
     25       dynticks_progress_counter = tmp + 1;
     26       tmp = rcu_update_flag;
     27       rcu_update_flag = tmp + 1;
     28     :: else -> skip;
     29     fi;
     30     tmp = in_interrupt;
     31     in_interrupt = tmp + 1;
     32     atomic {
     33       if
     34       :: outermost ->
     35         old_gp_idle = (grace_period_state == GP_IDLE);
     36       :: else -> skip;
     37       fi;
     38     }
     39     i++;
     40   :: j < i ->
     41     atomic {
     42       if
     43       :: j + 1 == i ->
     44         assert(!old_gp_idle ||
     45                grace_period_state != GP_DONE);
     46       :: else -> skip;
     47       fi;
     48     }
     49     tmp = in_interrupt;
     50     in_interrupt = tmp - 1;
     51     if
     52     :: rcu_update_flag != 0 ->
     53       tmp = rcu_update_flag;
     54       rcu_update_flag = tmp - 1;
     55       if
     56       :: rcu_update_flag == 0 ->
     57         tmp = dynticks_progress_counter;
     58         dynticks_progress_counter = tmp + 1;
     59       :: else -> skip;
     60       fi;
     61     :: else -> skip;
     62     fi;
     63     atomic {
     64       j++;
     65       in_dyntick_irq = (i != j);
     66     }
     67   od;
     68   dyntick_irq_done = 1;
     69 }
```

This is similar to the earlier `dynticks_irq()` process. It adds a second counter variable `j` on line 5, so that `i` counts entries to interrupt handlers and `j` counts exits. The `outermost` variable on line 7 helps determine when the `grace_period_state` variable needs to be sampled for the safety checks. The loop-exit check on line 10 is updated to require that the specified number of interrupt handlers are exited as well as entered, and the increment of `i` is moved to line 39, which is the end of the interrupt-entry model. Lines 12-15 set the `outermost` variable to indicate whether this is the outermost of a set of nested interrupts and to set the `in_dyntick_irq` variable that is used by the `dyntick_nohz()` process. Lines 32-37 capture the state of the `grace_period_state` variable, but only when in the outermost interrupt handler. 

Line 40 has the do-loop conditional for interrupt-exit modeling: as long as we have exited fewer interrupts than we have entered, it is legal to exit another interrupt. Lines 41-48 check the safety criterion, but only if we are exiting from the outermost interrupt level. Finally, lines 63-66 increment the interrupt-exit count `j` and, if this is the outermost interrupt level, clears `in_dyntick_irq`. 

This [model](<https://lwn.net/Articles/279104/>) results in a correct validation with a bit more than half a million states, passing without errors. However, this version of the model does not handle NMIs, which are taken up in the nest section. 

####  Validating NMI Handlers

We take the same general approach for NMIs as we do for interrupts, keeping in mind that NMIs do not nest. This results in a `dyntick_nmi()` process as follows: 

```
1 proctype dyntick_nmi()
      2 {
      3   byte tmp;
      4   byte i = 0;
      5   bit old_gp_idle;
      6 
      7   do
      8   :: i >= MAX_DYNTICK_LOOP_NMI -> break;
      9   :: i < MAX_DYNTICK_LOOP_NMI ->
     10     in_dyntick_nmi = 1;
     11     if
     12     :: rcu_update_flag > 0 ->
     13       tmp = rcu_update_flag;
     14       rcu_update_flag = tmp + 1;
     15     :: else -> skip;
     16     fi;
     17     if
     18     :: !in_interrupt && (dynticks_progress_counter & 1) == 0 ->
     19       tmp = dynticks_progress_counter;
     20       dynticks_progress_counter = tmp + 1;
     21       tmp = rcu_update_flag;
     22       rcu_update_flag = tmp + 1;
     23     :: else -> skip;
     24     fi;
     25     tmp = in_interrupt;
     26     in_interrupt = tmp + 1;
     27     old_gp_idle = (grace_period_state == GP_IDLE);
     28     assert(!old_gp_idle || grace_period_state != GP_DONE);
     29     tmp = in_interrupt;
     30     in_interrupt = tmp - 1;
     31     if
     32     :: rcu_update_flag != 0 ->
     33       tmp = rcu_update_flag;
     34       rcu_update_flag = tmp - 1;
     35       if
     36       :: rcu_update_flag == 0 ->
     37         tmp = dynticks_progress_counter;
     38         dynticks_progress_counter = tmp + 1;
     39       :: else -> skip;
     40       fi;
     41     :: else -> skip;
     42     fi;
     43     atomic {
     44       i++;
     45       in_dyntick_nmi = 0;
     46     }
     47   od;
     48   dyntick_nmi_done = 1;
     49 }
```

Of course, the fact that we have NMIs requires adjustments in the other components. For example, the `EXECUTE_MAINLINE()` macro now needs to pay attention to the NMI handler (`in_dyntick_nmi`) as well as the interrupt handler (`in_dyntick_irq`) by checking the `dyntick_nmi_done` variable as follows: 

```
1 #define EXECUTE_MAINLINE(label, stmt) \
      2 label: skip; \
      3     atomic { \
      4       if \
      5       :: in_dyntick_irq || in_dyntick_nmi -> goto label; \
      6       :: else -> stmt; \
      7       fi; \
      8     } \
```

We will also need to introduce an `EXECUTE_IRQ()` macro that checks `in_dyntick_nmi` in order to allow `dyntick_irq()` to exclude `dyntick_nmi()`: 

```
1 #define EXECUTE_IRQ(label, stmt) \
      2 label: skip; \
      3     atomic { \
      4       if \
      5       :: in_dyntick_nmi -> goto label; \
      6       :: else -> stmt; \
      7       fi; \
      8     } \
```

It is further necessary to convert `dyntick_irq()` to `EXECUTE_IRQ()` as follows: 

```
1 proctype dyntick_irq()
      2 {
      3   byte tmp;
      4   byte i = 0;
      5   byte j = 0;
      6   bit old_gp_idle;
      7   bit outermost;
      8 
      9   do
     10   :: i >= MAX_DYNTICK_LOOP_IRQ && j >= MAX_DYNTICK_LOOP_IRQ -> break;
     11   :: i < MAX_DYNTICK_LOOP_IRQ ->
     12     atomic {
     13       outermost = (in_dyntick_irq == 0);
     14       in_dyntick_irq = 1;
     15     }
     16 stmt1: skip;
     17     atomic {
     18       if
     19       :: in_dyntick_nmi -> goto stmt1;
     20       :: !in_dyntick_nmi && rcu_update_flag ->
     21         goto stmt1_then;
     22       :: else -> goto stmt1_else;
     23       fi;
     24     }
     25 stmt1_then: skip;
     26     EXECUTE_IRQ(stmt1_1, tmp = rcu_update_flag)
     27     EXECUTE_IRQ(stmt1_2, rcu_update_flag = tmp + 1)
     28 stmt1_else: skip;
     29 stmt2: skip;  atomic {
     30       if
     31       :: in_dyntick_nmi -> goto stmt2;
     32       :: !in_dyntick_nmi &&
     33          !in_interrupt &&
     34          (dynticks_progress_counter & 1) == 0 ->
     35            goto stmt2_then;
     36       :: else -> goto stmt2_else;
     37       fi;
     38     }
     39 stmt2_then: skip;
     40     EXECUTE_IRQ(stmt2_1, tmp = dynticks_progress_counter)
     41     EXECUTE_IRQ(stmt2_2, dynticks_progress_counter = tmp + 1)
     42     EXECUTE_IRQ(stmt2_3, tmp = rcu_update_flag)
     43     EXECUTE_IRQ(stmt2_4, rcu_update_flag = tmp + 1)
     44 stmt2_else: skip;
     45     EXECUTE_IRQ(stmt3, tmp = in_interrupt)
     46     EXECUTE_IRQ(stmt4, in_interrupt = tmp + 1)
     47 stmt5: skip;
     48     atomic {
     49       if
     50       :: in_dyntick_nmi -> goto stmt4;
     51       :: !in_dyntick_nmi && outermost ->
     52         old_gp_idle = (grace_period_state == GP_IDLE);
     53       :: else -> skip;
     54       fi;
     55     }
     56     i++;
     57   :: j < i ->
     58 stmt6: skip;
     59     atomic {
     60       if
     61       :: in_dyntick_nmi -> goto stmt6;
     62       :: !in_dyntick_nmi && j + 1 == i ->
     63         assert(!old_gp_idle || grace_period_state != GP_DONE);
     64       :: else -> skip;
     65       fi;
     66     }
     67     EXECUTE_IRQ(stmt7, tmp = in_interrupt);
     68     EXECUTE_IRQ(stmt8, in_interrupt = tmp - 1);
     69 
     70 stmt9: skip;
     71     atomic {
     72       if
     73       :: in_dyntick_nmi -> goto stmt9;
     74       :: !in_dyntick_nmi && rcu_update_flag != 0 ->
     75         goto stmt9_then;
     76       :: else -> goto stmt9_else;
     77       fi;
     78     }
     79 stmt9_then: skip;
     80     EXECUTE_IRQ(stmt9_1, tmp = rcu_update_flag)
     81     EXECUTE_IRQ(stmt9_2, rcu_update_flag = tmp - 1)
     82 stmt9_3: skip;
     83     atomic {
     84       if
     85       :: in_dyntick_nmi -> goto stmt9_3;
     86       :: !in_dyntick_nmi && rcu_update_flag == 0 ->
     87         goto stmt9_3_then;
     88       :: else -> goto stmt9_3_else;
     89       fi;
     90     }
     91 stmt9_3_then: skip;
     92     EXECUTE_IRQ(stmt9_3_1, tmp = dynticks_progress_counter)
     93     EXECUTE_IRQ(stmt9_3_2, dynticks_progress_counter = tmp + 1)
     94 stmt9_3_else:
     95 stmt9_else: skip;
     96     atomic {
     97       j++;
     98       in_dyntick_irq = (i != j);
     99     }
    100   od;
    101   dyntick_irq_done = 1;
    102 }
```

Note that we have open-coded the "if" statements (for example, lines 16-28). In addition, statements that process strictly local state (such as line 56) need not exclude `dyntick_nmi()`. 

Finally, `grace_period()` requires only a few changes: 

```
1 proctype grace_period()
      2 {
      3   byte curr;
      4   byte snap;
      5   bit shouldexit;
      6 
      7   grace_period_state = GP_IDLE;
      8   atomic {
      9     printf("MAX_DYNTICK_LOOP_NOHZ = %d\n", MAX_DYNTICK_LOOP_NOHZ);
     10     printf("MAX_DYNTICK_LOOP_IRQ = %d\n", MAX_DYNTICK_LOOP_IRQ);
     11     printf("MAX_DYNTICK_LOOP_NMI = %d\n", MAX_DYNTICK_LOOP_NMI);
     12     shouldexit = 0;
     13     snap = dynticks_progress_counter;
     14     grace_period_state = GP_WAITING;
     15   }
     16   do
     17   :: 1 ->
     18     atomic {
     19       assert(!shouldexit);
     20       shouldexit = dyntick_nohz_done &&
     21              dyntick_irq_done &&
     22              dyntick_nmi_done;
     23       curr = dynticks_progress_counter;
     24       if
     25       :: (curr == snap) && ((curr & 1) == 0) ->
     26         break;
     27       :: (curr - snap) > 2 || (curr & 1) == 0 ->
     28         break;
     29       :: else -> skip;
     30       fi;
     31     }
     32   od;
     33   grace_period_state = GP_DONE;
     34   grace_period_state = GP_IDLE;
     35   atomic {
     36     shouldexit = 0;
     37     snap = dynticks_progress_counter;
     38     grace_period_state = GP_WAITING;
     39   }
     40   do
     41   :: 1 ->
     42     atomic {
     43       assert(!shouldexit);
     44       shouldexit = dyntick_nohz_done &&
     45              dyntick_irq_done &&
     46              dyntick_nmi_done;
     47       curr = dynticks_progress_counter;
     48       if
     49       :: (curr == snap) && ((curr & 1) == 0) ->
     50         break;
     51       :: (curr != snap) ->
     52         break;
     53       :: else -> skip;
     54       fi;
     55     }
     56   od;
     57   grace_period_state = GP_DONE;
     58 }
```

We have added the `printf()` for the new `MAX_DYNTICK_LOOP_NMI` parameter on line 11 and added `dyntick_nmi_done` to the `shouldexit` assignments on lines 22 and 46. 

**Quick Quiz 13** : Do you always write your code in this painfully incremental manner??? 

The [model](<https://lwn.net/Articles/279105/>) results in a correct validation with several hundred million states, passing without errors. 

### Conclusions

This effort provided some lessons (re)learned: 

**Promela and spin can validate interrupt/NMI-handler interactions**. 

**Documenting code can help locate bugs**. In this case, the documentation effort located [this bug](<http://lkml.org/lkml/2008/3/16/150>). 

**Validate your code early, often, and up to the point of destruction.** This effort located one subtle [bug](<http://lkml.org/lkml/2008/3/21/347>) that might have been quite difficult to test or debug. 

**Always validate your validation code.** The usual way to do this is to insert a deliberate bug and verify that the validation code catches it. Of course, if the validation code fails to catch this bug, you may also need to verify the bug itself, and so on, recursing infinitely. However, if you find yourself in this position, getting a good night's sleep can be an extremely effective debugging technique. 

Finally, if `cmpxchg` instructions ever become inexpensive enough to tolerate them in the interrupt fastpath, their use could greatly simplify this code. The Promela model for an atomic-instruction-based implementation of this code has more than an order of magnitude fewer states, and the C code is much easier to understand. On the other hand, one must take care when using `cmpxchg` instructions, as some code sequences, if highly contended, can result in starvation. This situation is particularly likely to occur when part of the algorithm uses `cmpxchg`, other parts of the algorithm use atomic instructions that cannot fail (e.g., atomic increment), contention for the variable in question is high, and the code is running on a NUMA or a shared-cache machine. Sadly, almost all multi-socket systems with either multi-core or multi-threaded CPUs fit this description. 

### Acknowledgments

We are indebted to Andrew Theurer, who maintains the large-memory machine that ran the full test. We all owe a debt of gratitude to Vara Prasad for his help in rendering this article human-readable.
