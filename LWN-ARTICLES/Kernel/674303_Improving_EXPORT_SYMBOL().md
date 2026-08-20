---
title: Improving EXPORT_SYMBOL()
url: https://lwn.net/Articles/674303/
date: "February 3, 2016"
category: "Modules-Exported symbols"
author: "By Jonathan Corbet February 3, 2016"
---

By **Jonathan Corbet**  
February 3, 2016

The kernel's `EXPORT_SYMBOL()` directive is used to make kernel symbols (functions and data structures) available to loadable modules; only the symbols that have been explicitly exported can by used by modules. This directive is a simple macro that has, since the beginning, had a couple of annoying limitations, but nobody has ever gotten around to fixing them until now. Al Viro's [patch set](<https://lwn.net/Articles/674102/>) is a good opportunity to look at symbol exports and how they work. 

The actual implementation of `EXPORT_SYMBOL()` can be found in [`include/linux/export.h`](<http://lxr.free-electrons.com/source/include/linux/export.h>). Whenever this macro is invoked, it declares a `kernel_symbol` structure for the exported symbol: 

```
struct kernel_symbol
        {
    	unsigned long value;
    	const char *name;
        };
```

As one might expect, the `name` field is set to the name of the symbol, while `value` becomes its address. When the code is compiled, though, this structure is not placed with the rest of the surrounding object code; instead, it goes into a special ELF section called `__ksymtab` (or `__ksymtab_gpl` for GPL-only symbols). The kernel binary contains these sections; any module that exports symbols of its own can also have them. When the kernel boots (or a module that exports symbols is loaded), that section is read and the symbol table is populated from the structures found there. The symbol table can then be used to satisfy references from modules to exported symbols. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

In theory, the symbol-export mechanism limits the API available to loadable modules. There was once hope that this API could be kept to a relatively small and well-defined set. A quick grep in the kernel repository reveals, though, that there are currently over 27,000 exported symbols — not exactly a small set. When you have that many symbols, simply maintaining them all becomes a bit of a challenge. 

One rule of thumb meant to help with the maintenance of exported symbols is that the actual `EXPORT_SYMBOL()` directive should appear next to the function or data structure that it exports. That allows the function and its export declaration to be modified together. This rule is not always observed, though. Sometimes it's just a matter of old code that predates the adoption of this rule but, more often, it is actually the result of a couple of limitations in the export mechanism: 

  * The macros found in `<linux/export.h>` are written in C and use GCC-specific extensions; they do not work in assembly-language code. So the export declarations for any functions written in assembly must appear in a separate, C-language file. 

  * Code that is built into a separate library prior to being linked into the kernel image has a potential surprise of its own. If nothing that is built into the main kernel image uses the exported object, the linker will leave it out of the build. Later, when a module is loaded needing that symbol, the load will fail because that symbol is not actually present. One way to work around this problem is to put the export declaration in code that's known to be built in — away from the object actually being exported. 

Addressing these limitations is the goal of Al's patch set. Fixing the first one is relatively easy; it is mostly just a matter of writing a version of `<linux/export.h>` that uses the necessary assembler directives to create the `kernel_symbol` structures in the proper section. There are some details related to alignment requirements on some architectures, but they do not appear to have been that hard to get around. Once Al's patches are applied, assembly code can include `<asm/export.h>` and use `EXPORT_SYMBOL()` in the usual way. 

The solution to the second problem is a bit of scripting trickery. As part of the build process, `objdump` is run on any library objects to obtain a list of exported symbols. A dummy object file (`lib-ksyms.o`) is then created with an explicit reference to each exported symbol; that object file is linked directly into the kernel. That will cause the linker to pull in all of the exported functions as well, ensuring that they will be available later on when a module is loaded. That eliminates an annoying trap that can spring on unsuspecting users years in the future when they happen on a configuration that fails to pull in objects they will need. 

The bulk of the patch set is a set of cleanups enabled by the above changes; in particular, a lot of `EXPORT_SYMBOL()` and `EXPORT_SYMBOL_GPL()` declarations are moved into assembly code next to the objects they are exporting. In the process, Al found a number of dusty corners where unused functions could be removed; as he put it: ""I'm fairly sure that more junk like that is out there; that's one of the reasons why exports really ought to be near the definitions."" 

Not all of those cleanups need to be merged anytime soon, though; they can happen anytime after the enabling patches go into the mainline. So that part of the patch set will likely be left in the hands of the specific architecture maintainers (assembly-language code, by its nature, is found in the architecture-specific parts of the kernel tree). The core changes are straightforward and uncontroversial; there is unlikely to be much keeping them out of the mainline. So, in the near future, one longstanding build-system annoyance should be history.
