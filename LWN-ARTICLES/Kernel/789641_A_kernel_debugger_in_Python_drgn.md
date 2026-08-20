---
title: "A kernel debugger in Python: drgn"
url: https://lwn.net/Articles/789641/
date: "May 29, 2019"
category: "Debugging; Development tools-Kernel debugging"
author: "By Jake Edge May 29, 2019 LSFMM"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
May 29, 2019

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2019/>)

A kernel debugger that allows Python scripts to access data structures in a running kernel was the topic of Omar Sandoval's plenary session at the 2019 Linux Storage, Filesystem, and Memory-Management Summit (LSFMM). In his day job at Facebook, Sandoval does a fair amount of kernel debugging and he found the existing tools to be lacking. That led him to build [drgn](<https://github.com/osandov/drgn>), which is a debugger built into a Python library. 

Sandoval began with a quick demo of drgn (which is pronounced "dragon"). He was logged into a virtual machine (VM) and invoked the debugger on the running kernel with `drgn -k`. With some simple Python code in the REPL ([read-eval-print loop](<https://en.wikipedia.org/wiki/Read%E2%80%93eval%E2%80%93print_loop>)), he was able to examine the superblock of the root filesystem and loop through the inodes cached in that superblock—with their paths. Then he did "something a little fancier" by only listing the inodes for files that are larger than 1MB. It showed some larger kernel modules, libraries, systemd, and so on. 

He mostly works on Btrfs and the block layer, but he also tends to debug random kernel problems. Facebook has so many machines that there are "super rare, one-in-a-million bugs" showing up all the time. He often volunteers to take a look. In the process he got used to tools like GDB, crash, and eBPF, but found that he often wanted to be able to do arbitrarily complex analysis of kernel data structures, which is why he ended up building drgn. 

[ ![\[Omar Sandoval\]](https://static.lwn.net/images/2019/lsf-sandoval-sm.jpg) ](<https://lwn.net/Articles/789643/>)

GDB has some nice features, he said, including the ability to pretty-print types, variables, and expressions. But it is focused on a breakpoint style of debugging, which he cannot do on production systems. It has a scripting interface, but it is clunky and just wraps the existing GDB commands. 

[Crash](<http://man7.org/linux/man-pages/man8/crash.8.html>) is purpose built for kernel debugging; it knows about linked lists, structures, processes, and so on. But if you try to go beyond those things, you will hit a wall, Sandoval said. It is not particularly flexible; when he used it, he often had to dump a bunch of state and then post-process it. 

BPF and BCC are awesome and he uses them all the time, but they are limited to times when you can reproduce the bug live. Many of the bugs he looks at are something that happened hours ago and locked up the machine, or he got a core dump and wants to understand why. BPF doesn't really cover this use case; it is more for tracing and is not really an interactive debugger. 

Drgn makes it possible to write actual programs in a real programming language—depending on one's opinion of Python, anyway. It is much better than dumping things out to a text file and using shell scripts to process them or to use the Python bindings for GDB. He sometimes calls drgn a "debugger as a library" because it doesn't just provide a command prompt with a limited set of commands; instead, it magically wraps the types, variables, and such so that you can do anything you want with them. The [User Guide](<https://drgn.readthedocs.io/en/latest/user_guide.html>) and home page linked above are good places to start looking into all that it can do. 

He launched into another demo that showed some of the power of drgn. It has both interactive and scripting modes. He started in an interactive session by looking at variables and noted that drgn returns an object that represents the variable; that object has additional information like the type (which is also an object), address, and, of course, value. But one can also implement list iteration, which he showed by following the `struct task_struct` chain from the `init` task down to its children. 

While he had written the list iteration live in the demo, he pointed out that it would get tedious if you had to do so all of the time. Drgn provides a bunch of [helper functions](<https://drgn.readthedocs.io/en/latest/helpers.html>) that can do those kinds of things. Currently, most of those are filesystem and block-layer helpers, but more could be added for networking and other subsystems. 

He replayed an actual investigation that he and a colleague had done on a production server in a VM where the bug was reproduced. The production workload was a storage server for cold data; on it, disks that have not been used in a while are powered down to save power. So its disks tend to turn on and off a lot, which exposes kernel bugs. The cold-storage service ran in a container and it was reported that stopping the container would sometimes take forever. 

When he started looking at it, he realized that the container would eventually finish, but that it took a long time. That suggested some kind of a leak. He showed the process of working his way down through the block control group data structures and used the Python Set object type to track the number of unique request queues associated with the block control groups. He was also able to dig around in the radix tree associated with the [ID allocator](<https://www.kernel.org/doc/html/v4.18/core-api/idr.html#ida-usage>) (IDA) used for identifying request queues to double check some of his results. In the end, it was determined that the request queues were leaking due to a reference cycle. 

He mentioned another case where he used drgn to debug a problem with Btrfs unexpectedly returning `ENOSPC`. It turned out that it was reserving extra metadata space for orphaned files. Once he determined that, it was straightforward to figure out which application was creating these orphaned files; it could be restarted periodically until a real fix could be made to Btrfs. In addition, when he encounters a new subsystem in the kernel, he will often go in with drgn to figure out how all of the pieces fit together. 

The core of drgn is a C library called libdrgn. If you hate Python and like error handling, you can use it directly, he said. There are pluggable backends for reading memory of various sorts, including `/proc/kcore` for the running kernel, a crash dump, or `/proc/_PID_ /mem` for a running program. It uses [DWARF](<http://dwarfstd.org/>) to get the types and symbols, which is not the most convenient format to work with. He spent a surprising amount of time optimizing the access to the DWARF data. That interface is also pluggable, but he has only implemented DWARF so far. 

That optimization work allows drgn to come up in about half a second, while crash takes around 15s. Because drgn comes up quickly, it will get used more; he still dreads having to start up crash. 

There is a subset of a C interpreter embedded into drgn. That allows drgn to properly handle a bunch of corner cases, such as implicit conversions and integer promotion. It is prickly and took some effort, but it means that he has not run into any cases where the translated code does not work the way it does in the kernel. 

The biggest missing feature is backtrace support, he said. You can only access global variables at this point, which is not a huge limitation, but he does sometimes have to use crash to get addresses and other information to plug into drgn. It is something that is "totally possible to do in drgn", but he has not gotten there yet. He would like to use [BPF Type Format](<https://www.kernel.org/doc/html/latest/bpf/btf.html>) (BTF) instead of DWARF because it is much smaller and simpler. But the main limitation is that BTF does not handle variables; if and when it does, he will use it. A repository of useful drgn scripts and tools is in the works as well. 

Integration with BPF and BCC is something that has been nagging at him. The idea would be to use BPF for live debugging and drgn for after-the-fact debugging in some way. There is some overlap between the two, which he has not quite figured out how to unify. BPF is somewhat painful to work with due to its lack of loops, but drgn cannot really catch things as they happen. He has a "crazy insane idea" to have BPF breakpoints that call out to a user-space drgn program, but he is not at all sure it is possible. 

That was the last session I was able to sit in on and this article completes LWN's LSFMM coverage. The talk on drgn made a nice segue for me, as I had to leave to catch a plane to (eventually) end up in Cleveland for [PyCon](<https://us.pycon.org/2019/>).
