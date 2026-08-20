---
title: Using Promela and Spin to verify parallel algorithms
url: https://lwn.net/Articles/243851/
date: "August 1, 2007"
category: Development tools
author: "August 1, 2007 This article was contributed by Paul McKenney"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

August 1, 2007

This article was contributed by Paul McKenney

### Validating Parallel Algorithms

Parallel algorithms can be hard to write, and even harder to debug. Testing, though essential, is insufficient, as fatal race conditions can have extremely low probabilities of occurrence. Proofs of correctness can be valuable, but in the end are just as prone to human error as is the original algorithm. 

It would be very helpful to have a tool that could somehow locate all race conditions. Such a tool in fact exists in the form of the language Promela and its compiler Spin. 

### What are Promela and Spin?

Promela is a language designed to help verify protocols, but which can also be used to verify small parallel algorithms. You recode your algorithm and correctness constraints in the C-like language Promela, and then use Spin to translate it into a C program that you can compile and run. The resulting program conducts a full state-space search of your algorithm, either verifying or finding counter-examples for assertions that you can include in your Promela program. 

This full-state search can be extremely powerful, but can also be a two-edged sword. If your algorithm is too complex or your Promela implementation is careless, there might be more states than fit in memory. Furthermore, even given sufficient memory, the state-space search might well run for longer than the expected lifetime of the universe. Therefore, use this tool for compact but complex parallel algorithms. Attempts to naively apply it to even moderate-scale algorithms (let alone the full Linux kernel) will end badly. 

Promela and Spin may be downloaded from [here](<http://spinroot.com/spin/whatispin.html>). 

The above site also gives links to Gerard Holzmann's excellent book on Promela and Spin, as well as online references starting [here](<http://www.spinroot.com/spin/Man/index.html>). 

The remainder of this article describes how to use Promela to debug parallel algorithms, starting with simple examples and progressing to more complex uses. 

### Promela Example: Non-Atomic Increment

The following example demonstrates the textbook race condition resulting from non-atomic increment: 

```
1 #define NUMPROCS 2
      2 
      3 byte counter = 0;
      4 byte progress[NUMPROCS];
      5 
      6 proctype incrementer(byte me)
      7 {
      8   int temp;
      9 
     10   temp = counter;
     11   counter = temp + 1;
     12   progress[me] = 1;
     13 }
     14 
     15 init {
     16   int i = 0;
     17   int sum = 0;
     18 
     19   atomic {
     20     i = 0;
     21     do
     22     :: i < NUMPROCS ->
     23       progress[i] = 0;
     24       run incrementer(i);
     25       i++
     26     :: i >= NUMPROCS -> break
     27     od;
     28   }
     29   atomic {
     30     i = 0;
     31     sum = 0;
     32     do
     33     :: i < NUMPROCS ->
     34       sum = sum + progress[i];
     35       i++
     36     :: i >= NUMPROCS -> break
     37     od;
     38     assert(sum < NUMPROCS || counter == NUMPROCS)
     39   }
     40 }
```

Line 1 defines the number of processes to run (we will vary this to see the effect on state space), line 3 defines the counter, and line 4 is used to implement the assertion that appears on lines 29-39. 

Lines 6-13 define a process that increments the counter non-atomically. The argument "me" is the process number, set by the initialization block later in the code. Because simple Promela statements are each assumed atomic, we must break the increment into the two statements on lines 10-11. The assignment on line 12 marks the process's completion. Because the Spin system will fully search the state space, including all possible sequences of states, there is no need for the loop that would be used for conventional testing. 

Lines 15-40 are the initialization block, which is executed first. Lines 19-28 actually do the initialization, while lines 29-39 perform the assertion. Both are atomic blocks in order to avoid unnecessarily increasing the state space: because they are not part of the algorithm proper, we loose no validation coverage by making them atomic. 

The do-od construct on lines 21-27 implements a Promela loop, which can be thought of as a C "`for (;;)`" loop containing a "switch" statement that allows expressions in case labels. The condition blocks (prefixed by "`::`") are scanned non-deterministically, though in this case only one of the conditions can possibly hold at a given time. The first block of the do-od from lines 22-25 initializes the i-th incrementer's progress cell, runs the i-th incrementer's process, and then increments the variable "i". The second block of the do-od on line 26 exits the loop once these processes have been started. 

The atomic block on lines 29-39 also contains a similar do-od loop that sums up the progress counters. The `assert()` statement on line 38 verifies that if all processes have been completed, then all counts have been correctly recorded. 

You can build and run this program as follows: 

```
spin -a increment.spin          # Translate the model to C
            cc -DSAFETY -o pan pan.c        # Compile the model
            ./pan                           # Run the model
```

This will produce output as follows: 

```
pan: assertion violated ((sum<2)||(counter==2)) (at depth 20)
            pan: wrote increment.spin.trail
            (Spin Version 4.2.5 -- 2 April 2005)
            Warning: Search not completed
                    + Partial Order Reduction
    
            Full statespace search for:
                    never claim             - (none specified)
                    assertion violations    +
                    cycle checks            - (disabled by -DSAFETY)
                    invalid end states      +
    
            State-vector 40 byte, depth reached 22, errors: 1
                  45 states, stored
                  13 states, matched
                  58 transitions (= stored+matched)
                  51 atomic steps
            hash conflicts: 0 (resolved)
    
            2.622   memory usage (Mbyte)
```

The first line tells us that our assertion was violated (as expected given the non-atomic increment!). The second line that a "trail" file was written describing how the assertion was violated. The "Warning" line reiterates that all was not well with our model. The second paragraph describes the type of state-search being carried out, in this case for assertion violations and invalid end states. The third paragraph gives state-size statistics: this small model had only 45 states. The final line shows memory usage. 

The "trail" file may be rendered human-readable as follows: 

```
spin -t -p increment.spin
```

This gives the following output: 

```
Starting :init: with pid 0
      1:    proc  0 (:init:) line  20 "increment.spin" (state 1)    [i = 0]
      2:    proc  0 (:init:) line  22 "increment.spin" (state 2)    [((i<2))]
      2:    proc  0 (:init:) line  23 "increment.spin" (state 3)    [progress[i] = 0]
    Starting incrementer with pid 1
      3:    proc  0 (:init:) line  24 "increment.spin" (state 4)    [(run incrementer(i))]
      3:    proc  0 (:init:) line  25 "increment.spin" (state 5)    [i = (i+1)]
      4:    proc  0 (:init:) line  22 "increment.spin" (state 2)    [((i<2))]
      4:    proc  0 (:init:) line  23 "increment.spin" (state 3)    [progress[i] = 0]
    Starting incrementer with pid 2
      5:    proc  0 (:init:) line  24 "increment.spin" (state 4)    [(run incrementer(i))]
      5:    proc  0 (:init:) line  25 "increment.spin" (state 5)    [i = (i+1)]
      6:    proc  0 (:init:) line  26 "increment.spin" (state 6)    [((i>=2))]
      7:    proc  0 (:init:) line  21 "increment.spin" (state 10)   [break]
      8:    proc  2 (incrementer) line  10 "increment.spin" (state 1)       [temp = counter]
      9:    proc  1 (incrementer) line  10 "increment.spin" (state 1)       [temp = counter]
     10:    proc  2 (incrementer) line  11 "increment.spin" (state 2)       [counter = (temp+1)]
     11:    proc  2 (incrementer) line  12 "increment.spin" (state 3)       [progress[me] = 1]
     12: proc 2 terminates
     13:    proc  1 (incrementer) line  11 "increment.spin" (state 2)       [counter = (temp+1)]
     14:    proc  1 (incrementer) line  12 "increment.spin" (state 3)       [progress[me] = 1]
     15: proc 1 terminates
     16:    proc  0 (:init:) line  30 "increment.spin" (state 12)   [i = 0]
     16:    proc  0 (:init:) line  31 "increment.spin" (state 13)   [sum = 0]
     17:    proc  0 (:init:) line  33 "increment.spin" (state 14)   [((i<2))]
     17:    proc  0 (:init:) line  34 "increment.spin" (state 15)   [sum = (sum+progress[i])]
     17:    proc  0 (:init:) line  35 "increment.spin" (state 16)   [i = (i+1)]
     18:    proc  0 (:init:) line  33 "increment.spin" (state 14)   [((i<2))]
     18:    proc  0 (:init:) line  34 "increment.spin" (state 15)   [sum = (sum+progress[i])]
     18:    proc  0 (:init:) line  35 "increment.spin" (state 16)   [i = (i+1)]
     19:    proc  0 (:init:) line  36 "increment.spin" (state 17)   [((i>=2))]
     20:    proc  0 (:init:) line  32 "increment.spin" (state 21)   [break]
    spin: line  38 "increment.spin", Error: assertion violated
    spin: text of failed assertion: assert(((sum<2)||(counter==2)))
     21:    proc  0 (:init:) line  38 "increment.spin" (state 22)   [assert(((sum<2)||(counter==2)))]
    spin: trail ends after 21 steps
    #processes: 1
                    counter = 1
                    progress[0] = 1
                    progress[1] = 1
     21:    proc  0 (:init:) line  40 "increment.spin" (state 24) <valid end state>
    3 processes created
```

As can be seen, the first portion of the init block created both incrementer processes, both of which first fetched the counter, then both incremented and stored it, losing a count. The assertion then triggered, after which the global state is displayed. 

### Promela Example: Atomic Increment

It is easy to fix this example by placing the body of the incrementer processes in an atomic blocks as follows: 

```
1 proctype incrementer(byte me)
      2 {
      3   int temp;
      4 
      5   atomic {
      6     temp = counter;
      7     counter = temp + 1;
      8   }
      9   progress[me] = 1;
     10 }
```

One could also have simply replaced the pair of statements with `counter = counter + 1`, because Promela statements are atomic. Either way, running this modified model gives us an error-free traversal of the state space: 

```
(Spin Version 4.2.5 -- 2 April 2005)
            + Partial Order Reduction
    
    Full statespace search for:
            never claim             - (none specified)
            assertion violations    +
            cycle checks            - (disabled by -DSAFETY)
            invalid end states      +
    
    State-vector 40 byte, depth reached 20, errors: 0
          52 states, stored
          21 states, matched
          73 transitions (= stored+matched)
          66 atomic steps
    hash conflicts: 0 (resolved)
    
    2.622   memory usage (Mbyte)
    
    unreached in proctype incrementer
            (0 of 5 states)
    unreached in proctype :init:
            (0 of 24 states)
```

### Combinatorial Explosion

The following table shows the number of states and memory consumed as a function of number of incrementers modeled (by redefining `NUMPROCS`): 

> # incrementers|  # states| megabytes  
> ---|---|---  
> 1 |  11|  2.6  
> 2 |  52|  2.6  
> 3 |  372|  2.6  
> 4 |  3,496|  2.7  
> 5 |  40,221|  5.0  
> 6 |  545,720|  40.5  
> 7 |  8,521,450|  652.7  
  
Running unnecessarily large models is thus subtly discouraged, although 652MB is well within the limits of some of the more recent desktop and laptop machines. With this example under our belt, let's take a closer look at the commands used to analyze Promela models and then look at more elaborate examples. 

### How to Use Promela

Given a source file "qrcu.spin", one can use the following commands: 

```
spin -a qrcu.spin
```

> Create a file pan.c that fully searches the state machine. 

```
cc -DSAFETY -o pan pan.c
```

> Compile the generated state-machine search. The "-DSAFETY" generates optimizations that are appropriate if you have only assertions (and perhaps "never" statements). If you have liveness, fairness, or forward-progress checks, you may need to compile without "-DSAFETY". If you leave off "-DSAFETY" when you could have used it, the program will let you know. 
> 
> The optimizations produced by "-DSAFETY" greatly speed things up, so you should use it when you can. 

```
./pan
```

> This actually searches the state space. The number of states can reach into the tens of millions with very small state machines, so you will need a machine with large memory. For example, qrcu.spin with 3 readers and 2 updaters required 2.7GB of memory. 
> 
> If you aren't sure whether your machine has enough memory, run "top" in one window and "./pan" in another. Keep the focus on the "./pan" window so that you can quickly kill execution if need be. As soon as CPU time drops much below 100%, kill "./pan". If you have removed focus from the window running "./pan", you may wait a long time for the windowing system to grab enough memory to do anything for you. 
> 
> Don't forget to capture the output, especially if you are working on a remote machine, 

```
spin -t -p qrcu.spin
```

> Given "trail" file output by a run that encountered an error, output the sequence of steps leading to that error. 

### Promela Peculiarities

Although all computer languages have underlying similarities, Promela will provide some surprises to people used to coding in C. 

  * In C, ";" terminates statements. In Promela it separates them. Fortunately, more recent versions of Spin have become much more forgiving of "extra" semicolons. 

  * Promela's looping construct, the "do" statement, takes conditions. This "do" statement closely resembles a looping if-then-else statement. 

  * In C's "switch" statement, if there is no matching case, the whole statement is skipped. In Promela's equivalent, confusingly called "if", if there is no matching guard expression, you get an error without a recognizable corresponding error message. So, if the error output indicates an innocent line of code, check to see if you left out a condition from an "if" or "do" statement. 

  * When creating stress tests in C, one usually races suspect operations against each other repeatedly. In Promela, one instead sets up a single race, because Promela will search out all the possible outcomes from that single race. Sometimes you do need to loop in Promela, for example, if multiple operations overlap, but doing so greatly increases the size of your state space. 

  * In C, the easiest thing to do is to maintain a loop counter to track progress and terminate the loop. In Promela, loop counters must be avoided like the plague because they cause the state space to explode. On the other hand, there is no penalty for infinite loops in Promela as long as the none of the variables monotonically increase or decrease -- Promela will figure out how many passes through the loop really matter, and automatically prune execution beyond that point. 

  * In C torture-test code, it is often wise to keep per-task control variables. They are cheap to read, and greatly aid in debugging the test code. In Promela, per-task control variables should be used only when there is no other alternative. To see this, consider a 5-task validation with one bit each to indicate completion. This gives 32 states. In contrast, a simple counter would have only six states, more than a five-fold reduction. That factor of five might not seem like a problem, at least not until you are struggling with a validation program possessing more than 150 million states consuming more than 10GB of memory! 

  * One of the most challenging things both in C torture-test code and in Promela is formulating good assertions. Promela also allows "never" claims that act sort of like an assertion replicated between every line of code. 

  * Dividing and conquering is extremely helpful in Promela in keeping the state space under control. Splitting a large model into two roughly equal halves will result in the state space of each half being roughly the square root of the whole. For example, a million-state combined model might reduce to a pair of thousand-state models. Not only will Promela handle the two smaller models much more quickly with much less memory, but the two smaller algorithms are easier for people to understand. 

### Promela Coding Tricks

Promela was designed to analyze protocols, so using it on parallel programs is a bit abusive. The following tricks can help you to abuse Promela safely: 

  * Memory reordering. Suppose you have a pair of statements copying globals x and y to locals r1 and r2, where ordering matters (e.g., unprotected by locks), but where you have no memory barriers. This can be modeled in Promela as follows: 

```
if
                :: 1 -> r1 = x;
                        r2 = y
                :: 1 -> r2 = y;
                        r1 = x
                fi
```

The two branches of the "if" statement will be selected nondeterministically, since they both are available. Because the full state space is searched, _both_ choices will eventually be made in all cases. 

Of course, this trick will cause your state space to explode if used too heavily. In addition, it requires you to anticipate possible reorderings. 
  * State reduction. If you have complex assertions, evaluate them under "atomic". After all, they are not part of the algorithm. One example of a complex assertion (to be discussed in more detail later) is as follows: 

```
i = 0;
                sum = 0;
                do
                :: i < N_QRCU_READERS ->
                        sum = sum + (readerstart[i] == 1 &&
                                     readerprogress[i] == 1);
                        i++
                :: i >= N_QRCU_READERS ->
                        assert(sum == 0);
                        break
                od
```

There is no reason to evaluate this assertion non-atomically, since it is not actually part of the algorithm. Because each statement contributes to state, we can reduce the number of useless states by enclosing it in an "atomic" block as follows: 

```
atomic {
                        i = 0;
                        sum = 0;
                        do
                        :: i < N_QRCU_READERS ->
                                sum = sum + (readerstart[i] == 1 &&
                                             readerprogress[i] == 1);
                                i++
                        :: i >= N_QRCU_READERS ->
                                assert(sum == 0);
                                break
                        od
                }
```

  * Promela does not provide functions. You must instead use C preprocessor macros. However, you must use them carefully in order to avoid combinatorial explosion. 
Now we are ready for more complex examples. 

### Promela Example: Locking

Since locks are generally useful, `spin_lock()` and `spin_unlock()` macros are provided in `lock.h`, which may be included from multiple Promela models: 

```
1 #define spin_lock(mutex) \
      2   do \
      3   :: 1 -> atomic { \
      4       if \
      5       :: mutex == 0 -> \
      6         mutex = 1; \
      7         break \
      8       :: else -> skip \
      9       fi \
     10     } \
     11   od
     12 
     13 #define spin_unlock(mutex) \
     14   mutex = 0
```

The `spin_lock()` macro contains an infinite do-od loop spanning lines 2-11, courtesy of the single guard expression of "1" on line 3. The body of this loop is a single atomic block that contains an if-fi statement. The if-fi construct is similar to the do-od construct, except that it takes a single pass rather than looping. If the lock is not held on line 5, then line 6 acquires it and line 7 breaks out of the enclosing do-od loop (and also exits the atomic block). On the other hand, if the lock is already held on line 8, we do nothing ("skip"), and fall out of the if-fi and the atomic block so as to take another pass through the outer loop, repeating until the lock is available. 

The `spin_unlock()` macro simply marks the lock as no longer held. 

Note that memory barriers are not needed because Promela assumes full ordering. In any given Promela state, all processes agree on both the current state and the order of state changes that caused us to arrive at the current state. This is analogous to the "sequentially consistent" memory model used by a few computer systems (such as MIPS and PA-RISC). As noted earlier, and as will be seen in a later example, weak memory ordering must be explicitly coded. 

These macros are tested by the following Promela code: 

```
1 #include "lock.h"
      2 
      3 #define N_LOCKERS 3
      4 
      5 bit mutex = 0;
      6 bit havelock[N_LOCKERS];
      7 int sum;
      8 
      9 proctype locker(byte me)
     10 {
     11   do
     12   :: 1 ->
     13     spin_lock(mutex);
     14     havelock[me] = 1;
     15     havelock[me] = 0;
     16     spin_unlock(mutex)
     17   od
     18 }
     19 
     20 init {
     21   int i = 0;
     22   int j;
     23 
     24 end:  do 
     25   :: i < N_LOCKERS ->
     26     havelock[i] = 0;
     27     run locker(i);
     28     i++
     29   :: i >= N_LOCKERS ->
     30     sum = 0;
     31     j = 0;
     32     atomic {
     33       do
     34       :: j < N_LOCKERS ->
     35         sum = sum + havelock[j];
     36         j = j + 1
     37       :: j >= N_LOCKERS ->
     38         break
     39       od
     40     }
     41     assert(sum <= 1);
     42     break
     43   od
     44 }
```

This code is similar to that used to test the increments, with the number of locking processes defined by the `N_LOCKERS` macro definition on line 3. The mutex itself is defined on line 5, an array to track the lock owner on line 6, and line 7 is used by assertion code to verify that only one process holds the lock. 

The locker process is on lines 9-18, and simply loops forever acquiring the lock on line 13, claiming it on line 14, unclaiming it on line 15, and releasing it on line 16. 

The init block on lines 20-44 initializes the current locker's havelock array entry on line 26, starts the current locker on line 27, and advances to the next locker on line 28. Once all locker processes are spawned, the do-od loop moves to line 29, which checks the assertion. Lines 30 and 31 initialize the control variables, lines 32-40 atomically sum the havelock array entries, line 41 is the assertion, and line 42 exits the loop. 

We can run this model by placing the above two code fragments into files named "lock.h" and "lock.spin", respectively, and then running the following commands: 

```
spin -a lock.spin
            cc -DSAFETY -o pan pan.c
            ./pan
```

The output will look something like the following: 

```
(Spin Version 4.2.5 -- 2 April 2005)
                    + Partial Order Reduction
    
            Full statespace search for:
                    never claim             - (none specified)
                    assertion violations    +
                    cycle checks            - (disabled by -DSAFETY)
                    invalid end states      +
    
            State-vector 40 byte, depth reached 357, errors: 0
                 564 states, stored
                 929 states, matched
                1493 transitions (= stored+matched)
                 368 atomic steps
            hash conflicts: 0 (resolved)
    
            2.622   memory usage (Mbyte)
    
            unreached in proctype locker
                    line 18, state 20, "-end-"
                    (1 of 20 states)
            unreached in proctype :init:
                    (0 of 22 states)
```

As expected, this run has no assertion failures ("errors: 0"). 

**Quick Quiz 1:** Why is there an unreached statement in locker? After all, isn't this a _full_ state-space search??? 

**Quick Quiz 2:** What are some Promela code-style issues with this example? In particular, why does this example have more states than the n=3 case for atomic increment (564 vs. 372)? 

### Promela Example: QRCU

This final example demonstrates a real-world use of Promela on Oleg Nesterov's QRCU, but modified to speed up the `synchronize_qrcu()` fastpath. 

But first, what is QRCU? 

QRCU is a variant of [SRCU](<http://lwn.net/Articles/202847/>) (which in turn is a variant of [RCU](<http://lwn.net/Articles/37889/>)) that trades somewhat higher read overhead (atomic increment and decrement on a global variable) for extremely low grace-period latencies. If there are no readers, the grace period will be detected in less than a microsecond, compared to the multi-millisecond grace-period latencies of most other RCU implementations. 

  1. There is a `qrcu_struct` that defines a QRCU domain. Like SRCU (and unlike other variants of RCU) QRCU's action is not global, but instead focused on the specified `qrcu_struct`. 
  2. There are `qrcu_read_lock()` and `qrcu_read_unlock()` primitives that delimit QRCU read-side critical sections. The corresponding `qrcu_struct` must be passed into these primitives, and the return value from `rcu_read_lock()` must be passed to `rcu_read_unlock()`. 

For example: 

```
idx = qrcu_read_lock(&my_qrcu_struct);
                 /* read-side critical section. */
                 qrcu_read_unlock(&my_qrcu_struct, idx);
```

  3. There is a `synchronize_qrcu()` primitive that blocks until all pre-existing QRCU read-side critical sections complete, but, like SRCU's `synchronize_srcu()`, QRCU's `synchronize_qrcu()` need wait only for those read-side critical sections that are using the same `qrcu_struct`. 

For example, `synchronize_qrcu(&your_qrcu_struct)` would _not_ need to wait on the earlier QRCU read-side critical section. In contrast, `synchronize_qrcu(&my_qrcu_struct)` _would_ need to wait, since it shares the same `qrcu_struct`. 
A Linux-kernel patch for QRCU may be found [here](<http://lwn.net/Articles/223752/>). 

Returning to the Promela code for QRCU, the global variables are as follows: 

```
1 #include "lock.h"
      2
      3 #define N_QRCU_READERS 2
      4 #define N_QRCU_UPDATERS 2
      5
      6 bit idx = 0;
      7 byte ctr[2];
      8 byte readerprogress[N_QRCU_READERS];
      9 bit mutex = 0;
```

This example uses locking, hence including "lock.h". Both the number of readers and writers can be varied using the two "#define" statements, giving us not one but two ways to create combinatorial explosion. The "idx" variable controls which of the two elements of the "ctr" array will be used by readers, and the "readerprogress" variable allows the assertion to determine when all the readers are finished (since a QRCU update cannot be permitted to complete until all pre-existing readers have completed their QRCU read-side critical sections). The readerprogress array elements have values as follows, indicating the state of the corresponding reader: 

  * 0: not yet started. 
  * 1: within QRCU read-side critical section. 
  * 2: finished with QRCU read-side critical section. 
Finally, the "mutex" variable is used to serialize updaters' slowpaths. 

QRCU readers are modeled by the `qrcu_reader()` process shown below. 

```
1 proctype qrcu_reader(byte me)
      2 {
      3   int myidx;
      4
      5   do
      6   :: 1 ->
      7     myidx = idx;
      8     atomic {
      9       if
     10       :: ctr[myidx] > 0 ->
     11         ctr[myidx]++;
     12         break
     13       :: else -> skip
     14       fi
     15     }
     16   od;
     17   readerprogress[me] = 1;
     18   readerprogress[me] = 2;
     19   atomic { ctr[myidx]-- }
     20 }
```

A do-od loop spans lines 5-16, with a single guard of "1" on line 6 that makes it an infinite loop. Line 7 captures the current value of the global index, and lines 8-15 atomically increment it (and break from the infinite loop) if its value was non-zero (`atomic_inc_not_zero()`). Line 17 marks entry into the RCU read-side critical section, and line 18 marks exit from this critical section, both lines for the benefit of the `assert()` statement that we shall encounter later. Line 19 atomically decrements the same counter that we incremented, thereby exiting the RCU read-side critical section. 

The following C-preprocessor macro sums the pair of counters forcing weak memory ordering. 

```
1 #define sum_unordered \
      2   atomic { \
      3     do \
      4     :: 1 -> \
      5       sum = ctr[0]; \
      6       i = 1; \
      7       break \
      8     :: 1 -> \
      9       sum = ctr[1]; \
     10       i = 0; \
     11       break \
     12     od; \
     13   } \
     14   sum = sum + ctr[i]
```

Lines 2-13 fetch one of the counters, and line 14 fetches the other of the pair and sums them. The atomic block consists of a single do-od statement. This do-od statement (spanning lines 3-12) is unusual in that it contains two unconditional branches with guards on lines 4 and 8, which causes Promela to non-deterministically choose one of the two (but again, the full state-space search causes Promela to eventually make all possible choices in each applicable situation). The first branch fetches the zero-th counter and sets "i" to 1 (so that line 14 will fetch the first counter), while the second branch does the opposite, fetching the first counter and setting "i" to 0 (so that line 14 will fetch the second counter). 

**Quick Quiz 3:** Is there a more straightforward way to code the do-od statement? 

With the `sum_unordered` macro in place, we can now proceed to the update-side process below. 

```
1 proctype qrcu_updater(byte me)
      2 {
      3   int i;
      4   byte readerstart[N_QRCU_READERS];
      5   int sum;
      6
      7   do
      8   :: 1 ->
      9
     10     /* Snapshot reader state. */
     11
     12     atomic {
     13       i = 0;
     14       do
     15       :: i < N_QRCU_READERS ->
     16         readerstart[i] = readerprogress[i];
     17         i++
     18       :: i >= N_QRCU_READERS ->
     19         break
     20       od
     21     }
     22
     23     sum_unordered;
     24     if
     25     :: sum <= 1 -> sum_unordered
     26     :: else -> skip
     27     fi;
     28     if
     29     :: sum > 1 ->
     30       spin_lock(mutex);
     31       atomic { ctr[!idx]++ }
     32       idx = !idx;
     33       atomic { ctr[!idx]-- }
     34       do
     35       :: ctr[!idx] > 0 -> skip
     36       :: ctr[!idx] == 0 -> break
     37       od;
     38       spin_unlock(mutex);
     39     :: else -> skip
     40     fi;
     41
     42     /* Verify reader progress. */
     43
     44     atomic {
     45       i = 0;
     46       sum = 0;
     47       do
     48       :: i < N_QRCU_READERS ->
     49         sum = sum + (readerstart[i] == 1 &&
     50                readerprogress[i] == 1);
     51         i++
     52       :: i >= N_QRCU_READERS ->
     53         assert(sum == 0);
     54         break
     55       od
     56     }
     57   od
     58 }
```

The update-side process repeats indefinitely, with the corresponding do-od loop ranging over lines 7-57. Each pass through the loop first snapshots the global `readerprogress` array into the local `readerstart` array on lines 12-21. This snapshot will be used for the assertion on line 53. Line 23 invokes `sum_unordered`, and then lines 24-27 re-invoke `sum_unordered` if the fastpath is potentially usable. 

Lines 28-40 execute the slowpath code if need be, with lines 30 and 38 acquiring and releasing the update-side lock, lines 31-33 flipping the index, and lines 34-37 waiting for all pre-existing readers to complete. 

Lines 44-56 then compare the current values in the `readerprogress` array to those collected in the `readerstart` array, forcing an assertion failure should any readers that started before this update still be in progress. 

**Quick Quiz 4:** Why are there atomic blocks at lines 12-21 and lines 44-56, when the operations within those atomic blocks have no atomic implementation on any current production microprocessor? 

**Quick Quiz 5:** Is the re-summing of the counters on lines 24-27 _really_ necessary??? 

All that remains is the initialization block shown below: 

```
1 init {
      2   int i;
      3 
      4   atomic {
      5     ctr[idx] = 1;
      6     ctr[!idx] = 0;
      7     i = 0;
      8     do
      9     :: i < N_QRCU_READERS ->
     10       readerprogress[i] = 0;
     11       run qrcu_reader(i);
     12       i++
     13     :: i >= N_QRCU_READERS -> break
     14     od;
     15     i = 0;
     16     do
     17     :: i < N_QRCU_UPDATERS ->
     18       run qrcu_updater(i);
     19       i++
     20     :: i >= N_QRCU_UPDATERS -> break
     21     od
     22   }
     23 }
```

This block simply initializes the counter pair on lines 5-6, spawns the reader processes on lines 7-14, and spawns the updater processes on lines 15-21. This is all done within an atomic block to reduce state space. 

### Running the QRCU Example

To run the QRCU example, combine the code fragments in the previous section into a single file named "qrcu.spin", and place the definitions for `spin_lock()` and `spin_unlock()` into a file named "lock.h". Then use the following commands to build and run the QRCU model: 

```
spin -a qrcu.spin
            cc -DSAFETY -o pan pan.c
            ./pan
```

The resulting output shows that this model passes all of the cases shown in this table: 

> # updaters | # readers |  # states| megabytes  
> ---|---|---|---  
> 1| 1|  376|  2.6  
> 1| 2|  6,177|  2.9  
> 1| 3|  82,127|  7.5  
> 2| 1|  29,399|  4.5  
> 2| 2|  1,071,180|  75.4  
> 2| 3|  33,866,700|  2,715.2  
> 3| 1|  258,605|  22.3  
> 3| 2| 169,533,000| 14,979.9  
  
Now, it would be nice to run the case with three readers and three updaters, however, simple extrapolation indicates that this will require on the order of a terabyte of memory best case. So, what to do? Here are some possible approaches: 

  1. See whether a smaller number of readers and updaters suffice to prove the general case. 
  2. Manually construct a proof of correctness. 
  3. Use a more capable tool. 
  4. Divide and conquer. 

The following sections discuss each of these approaches. 

### How Many Readers and Updaters Are Really Needed?

One approach is to look carefully at the Promela code for `qrcu_updater()` and notice that the only global state change is happening under the lock. Therefore, only one updater at a time can possibly be modifying state visible to either readers or other updaters. This means that any sequences of state changes can be carried out serially by a single updater due to the fact that Promela does a full state-space search. Therefore, at most two updaters are required: one to change state and a second to become confused. 

The situation with the readers is less clear-cut, as each reader does only a single read-side critical section then terminates. It is possible to argue that the useful number of readers is limited, due to the fact that the fastpath must see at most a zero and a one in the counters. This is a fruitful avenue of investigation, in fact, it leads to the full proof of correctness described in the next section. 

### Proof of Correctness

The [proof](<http://lwn.net/Articles/223752/>) is as follows: 

  1. For `synchronize_qrcu()` to exit too early, then by definition there must have been at least one reader present during `synchronize_qrcu()`'s full execution. 

  2. The counter corresponding to this reader will have been at least 1 during this time interval. 

  3. The `synchronize_qrcu()` code forces at least one of the counters to be at least 1 at all times. 

  4. Therefore, at any given point in time, either one of the counters will be at least 2, or both of the counters will be at least one. 

  5. However, the `synchronize_qrcu()` fastpath code can read only one of the counters at a given time. It is therefore possible for the fastpath code to fetch the first counter while zero, but to race with a counter flip so that the second counter is seen as one. 

  6. There can be at most one reader persisting through such a race condition, as otherwise the sum would be two or greater, which would cause the updater to take the slowpath. 

  7. But if the race occurs on the fastpath's first read of the counters, and then again on its second read, there have to have been two counter flips. 

  8. Because a given updater flips the counter only once, and because the update-side lock prevents a pair of updaters from concurrently flipping the counters, the only way that the fastpath code can race with a flip twice is if the first updater completes. 

  9. But the first updater will not complete until after all pre-existing readers have completed. 

  10. Therefore, if the fastpath races with a counter flip twice in succession, all pre-existing readers must have completed, so that it is safe to take the fastpath. 
Of course, not all parallel algorithms have such simple proofs. In such cases, it may be necessary to enlist more capable tools. 

### More Capable Tools

Although Promela and Spin are quite useful, much more capable tools are available, particularly for validating hardware. This means that if it is possible to translate your algorithm to the hardware-design VHDL language, as it often will be for low-level parallel algorithms, then it is possible to apply these tools to your code (for example, this was done for the first realtime RCU algorithm). However, such tools can be quite expensive. 

Although the advent of commodity multiprocessing might eventually result in powerful free-software model-checkers featuring fancy state-space-reduction capabilities, this does not help much in the here and now. 

As an aside, there are Spin features that support approximate searches that require fixed amounts of memory, however, I have never been able to bring myself to trust approximations when validating parallel algorithms. 

Another approach might be to divide and conquer. 

### Divide and Conquer

It is often possible to break down a larger parallel algorithm into smaller pieces, which can then be proven separately. For example, a 10-billion-state model might be broken into a pair of 100,000-state models. Taking this approach not only makes it easier for tools such as Promela to validate your algorithms, it can also make your algorithms easier to understand. 

### Summary

Promela is a very powerful tool for validating small parallel algorithms. It is a useful tool in the parallel kernel hacker's toolbox, but it should not be the only tool. The QRCU experience is a case in point: given the Promela validation, the proof of correctness, and several [rcutorture](<http://git.kernel.org/?p=linux/kernel/git/torvalds/linux-2.6.git;a=blob;hb=HEAD;f=Documentation/RCU/torture.txt>) runs, I now feel reasonably confident in the QRCU algorithm and its implementation. But I would certainly not feel so confident given only one of the three! 

### Acknowledgements

Many thanks to Suparna Bhattacharya and to Josh Triplett for their careful review of this article. 

The answers to the Quick Quiz questions are available [here](<http://lwn.net/Articles/243966/>).
