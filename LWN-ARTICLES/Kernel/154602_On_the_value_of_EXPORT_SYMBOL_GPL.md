---
title: On the value of EXPORT_SYMBOL_GPL
url: https://lwn.net/Articles/154602/
date: "October 5, 2005"
category: "Copyright issues; EXPORT SYMBOL GPL; Modules-Licensing"
author: 
---

When a loadable module is inserted, any references it makes to kernel functions and data structures must be linked to the current running kernel. The module loader does not provide access to all kernel symbols, however; only those which have been explicitly exported are available. The export requirement narrows the API seen by modules, though not by all that much: there are over 6,000 symbols exported in the 2.6.13 kernel. 

Exports come in two flavors: vanilla (`EXPORT_SYMBOL`) and GPL-only (`EXPORT_SYMBOL_GPL`). The former are available to any kernel module, while the latter cannot be used by any modules which do not carry a GPL-compatible license. The module loader will enforce this distinction by denying access to GPL-only symbols if the module's declared license does not pass muster. Currently, less that 10% of the kernel's symbols are GPL-only, but the number of GPL-only symbols is growing. There is a certain amount of pressure to make new exports GPL-only in many cases. 

It has often been argued that there is no practical difference between the two types of exports. Those who believe that all kernel modules are required by the kernel license to be GPL-licensed see all symbols as being implicitly GPL-only in any case. Another camp, which sees the module interface as a boundary which the GPL cannot cross, does not believe that GPL-only restrictions can be made to stick. In any case, GPL-only symbols can be easily circumvented by patching the kernel, falsely declaring a GPL-compatible license, or by inserting a shim module which provides wider access to the symbols of interest. 

Linus, however, [believes that GPL-only exports are significant](<https://lwn.net/Articles/154603/>). 

I've talked to a lawyer or two, and (a) there's an absolutely _huge_ difference and (b) they liked it. 

The fact is, the law isn't a blind and mindless computer that takes what you say literally. Intent matters a LOT. And using the xxx_GPL() version to show that it's an internal interface is very meaningful indeed. 

One of the lawyers said that it was a much better approach than trying to make the license explain all the details - codifying the intention in the code itself is not only more flexible, but a lot less likely to be misunderstood. 

He also points out that circumventing a GPL-only export requires an explicit action, making it clear that the resulting copyright infringement was a deliberate act. 

Regardless of any legal significance they may have, the GPL-only exports do succeed in communicating the will of the large subset of the kernel development community which wants to restrict the use of non-free kernel modules. The outright banning of such modules may not be on the agenda anytime soon, but the functionality available to them is not likely to grow much.
