---
title: Debugging kernel modules
url: https://lwn.net/Articles/90913/
date: "June 23, 2004"
category: "Debugging; Modules-Debugging"
author: 
---

Linus is famously against the use of interactive debuggers on the kernel, but many developers use them anyway. Debugging a running kernel is a little harder than working with a typical application, but it can be done in a couple of ways. It is relatively easy to query kernel data structures in the current running kernel by running `gdb` with `/proc/kcore` as the "core" file. More extensive debugging, allowing the use of breakpoints and such, can be done by using `gdb` on a remote machine and controlling the target via a serial line or a network interface. The -mm tree contains the necessary patches for using gdb in this mode for a few architectures. 

One limitation with using gdb this way is that it can't be used to work with loadable modules. The debugger can query the memory used by loadable modules, set breakpoints there, etc. The problem is that it does not know what addresses get assigned to functions and variables when a module is loaded. Those addresses, obviously, are not in the core kernel executable, and there is no real way to find them at run time. The developer can thus work by typing in hex addresses directly, but that gets tiresome fairly quickly. 

Your editor was recently finishing out the debugging chapter for _Linux Device Drivers, Third Edition_ (which is getting closer to ready - honest) when he ran up against the loadable module problem. The kernel knows where all of the symbols go when it loads a module; it really seemed like it should be possible to communicate that information to a debugger. A bit of digging revealed that, in fact, the relevant information gets dropped once the module gets loaded. So it was time for a fix. 

Like any other ELF executable, a loadable module is divided up into several sections. The section called `.text` contains (most of) the module code itself; `.data` and `.bss` contain most of the variables. The module loader looks at all of the sections and lays them out sequentially in (vmalloc) memory; after relocating symbols it forgets about where the sections went. If the positions of the sections could be recovered, however, they could be passed to `gdb` in the same `add-symbol-file` command which tells the debugger about the module code. The section offsets are all that `gdb` needs to figure out where the module's variables live. 

Your editor, rather than tell LDD3 readers that symbolic debugging of kernel modules was impossible, chose to do a little hacking. The result was [this patch](<https://lwn.net/Articles/88052/>), which hangs a new kobject onto each loadable module and populates it with a set of attributes containing the section offsets. Those attributes will show up under `/sys/module`. Thus, for example, after module `foo` is loaded, `/sys/module/foo/sections/.data` will contain the beginning of the `.data` section. The `foo` developer can then fire up `gdb` and, after connecting to the target kernel, use the section offset information to issue a command like: 

```
add-symbol-file /path/to/module 0xd081d000 \  # .text
     		-s .data 0xd08232c0 \
    		-s .bss  0xd0823e20
```

Thereafter, debugging the module is just like debugging the rest of the kernel. There is a little script (included with the patch) which generates the `add-symbol-file` command, reducing the operation to a simple cut-and-paste. 

The patch has been merged into Linus's BitKeeper tree, and will be part of 2.6.8.
