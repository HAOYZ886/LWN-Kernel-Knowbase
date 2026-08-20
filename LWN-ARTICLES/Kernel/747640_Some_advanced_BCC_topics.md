---
title: Some advanced BCC topics
url: https://lwn.net/Articles/747640/
date: "February 22, 2018"
category: BPF
author: "February 22, 2018 This article was contributed by Matt Fleming BPF in the kernel"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

February 22, 2018

This article was contributed by [Matt Fleming](<http://www.codeblueprint.co.uk/>)

* * *

[BPF in the kernel](<https://lwn.net/Articles/740157/>)

The BPF virtual machine is working its way into an increasing number of kernel subsystems. [The previous article in this series](<https://lwn.net/Articles/742082/>) introduced the BPF Compiler Collection (BCC), which provides a set of tools for working with BPF. But there is more to BCC than a set of administrative tools; it also provides a development environment for those wanting to create their own BPF-based utilities. Read on for an exploration of that environment and how it can be used to create programs and attach them to tracepoints. 

The BCC runtime provides a macro, [`TRACEPOINT_PROBE`](<https://github.com/iovisor/bcc/blob/master/docs/reference_guide.md#3-tracepoints>), that declares a function to be attached to a tracepoint that will be called every time the tracepoint fires. The following snippet of C code shows an empty BPF program that runs every time `kmalloc()` is called in the kernel: 

```
TRACEPOINT_PROBE(kmem, kmalloc) {
            return 0;
        }
```

The arguments to this macro are the category of the tracepoint and the event itself; this translates directly into the debugfs file system hierarchy layout (e.g. `/sys/kernel/debug/tracing/events/_category_ /_event_ /`). In true BCC-make-things-simple fashion, the tracepoint is automatically enabled when the BPF program is loaded. 

The `kmalloc()` tracepoint is passed a number of arguments, which are described in the associated `format` file. Tracepoint arguments are accessible in BPF programs through the magic `args` variable. For our example, we care about `args->call_site`, which is the kernel instruction address of the `kmalloc()` call. To keep a count of the different kernel functions that call `kmalloc()`, we can store a counter in a hash table and use the call-site address as an index. 

While BCC provides access to the full range of data structures exported by the kernel (and covered in the first article of this series), the two most frequently used are `BPF_HASH` and `BPF_TABLE`. Fundamentally, all of BCC's data structures are maps, and higher-level data structures are built on top of them; the most basic of these is `BPF_TABLE`. The `BPF_TABLE` macro takes a type of table (`hash`, `percpu_array`, or `array`) as an argument, and other macros, such as `BPF_HASH` and `BPF_ARRAY` are simply wrappers around `BPF_TABLE`. Because all data structures are maps, they all support the same core set of functions, including `map.lookup()`, `map.update()`, and `map.delete()`. (There are also some map-specific functions such as `map.perf_read()` for `BPF_PERF_ARRAY` and `map.call()` for `BPF_PROG_ARRAY`.) 

Returning to our example program, we can store the kernel instruction-pointer address of the `kmalloc()` call-site (and the number of times it was called) using a `BPF_HASH` map and post-process it with Python. Here is the entire script, including the BPF program. 

```
#!/usr/bin/env python
    
        from bcc import BPF
        from time import sleep
    
        program = """
            BPF_HASH(callers, u64, unsigned long);
    
            TRACEPOINT_PROBE(kmem, kmalloc) {
                u64 ip = args->call_site;
                unsigned long *count;
                unsigned long c = 1;
    
                count = callers.lookup((u64 *)&ip);
                if (count != 0)
                    c += *count;
    
                callers.update(&ip, &c);
    
                return 0;
            }
        """
        b = BPF(text=program)
    
        while True:
            try:
                sleep(1)
                for k,v in sorted(b["callers"].items()):
                    print ("%s %u" % (b.ksym(k.value), v.value))
                print
            except KeyboardInterrupt:
                exit()
```

The syntax for the `BPF_HASH` macro is described in the [BCC reference guide](<https://github.com/iovisor/bcc/blob/master/docs/reference_guide.md#2-bpf_hash>). The macro takes a number of optional arguments, but for most uses all you need to specify is the name of this hash table instance (`callers` in this example), the key data type (`u64`), and the value data type (`unsigned long`). BPF hash table entries are accessed using the `lookup()` function; if no entry exists for a given key, NULL is returned. `update()` will either insert a new key-value pair (if none exists) or update the value of an existing key. Thus, the BPF code for working with hashes can be quite compact because you can use a single function (`update()`) regardless of whether you're inserting a new item or updating an existing one. 

Once a count has been stored in the hash table, it can be processed with Python. Accessing the table is done by indexing the BPF object (called `b` in the example). The resultant Python object is a `HashTable` (defined in the BCC Python front end) and its items are accessed using the `items()` function. Note that [Python BCC maps](<https://github.com/iovisor/bcc/blob/master/docs/reference_guide.md#maps-1>) provide a different set of functions than [BPF maps](<https://github.com/iovisor/bcc/blob/master/docs/reference_guide.md#maps>). 

`items()` returns a pair of [Python c_long types](<https://docs.python.org/3/library/ctypes.html#ctypes.c_long>) whose values can be retrieved using the `value` member. For example, the following code from the example above iterates over all items in the `callers` hash table and prints the kernel functions (using the BCC `[BPF.ksym()](<https://github.com/iovisor/bcc/blob/master/docs/reference_guide.md#1-ksym>)` helper function to convert kernel addresses to symbols) that invoked `kmalloc()` and the number of calls: 

```
for k,v in sorted(b["callers"].items()):
    	print ("%s %u" % (b.ksym(k.value), v.value))
```

The output from this little program looks like: 

```
# ./example.py
        i915_sw_fence_await_dma_fence 4
        intel_crtc_duplicate_state 4
        SyS_memfd_create 1
        drm_atomic_state_init 4
        sg_kmalloc 7
        intel_atomic_state_alloc 4
        seq_open 504
        SyS_bpf 22
```

Though this example is relatively straightforward, larger tools will not be, and developers need ways to debug more complex tools. Thankfully, there are a few ways that BCC helps simplify the debugging process. 

#### Controlling BPF program compilation and loading

Whenever a Python `[BPF](<https://github.com/iovisor/bcc/blob/master/docs/reference_guide.md#1-bpf>)` object is instantiated, the BPF program source code contained within it is automatically compiled and loaded into the kernel. The compilation process can be controlled by passing compiler flag arguments in the `cflags` parameter to the BPF class constructor. These flags are passed directly to the Clang compiler, so any options that you might normally pass to the compiler can be used; all compiler warnings can be turned on with "`cflags=['-Wall']`", for instance. 

A popular use of `cflags` in the official BCC tools is to pass macro definitions. For example, the [xdp_drop_count.py](<https://github.com/iovisor/bcc/blob/master/examples/networking/xdp/xdp_drop_count.py>) script statically allocates an array with enough space for every online CPU using Python's `multiprocessing` library and Clang's `-D` flag: 

```
cflags=["-DNUM_CPUS=%d" % multiprocessing.cpu_count()])
```

The BPF class constructor also accepts a number of [debugging flags](<https://github.com/iovisor/bcc/blob/master/src/cc/bpf_module.h#L37>) in the `debug` argument. Each of these flags individually enables extra logging during either the compilation or the loading process. For example, the `DEBUG_BPF` flag causes the BPF bytecode to be output, which can be a last hope for those really troublesome bugs. This output looks like: 

```
0: (79) r1 = *(u64 *)(r1 +8)
        1: (7b) *(u64 *)(r10 -8) = r1
        2: (b7) r1 = 1
        3: (7b) *(u64 *)(r10 -16) = r1
        4: (18) r1 = 0xffff8801a6098a00
        6: (bf) r2 = r10
        7: (07) r2 += -8
        8: (85) call bpf_map_lookup_elem#1
        9: (15) if r0 == 0x0 goto pc+3
         R0=map_value(id=0,off=0,ks=8,vs=8,imm=0) R10=fp0
        10: (79) r1 = *(u64 *)(r0 +0)
         R0=map_value(id=0,off=0,ks=8,vs=8,imm=0) R10=fp0
        11: (07) r1 += 1
        12: (7b) *(u64 *)(r10 -16) = r1
        13: (18) r1 = 0xffff8801a6098a00
        15: (bf) r2 = r10
        16: (07) r2 += -8
        17: (bf) r3 = r10
        18: (07) r3 += -16
        19: (b7) r4 = 0
        20: (85) call bpf_map_update_elem#2
        21: (b7) r0 = 0
        22: (95) exit
        
        from 9 to 13: safe
        processed 22 insns, stack depth 16
```

This output comes directly from the in-kernel verifier and shows every instruction of bytecode emitted by Clang/LLVM, along with the register state on branch instructions. If this level of detail still isn't enough, the `DEBUG_BPF_REGISTER_STATE` flag generates even more verbose log messages. 

For run-time debugging, `bpf_trace_printk()` provides a `printk()`-style interface for writing to `/sys/kernel/debug/tracing/trace_pipe` from BPF programs; those messages can then be consumed and printed in Python using the `BPF.trace_print()` function. 

However, a major drawback of this approach is that, since the `trace_pipe` file is a global resource, it contains all messages written by concurrent writers, making it difficult to filter messages from a single BPF program. The preferred method is to store messages in a [`BPF_PERF_OUTPUT`](<https://github.com/iovisor/bcc/blob/master/docs/reference_guide.md#2-bpf_perf_output>) map inside the BPF program, then process them with `[open_perf_buffer()](<https://github.com/iovisor/bcc/blob/master/docs/reference_guide.md#2-open_perf_buffer>)` and `[kprobe_poll()](<https://github.com/iovisor/bcc/blob/master/docs/reference_guide.md#1-kprobe_poll>)`. An example of this scheme is provided in the `open_perf_buffer()` documentation. 

#### Using BPF with applications

This article has focused exclusively on attaching programs to kernel tracepoints, but you can also attach programs to the user-space tracepoints included with many popular applications using User Statically-Defined Tracing (USDT) probes. In the next and final article of this series, I'll cover the origin of USDT probes, the BCC tools that use them, and how you can add them to your own application.
