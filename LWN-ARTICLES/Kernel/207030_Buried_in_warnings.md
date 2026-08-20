---
title: Buried in warnings
url: https://lwn.net/Articles/207030/
date: "November 1, 2006"
category: "Development model-Kernel quality; Remapper"
author: 
---

The 2.6.19-rc4 prepatch release did not go quite as well as the developers might have liked; some confusion over the return type for an internal function led to an undesirable mixing of pointer and integer types in the depths of the block layer. As it turns out, gcc noticed this problem and duly issued warnings about it, but nobody saw them before the mistaken patch was merged and the resulting kernel shipped. This is, in other words, a problem which should have been easily avoidable. 

Linus [responded](<https://lwn.net/Articles/207032/>) this way: 

And I have SYSFS enabled, so I should have seen this warning. 

But I've become innoculated against warnings, just because we have too many of the totally useless noise about deprecation and crud, and ppc has it's own set of bogus compiler-and-linker-generated warnings.. 

At some point we should get rid of all the "politeness" warnings, just because they can end up hiding the _real_ ones. 

A few kernel developers were doubtless wondering just why it took so long to reach this point - there have been complaints about excessive warnings for some time now. There is a lot of support for having the computer find problems whenever possible, and that has led to an increasing number of "must check" annotations and other changes which cause warnings to be issued whenever something looks suspicious. On top of that, gcc generates a fair number of warnings in situations where no real problems exist. The end result is that warnings which refer to real problems tend to get lost in the flood. 

Patches which address many of the spurious "this variable might not be initialized before being used" warnings have been circulating for some time. There is resistance to applying them, however; some developers resent cluttering up the code (and bloating the kernel) with unneeded initializations to deal with what they see as a gcc bug. There is no real sign that this latest episode has changed the thinking on that score; the initialization patches may well continue to languish. 

A different approach has been taken by Al Viro. He has developed a little tool called "remapper" which tracks how blocks of code move around from one kernel version to the next. Using the generated information, a set of compiler warnings from an old kernel can be remapped to their line numbers in a newer kernel. Then, a tool like `diff` can be used to compare the output from old and new compiles; the end result is a listing of the warnings which first appear in the new kernel - and only those. With this filtered output, developers can quickly find places where the compiler has pointed out real problems. 

Remapper can be had via git from: 

```
git://git.kernel.org/pub/scm/linux/kernel/git/viro/remapper.git
```

Dave Jones also makes [daily snapshots](<http://www.codemonkey.org.uk/projects/git-snapshots/remapper/>) available. 

Use of remapper is relatively straightforward: after building the `remap-log` tool, one starts with a command like this: 

```
diff-remap-data 2.6.19-rc2 2.6.19-rc3 > 2-to-3.map
```

The resulting "map" file is full of file names and numbers; they simply map line numbers from the old directory tree to the new one - and mark blocks of code which were removed altogether. There is another tool (`git-remap-data`) which performs the same task for two commits in a git repository; in this case, file renames can be handled properly as well. 

The `remap-log` tool can then be used to move old compile logs into the present: 

```
remap-log 2-to-3.map < 2.6.19-rc2.log > 2.6.19-rc2-remapped.log
```

If the new log is then compared to the output from a 2.6.19-rc3 build with `diff`, the only output will be any warnings (or errors) which have appeared or disappeared between the two kernel versions. Those which have only moved due to changes elsewhere in the file will be filtered out. The [short documentation file](<https://lwn.net/Articles/207021/>) packaged with the code offers some other potential uses, such as carrying forward annotated `grep` output as an ongoing "to do" list. 

Some developers swear by this tool. Jeff Garzik, however, is not entirely pleased; in an earlier discussion he [said](<https://lwn.net/Articles/207050/>): 

I think it's both sad, and telling, that the high level of build noise has trained kernel hackers to tune out warnings, and/or build tools of ever-increasing sophistication just to pick out the useful messages from all the noise. 

Jeff has, instead, [put together a separate kernel tree](<http://lwn.net/Articles/202249/>) with many of the bogus warnings silenced. It is a labor-intensive task - each warning must be investigated and shown to be spurious before being quieted. This work is not intended for merging; instead, it's meant to help create a development platform in which the useful warnings can actually be seen. This set of changes has been part of the -mm tree since 2.6.18-mm3. 

Yet another approach to the "may be uninitialized" warnings was [floated last May](<http://lkml.org/lkml/2006/5/11/50>); it introduces a special macro which "initializes" a variable without actually doing anything. That silences the warning without adding to the size of the kernel. The macro is only supposed to be used in cases where the code paths have been audited. The objection that was raised at the time was that, while the current use of a variable might be correct, future changes to the code could introduce a path where that variable is, indeed, used without initialization. The warning would still be suppressed, however, and the bug might not be caught until much later. So the patch was never merged. 

Compiler bugs can, perhaps, eventually be fixed. But the increasing interest in the use of automated tools to find potential bugs all but guarantees that there will continue to be a stream of spurious warnings for developers to deal with. If those automated warnings are to lead to real fixes - before somebody gets burned - ways of keeping the noise level down will have to be found.
