---
title: Using the TRACE_EVENT() macro (Part 2)
url: https://lwn.net/Articles/381064/
date: "March 31, 2010"
category: "Development tools-Kernel tracing; Tracing"
author: "March 31, 2010 This article was contributed by Steven Rostedt"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

March 31, 2010

This article was contributed by Steven Rostedt

In [Part 1](<http://lwn.net/Articles/379903/>), the process of creating a tracepoint in the core kernel was explained. This article continues from there with tricks to lower the tracepoint footprint by using the `DECLARE_EVENT_CLASS()` macro. In addition, the macros used to build the `TP_STRUCT__entry` fields are described and the `TP_printk` helper functions are explained. 

#### Saving space by using `DECLARE_EVENT_CLASS()`

Every tracepoint that is created with the `TRACE_EVENT()` macro creates several functions that allows perf and Ftrace to interact with the tracepoint automatically. Since these functions have unique prototypes (defined by the `TP_PROTO` and `TP_ARGS` macros in the `TRACE_EVENT()` definition), reference unique structures (defined by the `TP_STRUCT__entry` macro), assign them uniquely to the ring buffer (as defined by `TP_fast_assign`), and has a unique way to print out the data (defined in `TP_printk`), there is very little that the `TRACE_EVENT()` macro can do to reuse code. That means that every `TRACE_EVENT()` defined will increase the footprint of the kernel, which is enough to make quite a difference with hundreds of `TRACE_EVENT()` macros.

```
text          data     bss     dec     hex filename
          452114          2788    3520  458422   6feb6 fs/xfs/xfs.o.notrace
          996954         38116    4480 1039550   fdcbe fs/xfs/xfs.o.trace
```

The XFS filesystem declares over a hundred separate trace events. The data section increased substantially, but that is expected because each event has a corresponding structure with a set of function pointers attached to it. What was not acceptable, though, was that enabling the trace events causes the `xfs.o` text section to double in size! 

That pushed an effort to find a way to condense trace events. The obvious place to start was to have several events, which record the same structured data, share their functions. If two events have the same `TP_PROTO`, `TP_ARGS` and `TP_STRUCT__entry`, there should be a way to have these events share the functions that they use. This was the motivation for the new macro `DECLARE_EVENT_CLASS()` (originally called `TRACE_EVENT_TEMPLATE()`) and `DEFINE_EVENT()`. 

The `DECLARE_EVENT_CLASS()` macro has the exact same format as `TRACE_EVENT()`: 

```
DECLARE_EVENT_CLASS(sched_wakeup_template,
    
            TP_PROTO(struct rq *rq, struct task_struct *p, int success),
    
            TP_ARGS(rq, p, success),
    
            TP_STRUCT__entry(
                    __array(        char,   comm,   TASK_COMM_LEN   )
                    __field(        pid_t,  pid                     )
                    __field(        int,    prio                    )
                    __field(        int,    success                 )
                    __field(        int,    target_cpu              )
            ),
    
            TP_fast_assign(
                    memcpy(__entry->comm, p->comm, TASK_COMM_LEN);
                    __entry->pid            = p->pid;
                    __entry->prio           = p->prio;
                    __entry->success        = success;
                    __entry->target_cpu     = task_cpu(p);
            ),
    
            TP_printk("comm=%s pid=%d prio=%d success=%d target_cpu=%03d",
                      __entry->comm, __entry->pid, __entry->prio,
                      __entry->success, __entry->target_cpu)
       );
```

This creates a trace framework that can be used by multiple events. The `DEFINE_EVENT()` macro is used to create trace events defined by `DECLARE_EVENT_CLASS()`: 

```
DEFINE_EVENT(sched_wakeup_template, sched_wakeup,
                    TP_PROTO(struct rq *rq, struct task_struct *p, int success),
                    TP_ARGS(rq, p, success));
       DEFINE_EVENT(sched_wakeup_template, sched_wakeup_new,
                    TP_PROTO(struct rq *rq, struct task_struct *p, int success),
                    TP_ARGS(rq, p, success));
```

The example above creates two trace events `sched_wakeup` and `sched_wakeup_new`. The `DEFINE_EVENT()` macro requires 4 parameters: 

```
DEFINE_EVENT(class, name, proto, args)
```

  * class - the name of the class created with `DECLARE_EVENT_CLASS()`. 
  * name - the name of the trace event. 
  * proto - the prototype that is the same as `TP_PROTO` in the `DECLARE_EVENT_CLASS()`. 
  * args - the arguments of the prototype that is the same as `TP_ARGS` in `DECLARE_EVENT_CLASS()`. 

Unfortunately, due to the limitations of the C preprocessor, the `DEFINE_EVENT()` macro needs to repeat the prototype and arguments of the `DECLARE_EVENT_CLASS()`. 

Because several of the tracepoints in XFS are very similar, using the `DECLARE_EVENT_CLASS()` brought down the text and bss size quite substantially. 

```
text          data     bss     dec     hex filename
          452114          2788    3520  458422   6feb6 fs/xfs/xfs.o.notrace
          996954         38116    4480 1039550   fdcbe fs/xfs/xfs.o.trace
          638482         38116    3744  680342   a6196 fs/xfs/xfs.o.class
```

To keep the footprint of trace events down, try to consolidate events using the `DECLARE_EVENT_CLASS()` and `DEFINE_EVENT()` macros. There is no advantage to using the `TRACE_EVENT()` macro over the other two. In fact, the `TRACE_EVENT()` macro is now defined as just: 

```
#define TRACE_EVENT(name, proto, args, tstruct, assign, print) \
    	   DECLARE_EVENT_CLASS(name,			          \
    			        PARAMS(proto),		          \
    			        PARAMS(args),		          \
    			        PARAMS(tstruct),	          \
    			        PARAMS(assign),		          \
    			        PARAMS(print));		          \
    	   DEFINE_EVENT(name, name, PARAMS(proto), PARAMS(args));
```

Note that the `PARAMS` macro allows the arguments to contain commas and not be mistaken as multiple parameters of `DECLARE_EVENT_CLASS()` or `DEFINE_EVENT()`. 

#### `TP_STRUCT__entry` macros

The first article mentioned the `__field` and `__array` macros used to create the structure format of the event that is stored in the ring buffer. The `__field(type, item)` declared a field in the structure called `item` of type `type` (i.e. `type item;`). The `__array(type, item, len)` declared a static array called `item` with `len` elements of type `type` (i.e. `type item[len];`). Those two are the most common, but there are other macros that allow for more complex storage into the ring buffer. 

#### `__field_ext(type, item, filter_type)`

The `__field_ext` macro is mainly used for helping the event filter. The event filter (to be discussed in an upcoming article) allows the user to filter events based on the contents of its fields. The `type` and `item` are the same as the fields used by `__field`, the `filter_type` is an enum. Currently only the following values are used: 

  * `FILTER_OTHER` \- equivalent to the standard `__field()` macro. 
  * `FILTER_PTR_STRING` \- the field points to a string outside the ring buffer. 

The `FILTER_PTR_STRING` and `__field_ext` are currently only used by the big kernel lock tracepoints. These fields point to the function and file name that contain the tracepoint, which triggers when the big kernel lock is taken or released. This extension is not recommended since it makes the field useless for user-space tools that read the ring buffer in binary format. The big kernel lock tracepoints are an exception because they are currently being used to remove the big kernel lock, so hopefully these tracepoints will be removed from the kernel along with the big kernel lock. 

Fields defined by the `__field_ext` macro are assigned into the ring buffer in `TP_fast_assign` the same way that fields defined by `__field` are. 

#### `__string(item, src)`

The `__string` macro is used to record a variable length string, which must be null terminated. The first parameter is the name of the field in `TP_STRUCT__entry`, the second parameter is the source that will fill the string. For example, in the irq_handler_entry tracepoint's `TP_STRUCT__entry`: 

```
__string(        name,    action->name    )
```

The variable `action` is declared as one of the tracepoint's parameters. The `__string` macro will allocate enough space in the ring buffer and place the string at the end of the event data. To assign the string in the `TP_fast_assign`: 

```
__assign_str(name, action->name);
```

This will copy the string (`action->name`) into the reserved space in the ring buffer. To output the string, in `TP_printk`: 

```
TP_printk("irq=%d name=%s", __entry->irq, __get_str(name))
```

The `__get_str` macro returns a reference to the dynamic string in the `__entry` structure. 

#### `__dynamic_array(type, item, len)`

If more control is needed over a dynamic string or variable length array that is not a string, `__dynamic_array` can be used. The `__dynamic_array` macro is used to implement the `__string` macro. It takes three parameters: the `type` and `item` are the same as for the `__field` macro, but the third gives how to determine the length. For example, the `block_rq_with_error` tracepoint has the following: 

```
__dynamic_array( char,  cmd,    blk_cmd_buf_len(rq)     )
```

The call to `blk_cmd_buf_len()` will determine the length of the array needed to save the data. 

To assign a dynamic array field in `TP_fast_assign`, another macro is needed to get a reference to the array: `__get_dynamic_array(item)`. Note, that since the `block_rq_with_error` tracepoint defines a dynamic array that is a string, it uses the macro `__get_str(item)` instead: 

```
blk_dump_cmd(__get_str(cmd), rq);
```

The `blk_dump_cmd()` just fills the `cmd` array with data determined by the `rq` variable. The tracepoint can do this because the `__get_str` macro is defined as: 

```
#define __get_str(field) (char *)__get_dynamic_array(field)
```

Either `__get_dynamic_array` or `__get_str` can be used in the `TP_printk` macro to get a reference to the dynamic array. 

#### `TP_printk` helper functions

There are four `TP_printk` helper functions, two of which were already described in the previous section (`__get_str` and `__get_dynamic_array`). The other two helper functions are more complex and deal with mapping numbers to names. 

#### `__print_flags(flags, delimiter, values)`

Being able to see the values of flags in a field as symbolic names instead of numbers makes reading a trace much easier. Imagine having to manually parse `kmalloc()` GFP flags of 0x80d0 instead of `GFP_KERNEL|GFP_ZERO`. 

The first two parameters of the `__print_flags` are simply the variable that contains the flags (`__entry->gfp_flags`) and a string delimiter to use between flags if more than one is found ("|"). The delimiter may also be NULL or an empty string (""). The third parameter is an array of structures of the type: 

```
struct trace_print_flags {
               unsigned long          mask;
               const char             *name;
       };
```

The `module_load` tracepoint contains a good example of using `__print_flags`: 

```
TP_printk("%s %s", __get_str(name), __print_flags(flags, "",
                          { (1UL << TAINT_PROPRIETARY_MODULE),    "P" },
                          { (1UL << TAINT_FORCED_MODULE),         "F" },
                          { (1UL << TAINT_CRAP),                  "C" })
```

Depending on which taint flag is set, the corresponding letter ("P", "F", and/or "C") will be displayed. If the value of the flags field is not found within the `values` parameter, then the value of the flags parameter is converted to a hex string and that is returned. If no bit is set in the flags parameter, then `__print_flags` returns an empty string. Note that `__print_flags` internally terminates the `values` array, so no explicit termination is required. 

Alert readers will have noticed that the previous example of the `kmalloc` GFP flags used a complex bit mask. `GFP_KERNEL` is not a single bit, but is made up of multiple bits. A mask in `values` can contain more than one bit. `__print_flags` will iterate through `values`, and will use the first match for any particular set of bits. `GFP_KERNEL` is made up of `(__GFP_WAIT | __GFP_IO | __GFP_FS)`. The `kmalloc` tracepoint passes in the `GFP_KERNEL` mask before each of the single bit values. This allows `__print_flags` to pick the `GFP_KERNEL` over selecting the individual flags. If one of the three flags that make up `GFP_KERNEL` was listed in the values before `GFP_KERNEL`, then the individual flags would be in the output instead of printing `GFP_KERNEL`. Any remaining flag will also be parsed (as was `GFP_ZERO`). If bits are still set after all values have been applied, then those bits will show up as a hex number at the end following the delimiter. 

#### `__print_symbolic(val, values)`

The `__print_symbolic` function is very similar to `__print_flags` except that it only produces output for exact matches. The `values` field is still an array of `struct trace_print_flags` but the mask must match exactly to `val` in order to have it print `name`. If no match is found, `val` is converted to a hex string, which is returned. No delimiter is needed since only one value is returned by `__print_symbolic`. Here's an example of its use by the irq tracepoints: 

```
#define softirq_name(sirq) { sirq##_SOFTIRQ, #sirq }
       #define show_softirq_name(val)                          \
               __print_symbolic(val,                           \
                                softirq_name(HI),              \
                                softirq_name(TIMER),           \
                                softirq_name(NET_TX),          \
                                softirq_name(NET_RX),          \
                                softirq_name(BLOCK),           \
                                softirq_name(BLOCK_IOPOLL),    \
                                softirq_name(TASKLET),         \
                                softirq_name(SCHED),           \
                                softirq_name(HRTIMER),         \
                                softirq_name(RCU))
       [...]
               TP_printk("vec=%d [action=%s]", __entry->vec,
                         show_softirq_name(__entry->vec))
```

Notice how a helper macro is used to set up the values. This is recommended because macros will be evaluated before they show up in the output format, but functions will not. User-space tools will still be able to parse this because a macro was used rather than a function. 

#### A quick demo

To get a better understanding of what is happening with the events, the following contains some simple usage of event tracing. The examples assume that the user has changed directories to `tracing` in debugfs (usually, but not always, `/sys/kernel/debug/tracing`). Also notice that the prompt contains '#' which signifies that these operations require a privileged user: 

```
[tracing] # echo 1 > events/module/module_load/enable 
       [tracing] # insmod /tmp/taintme.ko 
       [tracing] # insmod /tmp/gpl-nice.ko
       [tracing] # cat trace
       # tracer: nop
       #
       #           TASK-PID    CPU#    TIMESTAMP  FUNCTION
       #              | |       |          |         |
                 insmod-1812  [003] 469717.724908: module_load: taintme P
                 insmod-1814  [003] 470058.525771: module_load: gpl_nice
```

The `taintme.ko` module is a module I wrote that does nothing, but does not have a GPL-compliant license. This causes the "P" taint flag to appear. Notice that no flag appeared for `gpl_nice` (which, as the name implies, _does_ have a GPL license). Remember, if no bit is set in the flags passed to the `__print_flags` macro, an empty string is returned. 

```
[tracing] # echo irq_handler_entry softirq_entry > set_event
       [tracing] # cat trace | head
       # tracer: nop
       #
       #           TASK-PID    CPU#    TIMESTAMP  FUNCTION
       #              | |       |          |         |
                 <idle>-0     [002] 470574.178475: irq_handler_entry: irq=26 handler=hpet4
                 <idle>-0     [002] 470574.178485: softirq_entry: softirq=1 action=TIMER
                 <idle>-0     [002] 470574.178492: softirq_entry: softirq=7 action=SCHED
                 <idle>-0     [002] 470574.178495: softirq_entry: softirq=9 action=RCU
                 <idle>-0     [000] 470574.178678: irq_handler_entry: irq=35 handler=eth0
                 <idle>-0     [000] 470574.178684: softirq_entry: softirq=3 action=NET_RX
```

Notice that this command used the `set_event` file to enable tracing. Using this file or the `enable` file within the `events` directory act the same way. Because the tracepoint names are (at least so far) unique, just echoing the name into `set_event` is the equivalent of enabling the tracepoint using `events/irq/irq_handler_entry/enable` for example. For enabling multiple tracepoints at once it is usually more convenient to use the `set_event` file, but when activating a singe event, all events in a subsystem, or all events it is more convenient to use the `enable` files. More details about using the event tracer will be explained in an upcoming article. 

The IRQ and soft IRQ events shown above illustrate the output of a dynamic string and use of the `__print_symbols` helper function. The `irq_handle_entry` saves the name of the interrupt device (`hpet4` and `eth0`) using a dynamic string to display the name in the trace. The `softirq_entry` uses the `__print_symbols` helper function to convert the number of the soft IRQ vector into a matching name that it represents (`TIMER`, `SCHED`, `RCU`, and `NET_RX`). 

#### Coming in Part 3

Part 3 will look at defining tracepoints outside of the `include/trace/events` directory (for modules and architecture-specific tracepoints) along with a look at how the `TRACE_EVENT()` macro does its magic. It will also include some more examples of how the tracepoints are used with Ftrace.
