---
title: "Smatch: pluggable static analysis for C"
url: https://lwn.net/Articles/691882/
date: "June 22, 2016"
category: "Development tools-Static analysis; Smatch"
author: "June 22, 2016 This article was contributed by Neil Brown"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

June 22, 2016

This article was contributed by Neil Brown

We recently [explored the "sparse" C parsing library](<https://lwn.net/Articles/689907/>) and saw that it made it easy to build tools to inspect and report on the abstract syntax tree (AST) of a C program. Sparse provides functionality to simplify that AST so that particular features of the code can stand out, but keeps the focus fairly local. In particular, it doesn't support any significant data-flow analysis to detect how values change across a sequence of code. [Smatch](<http://repo.or.cz/w/smatch.git>) is a tool that has been built on top of sparse and adds exactly that support, and more. This extra analysis makes it possible to detect such things as conditions that will always (or never) be true, pointers that might be null, and locks that end up in different states depending on which path is taken through the code. This can be very helpful for validating error paths and other rarely tested code. 

The [recent announcement](<https://lwn.net/Articles/690169/>) of the Core Infrastructure Initiative's "Best-Practices Badge," which requires the use of static code analysis where possible, will hopefully help focus people's thoughts on the value of adding such analysis to their workflow; that should lead to a focus on the tools for static analysis. As C is a highly flexible language, it is likely that different projects will use different patterns and styles and so be susceptible to different sorts of bugs. This flexibility suggests that a one-size-fits-all static analyzer may not be the best solution. We already saw with sparse that it has functionality specifically tailored to check patterns that are unique to the Linux kernel; there is no reason to expect other projects won't have their own unique needs. Smatch is already prepared to handle this variability; to that end it implements the concept of a current "project" that can be used to tune the analysis in various ways. It has hard-coded knowledge of two projects: "kernel" and "wine". Extending that list probably only requires interest to be shown.

#### Smatch: the big picture

Smatch is a project that Dan Carpenter has been working on for about 13 years — though, for the first few years, it was based on GCC rather than sparse. It has always had a particular focus on looking for bugs in the Linux kernel, so many of its tests are for patterns that have been seen in the kernel and which often result in buggy or confusing code. In a [brief introduction to the tool](<https://blogs.oracle.com/linuxkernel/entry/smatch_static_analysis_tool_overview>) posted last year he reported that about 3000 kernel bugs have been patched thanks to warnings from Smatch. Many were minor bugs in obscure corner cases, but others had serious, real-world consequences.

Smatch is clearly a work-in-progress. A common problem with static-analysis tools is that of false positives: it is hard to report all problematic code without also reporting some perfectly good and correct code too. To make that distinction, it is sometimes necessary to know what the programmer was thinking and that level of abstraction is still beyond the state of the art. Consequently, Smatch needs a lot of heuristics to try to guess what is intended, and some of those are more fully developed than others. Some of the code in Smatch doesn't seem particularly useful and there is some code that isn't used at all. Nonetheless, there is a great deal of functionality that appears quite solid and useful; that is where we will focus our attention. If you need some project-specific static analysis for your Best-Practices Badge, that would be the code to build upon.

As mentioned, Smatch itself is built upon a foundation of sparse. It doesn't use all of sparse, omitting the "tree simplification" stages completely and bypassing the normal calls to `evaluate_symbol()`, using instead calls to `evaluate_expression()` only when needed. On top of this foundation, it is possible to see three layers in the functionality provided by Smatch. The last layer contains various specific tests for particular sorts of problems, and it is to this layer that any new test would be added. Understanding the layers beneath it will allow a new test to be written with minimal fuss.

#### Smatch: a code checking framework

Where sparse is structured as a library, Smatch is structured as a framework, though it also provides a library of support code. Individual checkers are written as plugins that the Smatch framework will call into as needed. These plugins can be independent or they can cooperate to build up the state necessary to perform some particular test or match some general pattern. For example the `check_user_data.c` module keeps track of when variables refer to user-space data, and exports a function `is_user_data()` that other modules can use when performing specific tests. The plugin infrastructure does not support dynamic loading of plugins, but adding a plugin to the C code can be achieved by adding a single C file and a single line to another file to register an initialization function to be called. Adding support for dynamic loading would only require a small amount of work.

There are two key mechanisms that a checker uses to interact with the rest of Smatch: hooks and states. Hooks are registered functions that are called when particular features, such as pointer dereferences or function calls, are found in the parse. States are a means for those hook functions to communicate from one place in the code to another.

Smatch maintains a set of states as it walks around the AST. Each state belongs to a particular checker and has a key and a value. The key is often the name of a variable or a field within a structure, but it can be anything that has state, such as a particular signal that might have been blocked or a capability in the kernel that might have been tested with `capable()`. The value can be as simple as a label like "non-null" or "unlocked", or can have an arbitrarily complex structure, such as a list of possible values and probable values. This state can be updated whenever a hook is called for an operation that would change the relevant object or for an operation that tests the value of the object. The current state can be tested whenever it might be relevant.

The most interesting handling of state happens with conditional code: when the code flow can branch and then join. Smatch makes it easy to record different states on different paths of an `if` statement (or similar). The `CONDITION_HOOK` hook is called when Smatch finds a condition that affects the flow of code. A function attached to this hook can call `set_true_false_states()`, passing a variable name (or other key) and two states, one that will describe the variable if the condition evaluates true, and one for when it is false. These two states, which might for example be "is-null" and "non-null", will be used in the two different code paths that follow the condition. When an `if` statement has a complex condition, such as "`(!foo && bar > 3)`", Smatch drills down as deeply as it can and just passes the leaf tests to the hook, so, in this example, the `CONDITION_HOOK` will be called twice, once for "`foo`" and once for "`bar > 3`"; Smatch will make sure the right state ends up in the right code path. 

Merging states after a branching statement has completed is much more interesting. Smatch stores three different sorts of merges, each of which could be useful in different contexts. First, for each pair of states in need of merging, Smatch will call the `merge_hook()` function if the checker owning those states provides one. This hook must create a new state that reflects the merge. If no merge hook is available, a pre-defined state called `merged` is used. Second, Smatch keeps a list of possible states for each checker and each key. This list always contains the main state unless that state is `merged`; when two states are merged, the list of possible states is merged as well, discarding duplicates. So if Smatch processed code like:

```
switch(var) {
        	case 1: break;
        	case 2: break;
        	default: return
        }
```

then, after the `switch` statement, that list of possible states for `var` would include the state after it had been compared with "1" and the state after it had been compared with "2". Checkers can then test if a particular state of interest is in the list of possible states for a given key.

Finally Smatch keeps a more detailed history of which states were present at each previous merge: a tree of states with "left" and "right" branches reflecting the two inputs each at merge point. This history is used to effectively look back in time to a previous place in the code when a variable was known to have a particular state. The details of how this works are fairly complex but the result is easy to understand and powerful. Using an [example from the comments in Smatch](<http://repo.or.cz/smatch.git/blob/7a4fdad6e866cc5775322531813973e1f5c1d393:/smatch_implied.c#l17>):

```
* Imagine we have this code:
        * foo = 1;
        * if (bar)
        *         foo = 99;
        * else
        *         frob();
        *                   //  <-- point #1
        * if (foo == 99)    //  <-- point #2
        *         bar->baz; //  <-- point #3
```

At point #3, assuming that `foo` and `bar` are local variables that cannot be changed by `frob()`, we know that `bar` is not null, as that is the only context in which `foo` could be 99. Very superficially, the code searches for previous states of `foo` that don't conflict with the current state, then assumes that the union of all the states that `bar` could have had at those times can accurately describe `bar` now. This is referred to in the code as an `implied` value; many tests make use of this implied value calculation.

The hooks that can be used to call functions to update or test these states can roughly be divided into three groups: state-management hooks, syntax-directed hooks, and derived hooks.

There are three state-management hooks, the merge hook (set with `add_merge_hook()`) that we have already met, a `pre_merge()` hook that is called a little earlier and allows state to be prepared for easier merging, and an `unmatched_state()` hook that can provide a default state. The implied value calculation finds it easier to always have an actual state so it doesn't need to deal with non-existent state; when necessary it requests the default value.

There are 35 [syntax-directed hooks](<http://repo.or.cz/smatch.git/blob/7a4fdad6e866cc5775322531813973e1f5c1d393:/smatch.h#l92>) that trigger on particular syntactic constructs. As well as `CONDITION_HOOK` mentioned earlier, there is `WHOLE_CONDITION_HOOK` for when you really need the whole condition rather than the parts, though no current checker uses it. There is also `EXPR_HOOK` for expressions, `STMT_HOOK` for statements, `ASSIGNMENT_HOOK`, `RETURN_HOOK`, and many more.

There is one extra hook that is syntax-based but quite different from the others: `add_scope_hook()` adds a hook that will only be called once, when the current scope completes. Smatch comes with [a checker](<http://repo.or.cz/smatch.git/blob/7a4fdad6e866cc5775322531813973e1f5c1d393:/check_memory.c>) that uses the scope hook to ensure that memory allocated with `malloc()` is either freed or stored somewhere before the variable that holds the allocation goes out of scope.

Derived hooks are built on top of the syntax-directed hooks, so they really belong in the next layer but it is convenient to introduce them here. There are currently two groups of derived hooks: function hooks and modification hooks. Function hooks trigger when a specific function is called rather than just on any function call. The above-mentioned memory-allocation checker registers a function hook for each of the standard memory-allocation functions (`malloc()`, `calloc()`, `realloc()`, etc.), which is more convenient than registering a `FUNCTION_CALL_HOOK` and then checking the function name.

The modification hooks attach to assignment, increment, and decrement operators, as well as to function calls that pass the address of a variable. Tracking all of these together allows all possible modifications to a variable to be handled through a single interface.

#### Smatch extra: tracking the values of variables

The second layer of Smatch, built on top of the framework of checkers, states, and hooks, is a collection of service providers. These include the derived hooks mentioned earlier and a few other useful features, but the most substantial service in this layer is referred to as `SMATCH_EXTRA`, which attempts to track the value of every variable.

Each integer variable, including enums, and each pointer is potentially tracked by this "extra" module with a state (sometimes referred to as "estate" — extra state) that lists ranges of possible values. The estate also lists any other variables that are known to hold the same value, either due to a recent assignment or a recent equality comparison. For pointers, some [arbitrary ranges](<http://repo.or.cz/smatch.git/blob/5ff855790a51929cd233c9256c49dd810f23d3d2:/smatch.h#l482>) are assigned to memory regions such as the stack, the heap, and initialized data, so that they can be differentiated from each other and from null pointers. For integers, more precise numbers can often be determined and used to, for example, check if an array reference might be out of bounds.

Smatch can perform all the standard arithmetic and bit-wise operations on signed and unsigned numbers to achieve constant folding similar to that provided in sparse. When it doesn't know the value of a variable, it assumes the full range of possible values; after a comparison it can narrow that range down. When states are merged, it can merge the lists of ranges that a variable might have so as to keep a best-possible understanding of the current value. This tracking of ranges is closely tied in with the implied value calculations mentioned already, so `get_implied_value()`, `get_implied_max()`, or `implied_not_equal()` are the sorts of functions that are often used to collect information about the current value of a given variable.

#### Smatch: so much more

While this layered model should provide a reasonably good picture of the overall approach of Smatch, it is a long way from the full story. Around the edges of this infrastructure are various other mechanisms for making useful analysis such as:

  * An interface to a database for storing and recalling various details of functions and global variables so that inter-file analysis is possible.

  * A record of the locations of all macro definitions so that code can be treated differently if it came from a macro expansion.

  * A [collection of scripts](<http://repo.or.cz/smatch.git/tree/5ff855790a51929cd233c9256c49dd810f23d3d2:/smatch_scripts>) for post-processing output, for extracting reports from the inter-file database, and for making it easier to run Smatch.

Ultimately the only way to know for sure what Smatch can do is to examine the code and look at what the current 100 or so checkers actually do. Building on that shouldn't be too hard once you understand the overall approach, and providing you know what patterns you actually want to check for.

Smatch has already found a couple of bugs for me in a project I'm working on, and I have an idea for an extra check that I would like it to perform. Now that I understand how Smatch works, it is nearly time to try to build that test. I'll let you know how it goes.
