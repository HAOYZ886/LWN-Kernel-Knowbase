---
title: "Enforcement (or not) for module-specific exported symbols"
url: https://lwn.net/Articles/1029492/
date: "July 15, 2025"
category: "Development model-Loadable modules; Modules-Exported symbols"
author: "By Jonathan Corbet July 15, 2025"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
July 15, 2025

Loadable kernel modules require access to kernel data structures and functions to get their job done; the kernel provides this access by way of exported symbols. Almost since this mechanism was created, there have been debates over which symbols should be exported, and how. The 6.16 kernel gained a new export mechanism that limits access to symbols to specific kernel modules. That code is likely to change soon, but the addition of an enforcement mechanism has since been backed out. 

Restrictions on exported symbols are driven by two core motivations, the first of which is to ensure that kernel modules are truly modular and do not access core parts of the kernel. The intent is limit the amount of damage a module can do, and to keep kernel modules from changing fundamental aspects of the kernel's operation. The other motivation is a desire to make life difficult for proprietary kernel modules by explicitly marking exports that are so fundamental to the kernel's design that any code making use of them must be a derived product of the kernel. Those symbols are unavailable to any module that does not declare a GPL-compatible license. 

There have been many discussions about the proper exporting of symbols over the years; see [the LWN kernel index](<https://lwn.net/Kernel/Index/#Modules-Exported_symbols>) for the history. The most recent example may be [this discussion](<https://lwn.net/ml/all/aHC-_HWR2L5kTYU5@infradead.org>) on whether the in-progress [user-space stack-unwinding improvements](<https://lwn.net/Articles/1029189/>) should be made available to the out-of-tree [LTTng](<https://lttng.org/>) module. These discussions do not appear to have impeded the exporting of symbols in general; there are nearly 38,000 exported symbols in the upcoming 6.16 kernel. 

That kernel release will also include [a new member](<https://docs.kernel.org/next/core-api/symbol-namespaces.html#using-the-export-symbol-gpl-for-modules-macro>) of the `EXPORT_SYMBOL()` macro family: 

```
EXPORT_SYMBOL_GPL_FOR_MODULES(symbol, "mod1,mod2");
```

This macro will cause the given `symbol` to be exported, but only to the modules named in the second argument, and only if those modules are GPL-licensed. It is intended for use with symbols that kernel developers would strongly prefer not to export at all, but which are needed by one or more in-tree subsystems when those subsystems are built as modules. There are no users of this macro in the 6.16 kernel, but a few are waiting in the wings. The symbol exports needed for [the proposed unification of kselftests and KUnit](<https://lwn.net/Articles/1029077/>) are expressed this way, for example. 

The linux-next repository contains a couple of other users that are planned for merging in 6.17; included therein is the exporting of the rigorously undocumented [`anon_inode_make_secure_inode()`](<https://elixir.bootlin.com/linux/v6.15.5/source/fs/anon_inodes.c#L101>) exclusively for the KVM subsystem. That particular export came about as the result of a discussion following the posting, by Shivank Garg, of [this patch](<https://lwn.net/ml/all/20250619073136.506022-2-shivankg@amd.com/>) making that function available to KVM to fix a security issue. Vlastimil Babka [suggested](<https://lwn.net/ml/all/da5316a7-eee3-4c96-83dd-78ae9f3e0117@suse.cz/>) using the new, module-specific mechanism, a suggestion that Garg quickly adopted. 

Christian Brauner, having evidently learned about the new mechanism in this discussion, [said](<https://lwn.net/ml/all/20250623-warmwasser-giftig-ff656fce89ad@brauner/>) that he would happily switch a number of other core filesystem-level exports to the module-specific variety, but there was one problem: some of those exports are not GPL-only, so there would need to be a version of the macro that does not impose the extra license restriction. Christoph Hellwig [disagreed with the need](<https://lwn.net/ml/all/aFleB1PztbWy3GZM@infradead.org/>) for a non-GPL version, though, saying that any export that is limited to specific in-tree modules is, by definition, only exported to GPL-licensed code. He [suggested](<https://lwn.net/ml/all/aFleJN_fE-RbSoFD@infradead.org/>) just removing the `GPL_` portion of the name of the macro, instead. 

Babka [pointed out](<https://lwn.net/ml/all/c0cc4faf-42eb-4c2f-8d25-a2441a36c41b@suse.cz/>) one other potential difficulty, though: while the module-specific exports are intended for in-tree modules only, there is nothing in the kernel that enforces this expectation. So an evil maintainer of a proprietary module could, for example, simply name that module `kvm`, and `anon_inode_make_secure_inode()` would become available to it. To address this perceived problem, Babka posted [a patch series](<https://lwn.net/ml/all/20250708-export_modules-v1-0-fbf7a282d23f@suse.cz>) that added enforcement of the in-tree-only rule, and also changed the name of the macro to `EXPORT_SYMBOL_FOR_MODULES()`. 

It is worth noting that this enforcement implementation, were it to be applied, would still have the potential to be circumvented in external modules. The only way that the kernel knows that a specific module is of the in-tree variety is if the module itself tells it so. Specifically, the kernel looks at the `.modinfo` ELF section in the binary kernel module for a line reading "`intree=Y`"; if that line is found, the module is deemed to be an in-tree module. If an evil developer is willing to impersonate a privileged module to gain access to a specific symbol, they probably will not be daunted by the prospect of adding a false in-tree declaration to the module as well. 

In any case, though, that change ended up being dropped in [the second version of the patch series](<https://lwn.net/ml/all/20250711-export_modules-v2-1-b59b6fad413a@suse.cz>). Masahiro Yamada had [raised a concern](<https://lwn.net/ml/all/CAK7LNATpQrHX_8x4WvhDN7cODCCLr8kihydtfM-6wxhY17xtQw@mail.gmail.com/>) with the new enforcement mechanism: some developers of in-tree modules do their work on out-of-tree versions until the work is merged. In other words, sometimes, replacing an in-tree module with an out-of-tree version using the same name is a legitimate use case. Rather than break a workflow that some developers may depend on, Babka opted to remove the enforcement mechanism entirely, leaving only the name change. 

In this case, as always, the purpose of the removed change was not to create a bulletproof defense against attempts to circumvent the export mechanism. It was, instead, to make the intent of the development community clear, so that anybody engaging in such abuse has no excuses when they are caught. The existing mechanisms in the kernel should be sufficient for that purpose; anybody who is willing to bypass them would probably not have been slowed down much by the addition of another hurdle.
