---
title: "The end of modversions?"
url: https://lwn.net/Articles/707520/
date: "November 30, 2016"
category: Build system; Modules
author: "By Jonathan Corbet November 30, 2016"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
November 30, 2016

The 4.9-rc1 kernel prepatch, [released](<https://lwn.net/Articles/703664/>) on October 15, introduced a large set of new features — and, inevitably, a smaller set of new regressions. One of those problems, a module-related bootstrap failure, remains unfixed in the mainline even after the [4.9-rc7](<https://lwn.net/Articles/707447/>) release. A fix to the problem has been written and is known to work, but it may never be merged if, as seems reasonably likely, the community chooses a simpler option. 

#### The problem of module compatibility

Loading modules into the kernel is a tricky business. Among other things, the module must precisely match the kernel into which it is being loaded in any of a number of ways. If a function prototype differs between the module and the kernel, bad things are sure to happen when that function is called. The same holds for data-structure layouts, configuration options, and even the version of the compiler used to build the various pieces. The obvious way to be sure that everything matches is to build the kernel and all loadable modules together; that is, indeed, how it is done most of the time. But there are users who want to be able to build the kernel and its modules separately. 

One obvious use case for separately built modules is code that is not in the mainline, and, thus, cannot be built with the rest. There are also cases where users want to build and run a new kernel without necessarily rebuilding the modules that they use. Supporting these users while trying to protect the kernel against the loading of incompatible modules has led to the addition of a couple of layers of infrastructure. 

The first of those is the "vermagic" string compiled into the kernel and into every loadable module. The system on which this article is being written features the following vermagic string: 

```
4.8.6-2-default SMP preempt mod_unload modversions
```

In the simplest configuration, the module loader will simply check to ensure that a module and the kernel have the exact same vermagic string. That ensures that the module was built for the same kernel version and that major options like SMP support were configured in the same way. If the test fails, the module will not be loaded. 

That test, however, will thwart users who want to use the same binary module in multiple versions of the kernel. Even users who have a module built for a distribution kernel will run into trouble when the distributor ships an update; the version number will increment to something like 4.8.6-3 and the test will fail, even though the new kernel only adds a few fixes and is almost certainly compatible with the old module. Supporting those users requires a more nuanced compatibility test. 

The "modversions" configuration option is meant to be that test. When enabled, modversions changes both the compilation process and the module loader. When the kernel is built, a checksum is calculated from the prototype of every exported function; those checksums are stored in a special section of the binary. When modules are built, those same checksums are calculated for every exported function that the module calls; the result is built into the module binary. At module-load time, the kernel will drop the first part of the vermagic string (the kernel version number) before comparing it, meaning that modules can now be loaded into versions other than the one they were built for. But the loader will also compare the checksums for all kernel symbols used by the module; should one of those checksums fail to match, the module will not be loaded. This test will, thus, catch major changes in the functions used by modules, but it still cannot catch more subtle changes. 

#### Recent changes and modversions

Back in February, Al Viro posted [a set of changes](<https://lwn.net/Articles/674303/>) to the symbol-export mechanism; these changes were designed to, among other things, allow the placement of `EXPORT_SYMBOL()` directives in assembly code for functions defined there. These changes, merged into the mainline for 4.9-rc1, improved symbol exports in a number of ways, but there was one little problem: the generation of checksums for symbols exported from assembly code does not work properly with binutils 2.27. In particular, those checksums (which were set to zero anyway) would be dropped entirely; the module loader would then complain about the missing checksums and refuse to load the module. As a result, systems with that version of binutils and with modversions enabled will fail to boot if they require a module that uses symbols defined in assembly code. 

One [fix](<https://git.kernel.org/linus/4efca4ed05cbdfd13ec3e8cb623fb77d6e4ab187>), developed by Nick Piggin, is to create a special include file containing prototypes for functions exported from assembly code; the build process can read that file to generate the necessary modversions checksums. That ensures that the checksums are not only present, but also that they correspond to the symbols and can be meaningfully checked. This fix was merged for 4.9-rc6, but it failed to actually fix the problem because it did not finish the job. Functions defined in assembly code are, by their nature, architecture-specific, so the include file containing the prototypes must be created for each architecture. Those files were not actually created for any architecture beyond PowerPC so, as of 4.9-rc7, users of other architectures (i.e. most of us) can still run into the problem. Adam Borowski has posted [a patch](<https://lwn.net/Articles/707526/>) adding this file for the x86 architecture, but it has not been merged as of this writing. 

And, indeed, it may never be merged, because it seems that most of the use cases for modversions no longer exist. Some distributors (notably Debian) make use of it but, since they take pains to not change APIs in supported kernels, all they really gain is the ability to avoid the kernel-version check (though Debian also [counts on modversions](<https://lwn.net/Articles/707728/>) to allow internal API changes to be made without changing the kernel version). As Linus Torvalds [noted](<https://lwn.net/Articles/707527/>), the feature was once useful for developers who were tired of tracking down problems that were caused by stale kernel modules. In 2016, where the kernel version can contain the actual Git revision that was built and where the time required to build a full set of modules is short, modversions is no longer as useful as it once was. And, Piggin [noted](<https://lwn.net/Articles/707528/>), modversions uses a fair amount of complicated machinery for a mediocre result: 

But still, modversions is pretty complicated for what it gives us. It sends preprocessed C into a C parser that makes CRCs using type definitions of exported symbols, then turns those CRCs into a linker script which which is used to link the .o file with. What we get in return is a quite limited symbol "versioning" system. 

By "quite limited," he is referring to the fact that many changes will elude the modversions check. In particular, changes to a structure passed to a function will not be caught. Piggin suggested that a better result could be obtained if the whole mechanism were removed and replaced by a simple, manually maintained version number attached to each exported symbol. Whenever a developer made an incompatible change, they would be expected to increment the version number; modules using the affected interface would then fail to load until they were rebuilt. 

The version-number suggestion did not get far; the chances of those numbers actually being maintained in a useful manner are quite small. But the idea of removing modversions was better received. Torvalds agreed that the whole thing ""may just be too painful to bother with"" and that the number of users is quite small — an idea reinforced by the fact that few testers complained about this issue. So, rather than apply the fix, Torvalds chose instead to [mark modversions as "broken"](<https://git.kernel.org/linus/cd3caefb4663e3811d37cc2afad3cce642d60061>) (essentially disabling the feature altogether) instead. That change was merged just prior to the 4.9-rc7 release. 

It seems, though, that not everybody is ready to see modversions go away quite yet; in particular, Debian, which is planning on using 4.9 for the upcoming "stretch" release, would like to have modversions available. So, after 4.9-rc7 was released, Torvalds committed [another change](<https://git.kernel.org/linus/faaae2a581435f32781a105dda3501df388fddcb>) re-enabling modversions, but with a change. Rather than refuse to load a module when a checksum is missing, the loader will log a complaint and continue. That should suffice to get modversions working again on all systems without requiring the addition of architecture-specific include files. His real goal is clear, though: ""Some day I really do want to remove MODVERSIONS entirely. Sadly, today does not appear to be that day."" 

When that day does come, Piggin has [a patch](<https://lwn.net/Articles/707729/>) removing modversions altogether and replacing it with a simple option for distributors to supply their own ABI version string to be used instead of vermagic. Getting rid of modversions removes about 7,700 lines of code (much of which is generated by lex and bison) and simplifies the module-loading logic. It seems like a relatively easy sell — if distributors agree that they can do without modversions in the future.
