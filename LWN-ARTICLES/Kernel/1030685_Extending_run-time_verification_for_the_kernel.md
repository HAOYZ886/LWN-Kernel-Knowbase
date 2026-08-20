---
title: "Extending run-time verification for the kernel"
url: https://lwn.net/Articles/1030685/
date: "July 30, 2025"
category: Development tools
author: "By Daroc Alden July 30, 2025"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Daroc Alden**  
July 30, 2025

There are a lot of things people expect the Linux kernel to do correctly. Some of these are checked by testing or static analysis; a few are ensured by run-time verification: checking a live property of a running Linux system. For example, the scheduler has a handful of different correctness properties that can be checked in this way. Nam Cao posted a [ patch series](<https://lwn.net/ml/all/cover.1751634289.git.namcao@linutronix.de/>) that aims to extend the kinds of properties that the kernel's [run-time verification system](<https://docs.kernel.org/trace/rv/runtime-verification.html>) can check, by adding support for [ linear temporal logic](<https://en.wikipedia.org/wiki/Linear_temporal_logic>) (LTL). The patch set has seen eleven revisions since the [ first version](<https://lwn.net/ml/all/cover.1741708239.git.namcao@linutronix.de/>) in March 2025, and recently made it into the linux-next tree, from where it seems likely to reach the mainline kernel soon. 

Run-time analysis is present everywhere in the kernel; [ lockdep](<https://www.kernel.org/doc/html/latest/locking/lockdep-design.html>), for example, is a kind of run-time verification. But instrumenting the whole kernel for each kind of verification that people may want to perform is infeasible. The run-time verification subsystem allows for tracking more complex properties by hooking into the kernel's existing tracing infrastructure. For example, run-time verification can be used to ensure that a system schedules tasks correctly; there are options to ensure that task switches only occur during a call to `__schedule()`, that the scheduler is called in a context where it is safe to do so, and various other properties of the scheduler interface that depend on the global state of the system. Each property that is checked in this way is represented by a per-CPU or per-task state machine called a monitor. Tracing events drive the transitions in these machines. If they ever reach an error state, the kernel can be configured to log an error message or panic. 

The use of state machines has the nice property of keeping the actual overhead of the monitors as low as possible. A 2019 [paper](<https://bristot.me/wp-content/uploads/2019/09/paper.pdf>) by Daniel Bristot de Oliveira, Tommaso Cucinotta, and Rômulo Silva de Oliveira showed that the overhead of updating a state machine was actually lower than the overhead of just recording tracing events to a file for later analysis. Because state machines, ironically, do not track much state, the per-task memory usage of the system is quite small as well. 

Writing state machines by hand is a tedious process, though, so the kernel includes an `rvgen` tool that can convert a state machine described in [ Graphviz's](<https://gitlab.com/graphviz/graphviz#graphviz---graph-visualization-tools>) [ DOT](<https://en.wikipedia.org/wiki/DOT_\(graph_description_language\)>) format into appropriate C code. There is a bit of manual work to do in order to connect the generated state machine to the correct tracing events, but `rvgen` also generates appropriate kernel configuration and header files, and provides a checklist of what the programmer will need to implement themselves. 

The problem Cao ran into was that simple deterministic state machines are too inflexible to easily represent some desirable properties. For example, it would be nice to have a monitor that can detect priority inversion in realtime tasks, but representing this property as a state machine is complex and error prone. Cao's solution is to add another specification language to `rvgen` that can handle more complicated statements. The resulting code is still compiled to a state machine — specifically, a [ non-deterministic Büchi automaton](<https://en.wikipedia.org/wiki/B%C3%BCchi_automaton>) — but it can express properties about the future execution of a task more easily. 

The new specification language has a custom syntax, but the underlying semantics are taken from linear temporal logic (LTL), which is a kind of modal logic. LTL extends classical Boolean logic with a notion of time. Unlike some more complicated modeling systems, LTL only deals with a single, discrete, non-branching timeline — hence the "linear" part of the name. In addition to the fundamental operations on Booleans (such as "or" and "not"), LTL has two new operators "next" and "until". In LTL, "next A" means that some proposition A must be true on the next time-step. Similarly, "A until B" means that A must be true at all subsequent points in time until (and possibly after) B is true. 

Just as classical logic has derived operators such as "implies", these two temporal operators can be combined to produce more helpful operators like "eventually" and "always". This makes it possible to express constraints such as "a task that acquires a lock must release the lock before exiting" as something like "it is always the case that a task that acquires a lock does not exit until it releases the lock". In Cao's proposed syntax, that would look like this: 

```
RULE = always (ACQUIRE imply ((not EXIT) until RELEASE))
```

Upper-case words correspond to events or rules; the first rule of the file is used to generate the state machine. Lower-case words are operators. Cao's simple code does not implement operator precedence, so parentheses are mandatory on pain of surprising behavior. 

The code generator is currently fairly basic. The above rule compiles to a five-state non-deterministic state machine, but many sets of states are unreachable. To illustrate the kind of state machine produced for a simple property like the above, I took the generated Büchi machine and flattened it into a deterministic state machine shown in the diagram below. Red edges represent acquire events, blue edges represent exit events, and green edges represent release events. After pruning unreachable states, the machine looks like this: 

[ ![\[A complicated state machine diagram\]](https://static.lwn.net/images/2025/machine-illustration.png) ](<https://lwn.net/Articles/1030831>)

State `s0` is the rejecting state, which indicates that there was a problem. Much of the complexity in this example comes from correctly tracing situations where it is not certain whether a lock has been acquired or not. In any case, this kind of automaton would be painful to write by hand; [ the generated code](<https://lwn.net/Articles/1030831>) is much easier to deal with. In order to use it, the programmer must fill in the implementation of the `ltl_atoms_init()` function, which sets the initial state of the monitor, and then arrange for the `ltl_atom_update()` function to be called from appropriate tracepoints. The rest of the integration with the run-time verification subsystem is handled by the generated code. The actual state machine itself is generated and placed in a separate header file. 

The patch set includes [ two](<https://lwn.net/ml/all/c9d5d176c87c6f5df9c1132015828a3c366bf75b.1751634289.git.namcao@linutronix.de/>) [ example](<https://lwn.net/ml/all/06f4152fdc07f7dd0f28939f77a07ce03be8849f.1751634289.git.namcao@linutronix.de/>) definitions for run-time monitors using the new syntax. Both have to do with ensuring that realtime tasks do not sleep incorrectly, and are simple enough that they probably could have been written by hand. But the hope is that having a generator available will enable other kernel developers to write more complicated run-time checks in their areas of expertise.
