---
title: EXPORT_SYMBOL_GPL_FUTURE()
url: https://lwn.net/Articles/171838/
date: "February 13, 2006"
category: "Copyright issues; EXPORT SYMBOL GPL; Modules-Exported symbols; Read-copy-update"
author: 
---

The kernel has a couple of macros for making internal symbols available to loadable modules: 

```
EXPORT_SYMBOL(symbol);
        EXPORT_SYMBOL_GPL(symbol);
```

The second form is used for kernel symbols which are only available to modules with a GPL-compatible license. The idea behind GPL-only symbols is that they are so deeply internal to the kernel that any module using them can only be a derived product of the kernel. Either that, or it's a relatively new symbol whose creator simply wanted it to be GPL-only. 

Greg Kroah-Hartman has recently [proposed](<http://lwn.net/Articles/171129/>) a new variant: 

```
EXPORT_SYMBOL_GPL_FUTURE(symbol)
```

Its purpose would be to mark symbols which will be changed to a GPL-only export at some point in the future. If such a symbol is used by a non-GPL module, the kernel will emit warnings to the effect that the module will break at a future time. With luck, the warnings will help authors of proprietary modules prepare for changes ahead of time. 

This patch raised a few eyebrows. When GPL-only exports were first added to the kernel, they went in with the understanding that only new symbols would be tagged GPL-only. The current module interface - while always subject to change - was not to have symbols withdrawn arbitrarily. So, if the export status of symbols should not change, what is the use of this patch? Greg has a couple of uses in mind: 

  * The read-copy-update symbols are due to turn GPL-only in April of this year. The use of RCU by non-GPL modules has always been legally problematic: RCU is a patented technique which has been licensed by IBM for use in GPL code. Non-GPL modules will (in the absence of other arrangements) lack a license for RCU, and thus should not be using those symbols anyway. 

  * Current plans call for making the core USB subsystem GPL-only in early 2008\. The argument here is that this subsystem has changed greatly over time, and that it is possible to create full-speed USB drivers in user space. 

It is not clear that there will be uses beyond these; resistance to a larger-scale restricting of exported symbols remains strong. So the weapon of choice for those who wish to make life difficult for proprietary modules is likely to remain the combination of API changes and restrictions on new symbols.
