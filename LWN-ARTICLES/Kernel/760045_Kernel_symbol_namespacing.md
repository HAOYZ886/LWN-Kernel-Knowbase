---
title: Kernel symbol namespacing
url: https://lwn.net/Articles/760045/
date: "July 18, 2018"
category: "Modules-Exported symbols"
author: "By Jonathan Corbet July 18, 2018"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
July 18, 2018

In order to actually do anything, a kernel module must gain access to functions and data structures in the rest of the kernel. Enabling and controlling that access is the job of the symbol-export mechanism. While the enabling certainly happens, the control part is not quite so clear; many developers view the nearly 30,000 symbols in current kernels that are available to all modules as being far too many. The [symbol namespaces patch set](<https://lwn.net/ml/linux-kernel/20180716122125.175792-1-maco@android.com/>) from Martijn Coenen doesn't reduce that number, but it does provide a mechanism that might help to impose some order on exported symbols in general. 

Kernel code can make a symbol (a function or a data structure) available to loadable modules with the `EXPORT_SYMBOL()` and `EXPORT_SYMBOL_GPL()` macros; the latter only makes the symbol available to modules that have declared a GPL-compatible license. There is also `EXPORT_SYMBOL_GPL_FUTURE()`, which is meant to mark symbols that will be changed to a GPL-only export at some future time. The usage of this mechanism is also a matter for the future, though; it has not been employed since just after it was [introduced](<https://lwn.net/Articles/171838/>) in 2006\. On the rare occasions when symbols have been changed to GPL-only exports, it has proved easier to just change them without putting advance notice in the code. 

`EXPORT_SYMBOL()` works by declaring a `kernel_symbol` structure: 

```
struct kernel_symbol
        {
    	unsigned long value;
    	const char *name;
        };
```

After the link phase, this structure holds a pointer to the name of the symbol and the address corresponding to that symbol. The structures corresponding to all exported symbols are gathered together by the linker into two ELF sections in the kernel (or module) binary: `__ksymtab` and `__ksymtab_gpl`. There is no particular ordering of, or separation between, these symbols in either section; they all appear in one big pile. 

Not all exported symbols are alike, though. While most of them exist because loadable modules need them to get their job done, that is not universally the case. Some may be exported as a convenient way of debugging kernel code. Others are part of a large subsystem that consists of multiple modules, and should only be used within that particular subsystem. There is no way, beyond code comments, to mark symbols like these. 

Coenen's patch set seeks to address this problem by adding a simple namespace concept to exported symbols. While the default behavior will continue to be to put symbols into the unnamed global namespace, the possibility will exist to segregate symbols to a separate space where an explicit effort will be required to use them. There are two new macros for exporting symbols: 

```
EXPORT_SYMBOL_NS(symbol, namespace);
        EXPORT_SYMBOL_NS_GPL(symbol, namespace);
```

One might expect these new macros to create new sections for the namespaced symbols, but that's not what was done. Instead, the name of the namespace is appended to the symbol name and the result is placed in the same `__ksymtab` (or `__ksymtab_gpl`) section as before. So if `kmalloc()` were to be exported in a new `MM` namespace, it would appear in the resulting binary as `kmalloc.MM`. (Note that, in reality, a core symbol like `kmalloc()` probably would not be segregated in this way.) 

To use symbols from a specific namespace, a module would declare its access to that namespace with: 

```
MODULE_IMPORT_NS(namespace);
```

This mechanism _does_ use a new ELF section ("`__knsimport`") to hold a list of the namespaces that a given module has imported. Listing the imported namespaces is essentially all it does; the mechanism doesn't go much deeper than noting that a module wants access to a particular namespace. 

The actual enforcement of the namespace mechanism can be described as "light handed". There are no indications at compile time that a namespaced symbol is being used; in the fictional example from above, code could call `kmalloc()` without having imported the `MM` namespace, and the compiler would do nothing differently. Things do change in the post-compilation `modpost` step, where a warning will be issued for the use of symbols from a namespace that has not been imported. Another warning will happen when the module is loaded: the kernel will notice the use of a symbol without a declaration to import its containing namespace, but nothing will prevent the actual use of this symbol. 

The patch set only creates one namespace: `USB_STORAGE` for a set of USB symbols. It includes a mechanism to automatically create a patch for other subsystems, a feature that Greg Kroah-Hartman [described](<https://lwn.net/ml/linux-kernel/20180716153356.GA8034@kroah.com/>) as ""frickin amazing"". Overall, it's a small start for a mechanism that may someday help the kernel community get a handle on its huge pile of unsorted symbols, but the kernel itself started small as well. If it proves useful, it will grow over time and, perhaps, bring some order to a notoriously undisciplined part of the kernel.
