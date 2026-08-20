---
title: "Return values, warnings, and error situations"
url: https://lwn.net/Articles/204682/
date: "October 17, 2006"
category: "  must check; Development model-Kernel quality"
author: 
---

The function `pci_set_mwi()` enables the "memory write and invalidate" (MWI) mode on the PCI bus. If the device on the other end can work with MWI, a small optimization results. The MWI mode might not be enabled, however, even if a device driver requests it; the bus hardware itself might not support it. A failure to set MWI is not generally a problem; things just go a bit slower than they would have otherwise. The calling driver might still want to know if the call succeeded, however, so Matthew Wilcox recently [fixed the function](<https://lwn.net/Articles/204683/>) to return `-EINVAL` if the attempt fails. 

It turns out that this is one of the many patches which have recently sabotaged Andrew Morton's heavily abused Vaio laptop. Some code was checking the result of `pci_set_mwi()`; once that function actually returned the result of the operation, the calling code failed on an error path. But, as noted above, a failure to set MWI is almost never a fatal problem. So, in response to this series of events, Alan Cox [asserted](<https://lwn.net/Articles/204686/>): 

The underlying bug is that someone marked pci_set_mwi must-check, that's wrong for most of the drivers that use it. If you remove the must check annotation from it then the problem and a thousand other spurious warnings go away. 

One suspects Alan is also behind code like the following, from `drivers/ata/pata_cs5530.c`: 

```
compiler_warning_pointless_fix = pci_set_mwi(cs5530_0);
```

The `__must_check` annotation makes use of the gcc `warn_unused_result` attribute; it first found its way into the mainline in 2.6.8. If a function is marked `__must_check`, the compiler will issue a strong warning whenever the function is called and its return code is unused. 

The use of `__must_check` is another step in the long path toward automatic detection of potential bugs. It is intended for functions whose return value really does require checking - `copy_from_user()` is a good example. If that function fails, and the calling code does not notice, it will proceed using essentially random data. Similar issues come up in user space; witness the recent vulnerabilities resulting from privileged applications which fail to check the result of a `setuid()` call. In some cases, there clearly is no excuse for not looking at the return value, and `__must_check` is a good way to find incorrect function usage before it creates real problems. 

In current kernels, however, the list of `__must_check` functions has grown rather long: it includes most of the sysfs, PCI, kobject, and driver core APIs. In some cases, as with `pci_set_mwi()`, it now includes functions whose return values are often of no interest to the calling code. The result, in this case, is snide workarounds in the code, added warning noise, and an actual bug where code which need not fail does so in response to an error return code. 

Still, [according to Andrew Morton](<https://lwn.net/Articles/204689/>), it is a mistake to ignore an error return from a function like `pci_set_mwi()`: 

You, the driver author _do not know_ what pci_set_mwi() does at present, on all platforms, nor do you know what it does in the future. For you the driver author to make assumptions about what's happening inside pci_set_mwi() is a layering violation. Maybe the bridge got hot-unplugged. Maybe the attempt to set MWI caused some synchronous PCI error. For example, take a look at the various implementations of pci_ops.read() around the place - various of them can fail for various reasons. 

This discussion led, eventually, to what might be the real issue: how should in-kernel APIs be designed to properly return status information? A suggestion which has been made is that `pci_set_mwi()` should return zero or one, depending on whether MWI is a possible operating mode. Only if something goes drastically wrong on the PCI bus should a negative error code be returned. No such patch has yet been merged, but that seems like the way this particular issue is likely to be resolved. 

The larger discussion of how errors should be handled may just be beginning, however. There are a number of de-facto conventions for kernel APIs which have evolved over time, but no overall policy on error handling. So Andrew [would like to talk](<https://lwn.net/Articles/204691/>) about guidelines on how different kinds of errors should be handled. In particular, he suggests a rule that a negative error code should never be ignored in any situation. Cases where this kind of result is not relevant (`pci_set_mwi()` being an example) are an indication of an API in need of a redesign. 

So over time, it would not be surprising to see a number of kernel interfaces shift such that a number of error conditions are handled further down the call chain and with the goal of not returning error codes for non-error situations. There is also likely to be a continued effort to cut down on the warning noise, which, at times, threatens to drown out the real errors. With luck, all of this work will lead to safer interfaces and a more robust kernel in the future.
