---
title: Injecting faults into the kernel
url: https://lwn.net/Articles/209257/
date: "November 14, 2006"
category: "Development tools-Kernel debugging; Fault injection"
author: 
---

Some kernel developers, doubtless, feel that their systems fail too often as it is; they certainly would not go out looking for ways to make more trouble. Others, however, are most interested in how their code behaves when things go wrong. As your editor recently [discovered](<http://permalink.gmane.org/gmane.linux.kernel/465281>) to his chagrin, error paths tend to be debugged rather less well than the "normal" code. One can try to anticipate possible failures and try to code the right response, but it can be hard to actually test that code. So error-handling paths can be incorrect (or missing) but the code will appear to work - until something blows up. 

In an attempt to help test kernel error handling, Akinobu Mita has been working for some time on a framework for injecting faults into a running kernel. By causing things to go wrong occasionally, the fault injection code should help to ensure that error situations are handled - and handled correctly. This mechanism has found its way into [2.6.19-rc5-mm2](<http://lwn.net/Articles/209232/>) where, hopefully, it will be employed by developers to make sure that their code is bulletproof. Hopefully. 

The framework can cause memory allocation failures at two levels: in the slab allocator (where it affects `kmalloc()` and most other small-object allocations) and at the page allocator level (where it affects everything, eventually). There are also hooks to cause occasional disk I/O operations to fail, which should be useful for filesystem developers. In both cases, there is a flexible runtime configuration infrastructure, based on debugfs, which will let developers focus fault injections into a specific part of the kernel. 

Your editor built a version of 2.6.19-rc5-mm2 with the fault injection capability turned on. For whatever reason, the configuration system insisted that the locking validator be enabled too; perhaps somebody injected a fault into the config scripts. In any case, the resulting kernel exports a directory (in debugfs) for each of the available fault injection capabilities. 

So, for example, the slab allocation capability has a directory `failslab`. At system boot, failure injection is turned off; slab failures can be enabled by writing an integer value to the `failslab/probability` file. The value written there will be interpreted as the percent probability that any given allocation will fail; so writing "`5`" will cause a 5% failure rate. For situations where a failure rate of less than 1% (but greater than zero) is needed, there is a separate `interval` value which further filters the result. So a 0.1% failure rate could be had by setting `interval` to 1000 and `probability` to 100 - preferably in that order. There is also a `times` variable which puts an upper limit on the number of failures which will be simulated. 

As it happens, randomly injecting failures into the kernel as a whole does not necessarily lead to a lot of useful information for a developer, who is probably interested in the behavior of a specific subsystem. There is only so long that one can put up with basic shell commands failing while trying to make something happen in one particular driver. So there are a number of options which can be used to focus the faults on a particular part of the kernel. These include: 

  * `task-filter`: if this variable is set to a positive value, faults will only be injected when a specially-marked processes are running. To enable this marking, each process has a new flag (`make-it-fail`) in its `/proc` directory; setting that value to one will cause faults to be injected into that process. 

  * `address-start` and `address-stop`: if these values are set, fault injection will be concentrated on the code found within the address range specified. As long as any entry within the call chain is inside that address range, the fault injection code will consider causing a failure. 

  * `ignore-gfp-wait`: if this value is set to one, only non-waiting (`GFP_ATOMIC`) allocations will potentially fail. There is also a `ignore-gfp-highmem` option which will cause failures not to be injected into high-memory allocations. 

Various other options exist; there is also a set of boot options for turning on injection which might be useful for debugging early system initialization. The [documentation file](<https://lwn.net/Articles/209292/>) has the details. Also found in the documentation directory are a couple of scripts for concentrating faults on a specific command or module. 

The end result of all this is a useful tool. One need not just hope that the error recovery paths in a piece of kernel code will just work properly; it is now possible to actually run them and see what happens. This should lead to a better tested, more robust kernel in the near future, and that can only be a good thing.
