---
title: Tightening symbol exports
url: https://lwn.net/Articles/260041/
date: "November 27, 2007"
category: "Development model-Loadable modules; Modules-Exported symbols"
author: "By Jonathan Corbet November 27, 2007"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
November 27, 2007

The kernel's loadable module mechanism does not give modules access to all parts of the kernel. Instead, any kernel symbol which is intended to be usable by loadable modules must be explicitly exported to them via one of the variants of the `EXPORT_SYMBOL()` macro. The idea behind this restriction is to place limits on the reach of modules and to provide a relatively well-defined module API. In practice, there have been few limits placed on the exporting of symbols, with the result that many thousands of symbols are available to modules. Loadable modules can access many of the obviously useful symbols (`printk()`, say, or `kmalloc()`), but they can also get at generic symbols like `edd`, `tpm_pm_suspend()`, `vr41xx_set_irq_trigger()`, or `flexcop_dump_reg()`. 

There are reasons for the concern over excessive symbol exports felt by some developers. Wrongly exported symbols can lead module authors to use incorrect interfaces; for example, the exporting of `sys_open()` is an active inducement for developers to open files directly inside the kernel, which is almost never a good idea. But such symbols, once exported, [can prove hard to unexport](<http://lwn.net/Articles/249246/>). While the official line says that the internal kernel API can change at any time, the truth of the matter is that at least some developers are reluctant to break external modules when that can be avoided. 

A more timely example would be `init_level4_pgt`, a low-level symbol exported only by the x86_64 architecture. The current -mm tree removes that export, breaking the proprietary NVIDIA module in the process. Andrew Morton [describes](<https://lwn.net/Articles/260045/>) this removal as ""our clever way of reducing the tester base so we don't get so many bug reports."" While many developers make a show of not caring about binary-only modules, there is still a good chance that this particular export removal (of a symbol which should not really be available globally) may not make it into the mainline as a result of this breakage. 

The end result of all this is that there has long been interest in somehow cleaning up the modular API, though there have not been a whole lot of people who have put a lot of time toward that end. Occasionally somebody has remarked upon one piece of low-hanging fruit: symbols which are exported only to make it possible to modularize other bits of mainline kernel code. One example is a whole set of TCP stack symbols (things like `__tcp_put_md5sig_pool()`) which have exactly one user: the IPv6 module. Restricting these special-purpose exports has the potential to significantly narrow the modular API without making it harder to modularize the mainline. 

Andi Kleen's [module symbol namespace patch](<http://lwn.net/Articles/259839/>) is meant to enable just this sort of narrowing of the API. With this patch, symbols can be exported into specific "namespaces" which are only available to modules appearing on an associated whitelist. In a sense, the term "namespace" is a poor fit here; there is still a single, global namespace within which all exported symbols must be unique. These "namespaces" are more like special exclusion zones containing symbols which are not globally accessible. They work like GPL-only exports, which also restrict the availability of symbols to a subset of modules. 

To create a restricted export, an ordinary `EXPORT_SYMBOL()` declaration is changed to: 

```
EXPORT_SYMBOL_NS(namespace, symbol);
```

Where `namespace` is the name of a restricted symbol namespace. So, going back to the TCP example, Andi's patch contains a number of changes like: 

```
-EXPORT_SYMBOL(__tcp_put_md5sig_pool);
        +EXPORT_SYMBOL_NS(tcp, __tcp_put_md5sig_pool);
```

Note that there is no `_GPL` version; any symbol which is exported into a specific namespace is treated as GPL-only by default. 

The other part of the equation is to enable access to a namespace. That is done with: 

```
MODULE_NAMESPACE_ALLOW(namespace, module);
```

Such a declaration (which must appear in a module exporting symbols into the `namespace`) says that the given `module` can access symbols in that namespace. Andi's patch creates three namespaces (`tcp`, `tcpcong` for congestion control modules, and `udp`), removing about 30 symbols from the global namespace. 

A number of developers welcomed this patch, seeing it as a step forward in the rationalization of the loadable module API. It is seen as a way to prevent out-of-tree modules from using symbols which they should not be using. It also reduces the number of interfaces which must be kept stable in situations (enterprise kernels, for example) where changes are not allowed. And, finally, the symbol namespaces offer the ability to organize exports somewhat and document who the intended users are. 

There is a bit of dissent, though. In particular, Rusty Russell fears that the patch adds unneeded complexity and threatens to make life harder for out-of-tree developers for little (if any) gain. [Says Rusty](<https://lwn.net/Articles/260059/>): 

For example, you put all the udp functions in the "udp" namespace. But what have we gained? What has become easier to maintain? All those function start with "udp_": are people having trouble telling what they're for? 

If you really want to reduce "public interfaces" then it's much simpler to mark explicitly what out-of-tree modules can use. 

Herbert Xu [has similar concerns](<https://lwn.net/Articles/260061/>): 

These symbols are exported because they're needed by protocols. If they weren't available to everyone then it would be difficult to start writing new protocols.... 

So based on the network code at least I'm kind of starting to agree with Rusty now: if a symbol is needed by more than one in-tree module chances are we want it to be exported for all. 

While these voices seem to be in the minority, they still carry quite a bit of weight. So your editor is unwilling to make any sort of guess as to whether this patch will be merged, or in what form. The desire to clean up the modular API is unlikely to go away, though, so, sooner or later, something is likely to happen.
