---
title: A new version of modversions
url: https://lwn.net/Articles/986892/
date: "August 26, 2024"
category: "Build system; Modules-Exported symbols; Releases-6.14"
author: "By Jonathan Corbet August 26, 2024"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
August 26, 2024

The `genksyms` tool has long been buried deeply within the kernel's build system; it is one of the two C-code parsers shipped with the kernel (the other being [the horrifying `kernel-doc` script](<https://elixir.bootlin.com/linux/v6.11-rc4/source/scripts/kernel-doc>)). It is a key part of how the kernel's module-loading infrastructure works. While `genksyms` has quietly done its job for decades, that period may soon be coming to an end. It would seem that `genksyms` is not up to the task of handling Rust code, so Sami Tolvanen is [proposing a new tool](<https://lwn.net/ml/all/20240815173903.4172139-21-samitolvanen@google.com/>) to handle this task going forward. 

In the early days, the kernel only supported monolithic builds; there was no concept of loadable modules. That changed with the [0.99.15 release](<https://git.kernel.org/pub/scm/linux/kernel/git/tglx/history.git/commit/?id=a4c5b0f7436>) in early 1994, which added module support along with a number of other features. That release also was the beginning of the "code freeze" for the 1.0 release; Linus Torvalds said at the time: 

> Bumping the linux version number to 1.0 doesn't mean anything more than that: it's only a version number change. More explicitly, it does *NOT* mean that linux will become commercial (the copyright will remain as-is), nor does it mean that development stops here, and that 1.0 will be anything special in that respect. 

Loadable modules at that time were tied to the specific version of the kernel they were built for. As Linux steadily became more commercial, though, there was an increase in interest in the ability to load a given binary module into multiple kernel versions. That interest was especially strong among those who were shipping out-of-tree modules and wanted those modules to work in as many kernels as possible. 

Then, as is the case now, modules hooked into the rest of the kernel by way of exported symbols. Only symbols (corresponding to functions or variables) that have been explicitly exported are available to a loadable module, and modules can export symbols of their own as well. When a module is compiled to use a specific symbol, it naturally incorporates the type information associated with that symbol. Any change to a symbol (adding a parameter to a function prototype, for example, or rearranging the fields in a structure definition) will break modules that use those symbols, possibly in subtle and unpleasant ways. 

#### The beginning of modversions

Kernel development has always moved quickly; that results in many changes to the interface seen by loadable modules. There was some pressure to try to minimize such changes in the early days, but the project's policy has always been that the kernel's internal interfaces can change at any time. The best that maintainers of out-of-tree modules can do is to notice the changes and update their code accordingly. 

Even noticing the changes can be hard, though, since kernel developers do not normally worry much about making changes evident to external code. But that is a job that computers are well suited for. So, the 1.1.85 release (January 1995) added the first "modversions" support, intended to allow modules to be loaded into multiple kernel versions — and to make it clear when that is not possible. 

At the core of modversions is `genksyms`. This tool reads and parses a C source file that has been run through the preprocessor, collects the definition of every symbol exported by that file, calculates a checksum of each definition, and outputs the result in a form that the build process can access. The kernel's `modpost` tool then uses that information to create a C source file that populates a special ELF section with symbols (both exported and used) along with their checksums. The curious can see [an example](<https://lwn.net/Articles/986905/>) of what that file looks like. When the time comes to load a module, the loader compares the checksum of every symbol used by that module against the checksums for the running kernel; if the two match, then the module can be loaded safely. 

`genksyms` was initially shipped separately from the kernel in the `modutils` package; that [changed](<https://git.kernel.org/pub/scm/linux/kernel/git/tglx/history.git/commit/?id=46bd1da672d>) for the 2.5.64 development kernel in February 2003, when it was moved into the kernel's `scripts` directory. The tool has been maintained over the years without huge changes; it has seen (slightly) fewer than 100 patches during the Git era. 

`genksyms` has been able to work for so long because the declaration of a function or data structure in C fully describes the interface to the resulting binary object, at least once the options passed to the compiler are taken into account. The compiler is not going to rearrange things into a form that, for whatever reason, it likes better. But, as Tolvanen explains, the same is not true for Rust: 

> Unlike C, Rust source code doesn't have sufficient information about the final ABI, as the compiler has considerable freedom in adjusting structure layout for improved performance, for example, which makes using a source code parser like genksyms a non-starter. 

As a result of this problem, a kernel configuration cannot simultaneously enable both modversions and Rust. Naturally, this is not welcome news for anybody who wants to enable Rust and still load binary modules into multiple kernel versions. Since Android falls into that category, it is not surprising that some attention has gone toward remedying this situation. 

#### Tossing DWARF at Rust

Teaching `genksyms` how to parse Rust would not be a small job and, as noted above, it would not be a sufficient solution in any case. The chosen solution, instead, is to stop parsing the code and trying to second-guess the compiler; instead, the DWARF debugging information produced by the compiler, which fully describes the interfaces of interest, is used. Taking this approach, the new `gendwarfksyms` tool can conceivably do symbol versioning for code written in any language. 

The [first version of this patch set](<https://lwn.net/ml/linux-kernel/20240617175818.58219-17-samitolvanen@google.com/>) used the new tool to generate checksums for Rust code, while retaining the existing machinery for C code. Modules maintainer Luis Chamberlain had [suggested](<https://lwn.net/ml/linux-kernel/ZnHjO7x9nszs9pUC@bombadil.infradead.org/>), though, that it should just be used for all code, so that is what the current series does. 

This solution seems promising, but it not without its downsides. The new tool can only process the DWARF data if it exists, meaning that the kernel must be compiled with full debugging information enabled. That is not necessarily a problem for distributors, since they typically want that information anyway, but generating all that information slows down the build process significantly. Build-time regressions for the kernel get developer attention more quickly than almost any other kind of problem, so this change may not be universally welcomed. 

While `gendwarfksyms` generates checksums like `genksyms` does, it does not generate the _same_ checksums. Switching an existing kernel to use `gendwarfksyms` will thus result in all checksums changing, and any existing modules will no longer be loadable. That makes this switch into a sort of flag day that would have to be carefully managed by distributors. 

The [`modversion_info` structure](<https://elixir.bootlin.com/linux/v6.11-rc4/source/include/linux/module.h#L37>) used to hold symbol names and checksums is limited to names that are no longer than 55 characters; after all, that should be enough for anybody. But it is not enough for the Rust compiler, which uses name mangling to encode type information into identifiers. Changing that structure would break user-space tools, and is thus not an easy option. The first version of the series hashed longer names to make them fit into the available space, but that behavior has been removed in the second posting; instead, a [separate effort](<https://lwn.net/ml/linux-kernel/20240806212106.617164-1-mmaurer@google.com/>) by Matthew Maurer is working toward making the representation of this information more flexible. 

Another interesting challenge was [described](<https://lwn.net/ml/linux-kernel/0b2697fd-7ab4-469f-83a6-ec9ebc701ba0@suse.com/>) by Petr Pavlu. There are distributors that try to maintain ABI compatibility for kernel modules, even though the kernel project itself is indifferent (or hostile) to that goal. One trick they use is to identify data structures that, they think, may gain additional elements over the support life of their kernel. Those structures will be augmented with some placeholder fields that are compiled out when `__GENKSYMS__` is set, which only happens during `genksyms` runs. That allows them to, at some future time, change the type of that field while hiding the change from `genksyms`, and thus avoiding an apparent ABI change. 

The first version of the series did not support this feature, but it was added in the second. Use of preprocessor macros will not work when dealing with DWARF data, though; a different approach is needed. So, instead, a different hack was developed using the names of the symbols. The new mechanism is described in [this patch](<https://lwn.net/ml/all/20240815173903.4172139-37-samitolvanen@google.com>); imagine a kernel structure that has been augmented in a distribution's kernel with a placeholder field like this: 

```
struct struct1 {
        	long a;
        	**long __kabi_reserved_0; /* reserved for future use */**
        };
```

`gendwarfksyms` recognizes the `__kabi_reserved_` prefix, especially when it appears in unions. At some future time, the above structure could be changed to something like: 

```
struct struct1 {
        	long a;
        	union {
                long __kabi_reserved_0;
          	    struct {
                    int b;
              	int v;
          	    };
            };
        };
```

The DWARF data describing this structure will change accordingly, but `gendwarfksyms` will replace the union with just the definition of `__kabi_reserved_0`, with the result that the structure appears unchanged for the purpose of checksum generation. 

There was some ongoing discussion about this last trick, especially with regard to how it would work on the Rust side. Bare union types run counter to the Rust way of doing things since they provide no way to ensure that the correct field is used at any given time, so there has been talk of providing something a bit more complex. 

There is also some concern about how the overall approach will interact with link-time optimization, which will currently slow the build process even more; this series currently explicitly conflicts with link-time optimization for that reason. For the most part, though, reviewers seem reasonably happy with the current form of the patch series. Once a solution to the long-name problem has been worked out, entry into the mainline may follow soon.
