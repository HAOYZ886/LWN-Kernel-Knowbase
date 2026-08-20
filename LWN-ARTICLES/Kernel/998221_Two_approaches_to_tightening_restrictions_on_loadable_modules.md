---
title: Two approaches to tightening restrictions on loadable modules
url: https://lwn.net/Articles/998221/
date: "November 15, 2024"
category: "Copyright issues; Development model-Loadable modules; Modules-Exported symbols"
author: "By Jonathan Corbet November 15, 2024"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
November 15, 2024

The kernel's loadable-module facility allows code to be loaded into (and sometimes removed from) a running kernel. Among other things, loadable modules make it possible to run a kernel with only the subsystems needed for the system's hardware and workload. Loadable modules can also make it easy for out-of-tree code to access parts of the kernel that developers would prefer to keep private; this has led to [many discussions](<https://lwn.net/Kernel/Index/#Modules-Exported_symbols>) in the past. The topic has returned to the kernel's mailing lists with two different patch sets aimed at further tightening the restrictions applied to loadable modules. 

When the static kernel image is linked, references to symbols (the names of functions and data structures) are resolved using the entire global namespace. Loading a module also involves a linking step, but modules do not have access to the full namespace; instead, they can only access symbols that have been explicitly exported to them. There are two sets of exported symbols: those that are available to any loadable module, and those that are only available to modules that declare a GPL-compatible license. Access to symbols is the primary means by which the capabilities of loadable modules are limited, so it is not surprising that both patch sets make changes to that mechanism. 

#### Restricted namespaces

For most of the kernel's existence, there has been a single namespace to hold all of the symbols available to a loadable module; that namespace only contains the GPL-restricted symbols if the module is appropriately licensed. In 2018, the kernel gained [a symbol namespacing capability](<https://lwn.net/Articles/760045/>) that can segregate some symbols and restrict their availability to modules that explicitly import the relevant namespace. This feature was meant to (among other things) make abuses (modules accessing symbols that they should not) more evident, but it has no access-control capability; symbols can still be made available just by importing the namespace that contains them. 

There has long been a wish, though, for the ability to export symbols for use by a specific module, but no others. [This patch](<https://lwn.net/ml/all/20241106190240.GR10375@noisy.programming.kicks-ass.net>) from Peter Zijlstra adds that feature. In current kernels, a symbol is exported into a specific namespace (call it `foo`) with a declaration like: 

```
EXPORT_SYMBOL_NS_GPL(symbol_name, foo);
```

Any module that contains a line like: 

```
MODULE_IMPORT_NS(foo);
```

can then access the symbols exported into that namespace. Zijlstra's patch adds a tweak to the export declaration. To export a symbol that is _only_ available within the module called `foo`, the declaration would be: 

```
EXPORT_SYMBOL_NS_GPL(symbol_name, MODULE_foo);
```

This creates a namespace with a couple of special properties. When a module named `foo` is loaded, this namespace will be implicitly imported; there is no need for a `MODULE_IMPORT_NS()` declaration. And, in fact, any attempt to import a namespace whose name starts with `MODULE_` will be blocked. The end result is that the symbol is available to `foo`, but to no other module. 

In the discussion, nobody argued against the addition of this capability. There were a few thoughts on the syntax. Luis Chamberlain, the module-loader maintainer, [suggested](<https://lwn.net/ml/all/Zyv-yxClglfwvmUa@bombadil.infradead.org/>) that a separate `EXPORT_SYMBOL_GPL_FOR()` syntax might be preferable to the `MODULE_` convention; he also said that it would be useful to be able to export symbols to more than one module. 

Masahiro Yamada, the maintainer of the kernel's build system, [said](<https://lwn.net/ml/all/CAK7LNASDnB86Ds_dqBTxfHi=OfTXqu66U3v+4M_OEzKiK5Skdw@mail.gmail.com>) that it would be better for the namespace name to be a C string rather than a bare name. That would eliminate some ambiguities within the kernel code; it would also be possible for that string to be a comma-separated list of target modules. That would be a big change, as was demonstrated when Zijlstra [took a stab at it](<https://lwn.net/ml/all/20241107125044.GB38786@noisy.programming.kicks-ass.net>): the resulting patch touched 847 files. 

It seems likely that the quoted-string approach will be favored going forward, though. Zijlstra has put together [a version of the patch](<https://lwn.net/ml/all/20241107152749.GA38972@noisy.programming.kicks-ass.net/>) that supports exporting to multiple modules using that syntax. It ""seems to work with very limited testing"", but has not yet been reposted to the list. The posting [can be expected](<https://lwn.net/ml/all/20241115124935.GC22801@noisy.programming.kicks-ass.net>) soon if all goes well, but chances are that this work is a bit too late to make it into the 6.13 kernel release. 

#### When "GPL" is not GPL

Meanwhile, a separate patch is taking a rather different approach to the problem of inappropriate access to symbols by loadable modules. The kernel is licensed under version 2 of the GNU General Public License, and no other. When the Free Software Foundation created version 3 of the GPL, it was made incompatible with version 2; the kernel community declined to switch to the new license, and so cannot accept code that is licensed under GPLv3. So one would not normally expect to see device drivers (or other kernel modules) released under that license. 

It turns out, though, that [Tuxedo Computers](<https://www.tuxedocomputers.com/en>) maintains [a set of device drivers](<https://gitlab.com/tuxedocomputers/development/packages>) for its hardware, and those drivers are indeed licensed under GPLv3. In the `MODULE_LICENSE()` declaration within those modules, though, the license is claimed to be "GPL". As a result, these modules have access to GPL-only kernel exports, even though they do not have a license that is compatible with the kernel's. 

This situation has been in the open for some time, but it was only brought to the foreground after [this research](<https://fosstodon.org/@kernellogger/113423314337991594>) from Thorsten Leemhuis pulled it all together. Neal Gompa [pointed it out](<https://github.com/tuxedocomputers/tuxedo-keyboard/issues/61>) in 2020 and asked for a relicensing to GPLv2. The discussion has resurfaced a few times since then, but the company has refused to make that change. Earlier this year, Tuxedo's Werner Sembach [made the company's position clear](<https://gitlab.com/tuxedocomputers/development/packages/tuxedo-drivers/-/issues/137#note_1807179414>): ""We do not plan to relicense the tuxedo-drivers project directly as we want to keep control of the upstream pacing"". In other words, the incompatible license is a deliberate choice made by the company to keep its drivers out of the mainline until a time of its own choosing. 

The licensing decision may be a bit strange, but it is certainly within the company's rights. Declaring a compatible license to gain access to restricted symbols is not, though. In response, Uwe Kleine-König has posted [a patch series](<https://lwn.net/ml/all/20241114103133.547032-4-ukleinek@kernel.org/>) that explicitly blocks the Tuxedo drivers from accessing GPL-only symbols. With this patch in place, those drivers will no longer load properly into the kernel and will stop working. 

The response to the patch has been generally (if [not exclusively](<https://lwn.net/ml/all/20241114131843.0df6a5a2@kf-ir16/>)) positive. But Sembach, unsurprisingly, [is not a fan](<https://lwn.net/ml/all/e32e9f5c-3841-41f7-9728-f998f123cc8a@tuxedocomputers.com/>). According to him, the situation is the result of understandable confusion: ""We ended up in this situation as MODULE_LICENSE("GPL") on its own does not hint at GPL v2, if one is not aware of the license definition table in the documentation"". The licensing situation is being worked on, he said, and will eventually be resolved. 

If the company truly intends to work things out in good faith, it would almost certainly make sense to hold off on explicitly blocking its modules while that work proceeds. Given how long this problem has been known, though, and given the company's deliberate use of license incompatibility to retain control over its code, convincing the development community of its good faith may be difficult. That hasn't kept Sembach from trying; he [has relicensed](<https://lwn.net/ml/all/20241115130139.1244786-1-wse@tuxedocomputers.com>) some of the modules in question, and promises to change the rest as soon as possible. 

That is a step in the right direction, but there is no fury that compares to that of a kernel developer who feels lied to about module licensing. Kleine-König has [indicated](<https://lwn.net/ml/all/h5q36ajuzgwf5yrjmqv46x62evifcgoi5imxhcvsv7oxauvxak@sj54oisawqnf>) his intent to try to merge the patch during the 6.13 merge window. Then, he said, if the licensing issue is fully resolved, ""you have my support to revert the patch under discussion"". Whether things will truly go that far is unclear; if Tuxedo is working to resolve the problem quickly, there will probably be little appetite for merging a patch punishing the company in the meantime. It seems unlikely, though, that Tuxedo will attempt this particular trick again, and any others considering it have reason to think again.
