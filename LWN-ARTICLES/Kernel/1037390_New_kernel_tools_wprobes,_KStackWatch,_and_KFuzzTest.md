---
title: "New kernel tools: wprobes, KStackWatch, and KFuzzTest"
url: https://lwn.net/Articles/1037390/
date: "September 15, 2025"
category: Development tools
author: "By Jonathan Corbet September 15, 2025"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
September 15, 2025

The kernel runs in a special environment that makes it difficult to use many of the development tools that are available to user-space developers. Kernel developers often respond by simply doing without, but the truth is that they need good tools as much as anybody else. Three new tools for the tracking down of bugs have recently landed on the linux-kernel mailing list; here is an overview. 

#### wprobes

One nice feature long found in user-space debuggers is watchpoints — the ability to request a trap whenever a particular spot in memory is accessed. Watchpoints can be useful for finding out which code is guilty of corrupting a given variable, among other things. This feature could be especially useful in the kernel context, where many things are happening at once and the source of a problem could be anywhere in a large and complex system. But kernel developers have had to do without watchpoints, especially when working outside of virtualized environments. 

What the kernel _does_ have is [kprobes](<https://docs.kernel.org/trace/kprobes.html>), which enable the placement of debugging code (almost) anywhere within a running kernel. They are somewhat similar to a dynamic breakpoint inserted by a debugger, but they do not actually stop the execution of the kernel; instead, they print some (hopefully useful) information and continue on. 

[This patch series](<https://lwn.net/ml/all/175785897434.234168.6798590787777427098.stgit@devnote2>) from Masami Hiramatsu adds a new feature to the kprobes subsystem called "wprobes". A wprobe is similar to a user-space watchpoint, in that it traps accesses to a given range of memory; like kprobes, though, wprobes do not actually stop execution. But, with luck, they can be made to print enough information to help pinpoint the source of a problem. 

A watchpoint is set up by writing a cryptic string to `/sys/kernel/tracing/dynamic_events`, following this format: 

```
w:[GRP/][EVENT] [r|w|rw]@<ADDRESS|SYMBOL[+OFFS]> [FETCHARGS]
```

As an example (provided in the patch cover letter), this set of commands will create a watchpoint that will trigger whenever the kernel's `jiffies` variable is modified: 

```
# cd /sys/kernel/tracing
        # echo 'w:my_jiffies w@jiffies:8 value=+0($addr)' >> dynamic_events
        # echo 1 > events/wprobes/my_jiffies/enable
```

In the middle line, "`w:my_jiffies`" sets up a watch point named `my_jiffies`; "`w@jiffies:8`" targets writes to the eight-byte `jiffies` variable, and "`value=+0($addr)`" prints out the value written to that address. The final line then activates the watchpoint. 

There is also a mechanism that can create watchpoints dynamically when certain things happen; it enables watching dynamically created memory objects (slab allocations, for example) during their lifecycle. See the above-linked patch cover letter for an example, [this patch](<https://lwn.net/ml/all/175708429018.64001.14265428106568623070.stgit@devnote2>) for a documentation update describing wprobes, and [this patch](<https://lwn.net/ml/all/175708432404.64001.2572289500054005289.stgit@devnote2/>) for documentation on dynamic wprobe creation. 

#### KStackWatch

Corruption of the call stack can be an especially pernicious problem for developers in any context. The kernel can be built with stack canaries and others tools that will detect that a stack overflow has happened, but they cannot pinpoint exactly _when_ the problem occurred. The [KStackWatch patch series](<https://lwn.net/ml/all/20250912101145.465708-1-wangjinchao600@gmail.com>) from Jinchao Wang aims to fill that gap by providing a lightweight tool that can trap stack-corruption events. 

In essence, KStackWatch will keep an eye on a specific part of a given function's stack frame while that function is active. Like wprobes, it sets up a trap on access to the memory of interest, and prints out useful information when that memory is altered. The kernel's tracing infrastructure is used to enable this watchpoint on entry to the function of interest, and to disable it on return. 

The tool adds a new control file, `/proc/kstackwatch`, running counter to the usual practice of putting such files in debugfs or tracefs. The administrator can enable the tool by writing a string of this form to that file: 

```
function+ip_offset[+depth] [local_var_offset:local_var_len]
```

Here, `function` is the name of the function of interest, and `ip_offset` indicates where, in the function, the entry probe should be placed. It's not entirely clear why that offset is needed, or how a user would determine what its value should be. The `depth` value can be used with recursive functions; it specifies which level of recursion should be watched. The `local_var_offset` and `local_var_len` parameters indicate the location and size of a specific variable on the stack to watch, expressed as an offset from the stack pointer at entry. If these values are not provided, KStackWatch will watch the stack canary instead. 

The current stack watch is global across the system, there can only be one active at any given time. Writing a new configuration to `/proc/kstackwatch` will remove the previous watch; writing an empty string will disable the tool entirely. 

When a write to the watched variable is detected, KStackWatch will output some diagnostic information to the system log. There is a module parameter, `panic_on_catch`, that will cause an immediate system panic when a write is detected. There does not appear to be a way to change that parameter without unloading and reloading the module. 

This series is in its fourth revision as of this writing; this revision, happily, includes [a basic documentation file](<https://lwn.net/ml/all/20250910053147.1152253-11-wangjinchao600@gmail.com>) describing KStackWatch. There has been some interesting interplay with the wprobes patch set, as each has adapted useful code from the other; both appear to be approaching a state of readiness. 

#### KFuzzTest

Fuzz testing of the kernel is not particularly new; tools like syzkaller have been [exposing kernel bugs](<https://lwn.net/Articles/677764/>) for the better part of a decade. This testing, though, is all run from user space, so the code it can exercise is limited by the kernel's user-space API. The [KFuzzTest subsystem](<https://lwn.net/ml/all/20250901164212.460229-1-ethan.w.s.graham@gmail.com>), proposed by Ethan Graham, is an attempt to give fuzz testers a way to reach inside the kernel and abuse the interfaces of low-level functions directly. 

To arrange for a kernel function to be stressed by KFuzzTest, a developer must set up some scaffolding within the kernel, most likely in the same source file as the target function. The first step is to define a structure that encapsulates the arguments to that function. If we wanted to be able to test this internal kernel function: 

```
int mangle_data(const u8 *data, size_t len) { /* do something interesting */ }
```

We would start with this definition: 

```
struct mangle_data_input {
        	const u8 *data;
    	size_t len;
        };
```

Once that is in hand, it is time to define the test itself. That is done with the `FUZZ_TEST()` macro: 

```
FUZZ_TEST(test_mangle_data, mangle_data_input)
        {
        	/* Constraints and annotations here, then ... */
    	mangle_data(arg->data, arg->len);
        }
```

This incantation defines a test named `test_mangle_data`. Within the body of the declaration, the variable `arg` will point to a `mangle_data_input` structure containing arguments to pass to the function; the call to the function to test is made at the end of this declaration. User space will be able to invoke this test as described below, providing whatever evil data it thinks might expose bugs. It is worth noting that KFuzzTest will not, by itself, detect those bugs; it just runs the function with the provided data. So KFuzzTest will normally be run alongside tools like [KASAN](<https://docs.kernel.org/dev-tools/kasan.html>) to detect when things go off the rails. 

The test can include constraints on the data that is passed in. For example, if the definition of `test_mangle_data` includes a line like: 

```
KFUZZTEST_EXPECT_NOT_NULL(mangle_data_input, data);
```

Any input value will be tested to ensure that the `data` pointer is non-NULL. Should that condition not be met, the test itself will not be run. There is a whole set of `KFUZZTEST_EXPECT_` macros that can be used to constrain the input data to the function; they can be found in [this patch](<https://lwn.net/ml/all/20250901164212.460229-3-ethan.w.s.graham@gmail.com>). 

The test definition can also contain annotations with further information about the input data for the function. For example: 

```
KFUZZTEST_ANNOTATE_LEN(mangle_data_input, len, data);
```

documents that `len` is meant to be the length of the `data` array. Other possible annotations indicate that a given argument is an array, or that it is expected to contain a C string. Annotations do not affect the running of the test itself. The constraints and annotations are compiled into a special section of the kernel executable, where user-space tools can find them. 

On the user-space side, KFuzzTest sets up a debugfs directory (called `fuzztest`) with a subdirectory for each defined test. A user-space tool can use this directory to discover the available tests, but it must still read the kernel image file to obtain constraint and annotation information. It is also necessary to build the kernel with DWARF debugging information to provide information about the layout of structures; it is not clear why the kernel's BTF information, which is usually present and rather more compact, is not used for this purpose. 

Running a test is a matter of writing some data to the `input` file in the test's directory; for our example above, that file would be `.../kfuzztest/test_mangle_data/input`. The format of that data is _not_ simple, though. The input to a function may include pointers to a set of complex, pointer-connected data structures; KFuzzTest allows the testing tool to provide that whole set as input. To do so, user space must serialize the data into the format expected by KFuzzTest, then write the result to the `input` file. 

The patch series includes a tool (`kfuzztest-bridge`) that can be used to run a test with random data; its input, too, is on the complex side. See [this documentation patch](<https://lwn.net/ml/all/20250901164212.460229-6-ethan.w.s.graham@gmail.com>) for details on how all of this stuff works, and [this patch](<https://lwn.net/ml/all/20250901164212.460229-7-ethan.w.s.graham@gmail.com>) for a couple of example tests. This work is still in the RFC stage, but there does appear to be a certain amount of interest in it, so it is likely to pass out of that stage at some point.
