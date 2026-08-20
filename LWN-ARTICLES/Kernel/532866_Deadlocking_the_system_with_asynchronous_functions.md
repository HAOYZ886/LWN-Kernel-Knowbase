---
title: Deadlocking the system with asynchronous functions
url: https://lwn.net/Articles/532866/
date: "January 16, 2013"
category: Asynchronous function calls
author: "By Jonathan Corbet January 16, 2013"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
January 16, 2013

Deadlocks in the kernel are a relatively rare occurrence in recent years. The credit largely belongs to the "[lockdep](<https://lwn.net/Articles/185666/>)" subsystem, which watches locking activity and points out patterns that could lead to deadlocks when the timing goes wrong. But locking is not the source of all deadlock problems, as was recently shown by an old deadlock bug which was only recently found and fixed. 

In early January, Alex Riesen [reported](<https://lwn.net/Articles/532868/>) some difficulties with USB devices on recent kernels; among other things, it was easy to simply lock up the system altogether. A fair amount of discussion followed before Ming Lei [identified](<https://lwn.net/Articles/532869/>) the problem. It comes down to the block layer's use of the [asynchronous function call infrastructure](<https://lwn.net/Articles/314808/>) used to increase parallelism in the kernel. 

The asynchronous code is relatively simple in concept: a function that is to be run asynchronously can be called via `async_schedule()`; it will then run in its own thread at some future time. There are various ways of waiting until asynchronously called functions have completed; the most thorough is `async_synchronize_full()`, which waits until all outstanding asynchronous function calls anywhere in the kernel have completed. There are ways of waiting for specific functions to complete, but, if the caller does not know how many asynchronous function calls may be outstanding, `async_synchronize_full()` is the only way to be sure that they are all done. 

The block layer in the kernel makes use of I/O schedulers to organize and optimize I/O operations. There are several I/O schedulers available; they can be switched at run time and can be loaded as modules. When the block layer finds that it needs an I/O scheduler that is not currently present in the system, it will call `request_module()` to ask user space to load it. The module loader, in turn, will call `async_synchronize_full()` at the end of the loading process; it needs to ensure that any asynchronous functions called by the newly loaded module have completed so that the module will be fully ready by the time control returns to user space. 

So far so good, but there is a catch. When a new block device is discovered, the block layer will do its initial work (partition probing and such) in an asynchronous function of its own. That work requires performing I/O to the device; that in turn, requires an I/O scheduler. So the block layer may well call `request_module()` from code that is already running as an asynchronous function. And that is where things turn bad. 

The problem is that the (asynchronous) block code must wait for `request_module()` to complete before it can continue with its work. As described above, the module loading process involves a call to `async_synchronize_full()`. That call will wait for _all_ asynchronous functions, including the one that called `request_module()` in the first place, and which is still waiting for `request_module()` to complete. Expressed more concisely, the sequence looks like this: 

  1. `sd_probe()` calls `async_schedule()` to scan a device asynchronously. 

  2. The scanning process tries to read data from the device. 

  3. The block layer realizes it needs an I/O scheduler, so, in `elevator_get()`, it calls `request_module()` to load the relevant kernel module. 

  4. The module is loaded and initializes itself. 

  5. `do_module_init()` calls `async_synchronize_full()` to wait for any asynchronous functions called by the just-loaded module. 

  6. `async_synchronize_full()` waits for all asynchronous functions, including the one called back in step 1, which is waiting for the `async_synchronize_full()` call to complete. 

That, of course, is a classic deadlock. 

Fixing that deadlock turns out not to be as easy as one would like. Ming suggested that the call to `async_synchronize_full()` in the module loader should just be removed, and that user space should be taught that devices might not be ready immediately when the `modprobe` binary completes. Linus [was not impressed](<https://lwn.net/Articles/532836/>) with this approach, however, and it was quickly discarded. 

The optimal solution would be for the module loader to wait only for asynchronous functions that were called by the loaded module itself. But the kernel does not currently have the infrastructure to allow that to happen; adding it as an urgent bug fix is not really an option. So something else needed to be worked out. To that end, Tejun Heo was brought into the discussion and asked to help come up with a solution. Tejun originally [thought](<https://lwn.net/Articles/532870/>) that the problem could be solved by detecting deadlock situations and proceeding without waiting in that case, but the problem of figuring out when it would be safe to proceed turned out not to be tractable. 

The [solution](<https://lwn.net/Articles/532871/>) that emerged instead is regarded as a bit of a hack by just about everybody involved. Tejun added a new process flag (`PF_USED_ASYNC`) to mark when a process has called asynchronous functions. The module loader then tests this flag; if no asynchronous functions are called as the module is loaded, the call to `async_synchronize_full()` is skipped. Since the I/O scheduler modules make no such calls, that check avoids the deadlock in this particular case. Obviously, the problem remains in any case where an asynchronously-loaded module calls asynchronous functions of its own, but no other such cases have come to light at the moment. So it seems like a workable solution. 

Even so, Tejun remarked ""It makes me feel dirty but makes the problem go away and I can't think of anything better"". The patch has [found its way into the mainline](<http://git.kernel.org/linus/774a1221e862b343388347bac9b318767336b20b>) and will be present in the 3.8 final release. By then, though, it would not be entirely surprising if somebody else were to take up the task of finding a more elegant solution for a future development cycle.
