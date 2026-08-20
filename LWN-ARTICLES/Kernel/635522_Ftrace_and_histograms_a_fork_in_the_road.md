---
title: "Ftrace and histograms: a fork in the road"
url: https://lwn.net/Articles/635522/
date: "March 4, 2015"
category: "BPF-Tracing; Development tools-Kernel tracing; Ftrace; Tracing-Ftrace"
author: "By Jonathan Corbet March 4, 2015"
---

By **Jonathan Corbet**  
March 4, 2015

The kernel's "ftrace" tracing machinery is useful for obtaining a great deal of information about what is going on inside a running kernel. What ftrace generally does not provide is analysis features to "boil down" tracing data into a more useful format. Tom Zanussi's "[hist triggers](<https://lwn.net/Articles/635280/>)" patch could change that situation, but it has exposed a significant difference of opinion over how such capabilities should be implemented in the kernel. 

The idea behind hist triggers is simple enough: a user interested in a histogram of tracing data would write a command string to the appropriate ftrace control file specifying the parameters of the histogram. For example, one could look at `kmalloc()` calls with a command like: 

```
# echo 'hist:key=call_site:val=bytes_req' > \
               /sys/kernel/debug/tracing/events/kmem/kmalloc/trigger
```

Here, the `hist:` prefix indicates that histogram output is desired. The `key=` and `val=` parameters describe the axes of the histogram; in this case, the user will get the total amount of memory requested from each location where `kmalloc()` is called. One obtains the results by reading the `hist` file that magically pops up in the tracing control directory: 

```
# cat /sys/kernel/debug/tracing/events/kmem/kmalloc/hist
        trigger info: hist:keys=call_site:vals=bytes_req:sort=hitcount:size=2048 [active]
    
        call_site: 18446744071581750326 hitcount:          1  bytes_req:         24
        call_site: 18446744071583151255 hitcount:          1  bytes_req:         32
        call_site: 18446744071582443167 hitcount:          1  bytes_req:        264
        call_site: 18446744072104099935 hitcount:          2  bytes_req:        464
        call_site: 18446744071579323550 hitcount:          3  bytes_req:        168
        [...]
```

There are additional options that can, for example, turn the call-site address into a symbolic location. This example was taken from [the documentation posted with Tom's patch set](<https://lwn.net/Articles/635523/>); many more examples and details can be found there. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Tom thinks that this sort of functionality will be useful to a wide variety of tracing users. Indeed, it may reduce the need for more sophisticated tools: 

A surprising number of typical use cases can be accomplished by users via this simple mechanism. In fact, a large number of the tasks that users typically do using the more complicated script-based tracing tools, at least during the initial stages of an investigation, can be accomplished by simply specifying a set of keys and values to be used in the creation of a hash table. 

Nobody seems to disagree that this would be a nice feature to have, but, still, criticism of the patch set came from two directions. Ftrace maintainer Steve Rostedt [complained](<https://lwn.net/Articles/635524/>) that the tracepoint code generating the histograms performs memory allocations; those allocations are necessary to maintain the hash table used to hold the histogram data. Tracepoint callbacks can be called with all sorts of locks held; allocating memory in such a situation is not a safe thing to do. So, Steve said, that aspect of the patch set is a "showstopper." Future versions of the patch set will, thus, have to accumulate this data without allocating memory in the tracepoint callbacks. 

A different type of criticism came from Alexei Starovoitov, the developer behind the [eBPF work](<https://lwn.net/Articles/612878/>) that has gone into the kernel over the last year. One of the use cases for a tool like eBPF is to allow users to gather data in kernel space and generate output in forms like histograms. Alexei thus duly [suggested](<https://lwn.net/Articles/635527/>) that eBPF should be used to implement Tom's histogram functionality. Rather than parse the commands in the kernel, though, Alexei would like to see the development of a tool that would parse the same commands in user space and load an eBPF program to do the actual work. 

To Tom, the idea [seemed](<https://lwn.net/Articles/635528/>) "silly"; a lot of work would be required to implement the functionality that already exists. He saw the request as an attempt to force the use of eBPF on users who may not want to deal with it. Alexei [responded](<https://lwn.net/Articles/635529/>) by saying: 

Your 'hist->bpf' tool could have been first to solve 'bpf hard to use' problem and over time it could have evolved into full dtrace alternative. Whereas by adding 'hist:keys=..' parsing to kernel you'll be stuck with it and somebody else's dtrace-like tool will supersede it. 

Tom [remained unimpressed](<https://lwn.net/Articles/635530/>): 

I think there's some misunderstanding there - it was never my intent to create a full dtrace alternative, really the idea was (and has been, even before there was any such thing as ebpf in the kernel) only to provide access to some low-hanging fruit via the standard read/write interfaces people are used to with ftrace. 

In the end, there is an important question to answer here. The eBPF subsystem provides a mechanism by which a great deal of interesting tracing functionality could be implemented without having to hardwire the logic in the kernel. Now that eBPF is here, adding new tracing modes as more C code in the kernel could lead to duplicated functionality that needs to be supported indefinitely, even if, someday, an alternative implemented in eBPF draws most of the users. 

On the other hand, the current interface to ftrace, wherein users write simple control strings to a set of virtual files, appeals to a lot of users. It is relatively easy to work with, does not require any additional tools to use, and is straightforward to script. Some of those users would not be pleased if they felt pushed to move over to an interface requiring the compilation and loading of eBPF programs to get their work done. 

This has the look of a debate that could go on for some time. In the absence of a decision by decree from a suitably placed subsystem maintainer, it seems unlikely that the developers involved will settle on a single approach to the problem of how to add new tracing features. The kernel's tracing subsystem is arguably at a fork in the road, but we may not know which branch will be taken for a while yet.
