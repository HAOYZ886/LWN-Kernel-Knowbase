---
title: Inside the mind of a Coccinelle programmer
url: https://lwn.net/Articles/698724/
date: "August 31, 2016"
category: "Development tools-Coccinelle"
author: "By Jake Edge August 31, 2016 Linux Security Summit"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jake Edge**  
August 31, 2016

* * *

[Linux Security Summit](<https://lwn.net/Archives/ConferenceByYear/#2016-Linux_Security_Summit>)

[Coccinelle](<http://coccinelle.lip6.fr/>) developer Julia Lawall started her keynote at the [Linux Security Summit](<http://events.linuxfoundation.org/events/linux-security-summit>) in Toronto by saying that she didn't know much about security. But the tool she developed, and the patches it has generated, have clearly fixed various patterns of bugs in the kernel, some of which have had security implications. In the talk, Lawall related some of what she has learned by [using the tool](<https://lwn.net/Articles/315686/>), while also introducing its capabilities to the roughly 90 attendees. 

[ ![\[Julia Lawall\]](https://static.lwn.net/images/2016/lss-lawall-sm.jpg) ](<https://lwn.net/Articles/698824/>)

The principle behind Coccinelle is to "find once, fix everywhere". It is a static analyzer to find patterns in C code. So when someone spots a problem in a large code base, Coccinelle can be used to find other examples of the same kind of bug. It can also then apply transformations to the code to create patches to fix those other occurrences. 

#### Semantic patches

Coccinelle is user-scriptable using "semantic patches", which are based on the patch notation familiar to developers. Ideally, she said, there is not much new that someone has to learn to start using the tool. The goal from the outset was to make it accessible to C developers. 

Lawall showed a classic example of where Coccinelle can be used. A [commit](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=e6bafba5b4765a5a252f1b8d31cbf6d2459da337>) message by Al Viro (""wmi: (!x & y) strikes again"") pointed to a recurring problem with tests that look like: 

```
if (!block->flags & ACPI_WMI_METHOD)
```

The proper fix is to parenthesize the expression: 

```
if (!(block->flags & ACPI_WMI_METHOD))
```

The problem is that the logical negation (`!`) operator binds more tightly than the bitwise-and (`&`) leading to an incorrect test. Given the message, it is clear this problem occurs with some frequency, but it is difficult to use `grep` to find this kind of pattern. There are too many `!` and `&` characters in the kernel and the test may stretch over multiple lines, which will also defeat simple searches. 

But a simple Coccinelle semantic patch will find (and fix by way of a generated patch) these kinds of problems: 

```
@@
        expression E;
        constant C;
        @@
        - !E & C
        + !(E & C)
```

Lawall then showed an example of the same problem (spanning two lines) elsewhere in the kernel and how the code was changed to fix it. 

#### History

Coccinelle began when Lawall was on sabbatical in 2004. At that time, the 2.6 kernel had recently been released, but many drivers were still targeting 2.4. She wanted to automate the porting of those drivers to 2.6, but that "turned out to be unrealistic". However, certain changes, which she called "collateral evolution", could be made automatically, such as changes to function calls like adding new parameters. 

To solve the problem of collateral evolution, Coccinelle was initially developed by Lawall and three others at the University of Copenhagen between 2005 and 2007. The first patches to the kernel based on Coccinelle output were submitted in 2007; they addressed places where `kmalloc()` calls were followed by a `memset()` and replaced those with calls to `kzalloc()`. Two papers followed, in 2008 and 2011. 

Currently, Coccinelle is developed by Lawall and three others at Inria. It is entirely implemented in OCaml, which may reduce the contributor base by a substantial amount, she said to some audience laughter. On the other hand, though, that does reduce the number of patches she has to review. 

The tool has had a fairly large impact on the kernel since the first patches were posted in 2007. Over 4,500 patches in the kernel mention Coccinelle; 3,000 of those are from more than 500 developers outside of the Coccinelle project. There are also 56 semantic patches used as tests that are part of the kernel (accessible via `make coccicheck`). There are more Linux-related semantic patches available at [coccinellery.org](<http://coccinellery.org/>). 

The semantic patches in the kernel are meant to catch various types of errors that can crop up. There are tests for generic C errors (such as testing for unsigned variables for values less than zero or null pointer dereferencing), generic Linux problems (double locks or using the iterator index after a loop), API-specific errors (using `free()` on a `devm` allocation), and API modernization (using `kmemdup()` rather than `kmalloc()` and `memcpy()`). Contributions of new semantic patches are welcome; contributors can either make the patch themselves or tell the team about the pattern for it to create the patch. 

#### Applications of Coccinelle

Lawall then presented some more complex applications of Coccinelle to the kernel. The first was an effort at "devmification"—switching drivers to use the `devm` framework to manage their memory to avoid memory leaks. A driver's probe function often allocates memory with `kzalloc()` that then gets freed in the remove function. Switching to `devm_kzalloc()` means that the `devm` library will manage the memory instead. 

There were three steps to the process; the first was to find the names of the probe and remove functions. For that, a regular expression could have been used, but that "seems unappealing", she said. Pointers to the functions are put into the `platform_driver` (and similar) structures, so their names can be extracted with a Coccinelle rule. The second step is to find the definition of the probe function and to transform the `kzalloc()` calls to `devm_kzalloc()`, while also removing the `kfree()` calls from any error paths. Lastly, it removes the `kfree()` call from the remove function. 

Lawall worked on this semantic patch in 2012 and ended up submitting 39 patches to the kernel to make that change. Her semantic patch finds over 170 opportunities to make the change, but they can't just be applied blindly; there is a responsibility to look at the patches that are generated, she said. The whole process takes just 30 seconds to run (using [GLIMPSE](<https://en.wikipedia.org/wiki/GLIMPSE>) indexing), so it changes the problem from finding the pattern to checking the fixes. 

Another application of Coccinelle was to detect user-level calls that might block due to a page fault (e.g. `copy_*_user()`, `get_user()`) that were being called under a spinlock. Blocking is not allowed under a spinlock, so those calls should not be made under the lock. The patch simply reported when it found one of the candidate functions between the `spin_lock()` (and variants) and the `spin_unlock()`. Normally, Coccinelle handles single functions at a time, but iteration can be used to scale it up to look at inter-function and inter-file patterns. 

One confirmed bug has been found with `copy_from_user()`. The `copy_to_user()` checking found a problem but there was a "FIXME" comment next to it, which might indicate the developer plans to get back to it at some point, she said. When that was met with laughter, she replied: "I understand your lack of confidence." That checking also found a false positive. She noted that she is not searching for perfection with the semantic patches, just trying to find things of interest, so she is not interested in complicating the semantic patch to eliminate all false positives. The `get_user()` and `put_user()` checking has found one occurrence of each, but she has not yet evaluated them. 

She also worked on a semantic patch for "constification"—adding the `const` keyword to some structures in the kernel. She used a four-step process that included two manual steps. She used Coccinelle to identify structures that only contain function pointers. Those are not the only ones that need `const`, she said, but they are often good candidates. In addition, protecting those structures from getting overwritten is important to avoid code execution in the kernel. From the list of such structures, she will choose one manually then use Coccinelle to change all uses of the type to add `const`. The last step is to build the kernel using GCC to see if it all works. 

She submitted 115 patches in 2015 for constification. She does not pay attention to whether they actually get merged. None have been turned down that she remembers, but some of them may not have gotten picked up. Detecting structures of all function pointers is slow—she runs it overnight—but adding `const` to a type is instantaneous. 

#### Lessons learned

Lawall has learned some lessons along the way that she wanted to pass along. The most important one is to start with something simple. The semantic patches in the kernel tree are more complex and are not representative of where to start with the tool. Start with a common case that might end up with some false positives; perhaps you get 100 results, which results in finding two bugs. Making it more complex is possible, of course, but there is a tradeoff there. 

For example, during the devmification process it was hard to figure out how to get rid of the right `kfree()` in the remove function. She chose a simple solution by just assuming that the kernel developers would use the same name for the variable to be freed that was used in the probe function. It is "completely unsafe, but it works fairly well". The worst case, though, would be false negatives—not removing a `kfree()` that should have been removed. So she did a test where she used Coccinelle to delete all of the `kfree()` calls in the remove functions, which turns out to be the same set of files as with her rule, so it "seems probable that the rule is OK" and "simple was good enough". 

Semantic patches should be developed incrementally, she said. The devmification rules resulted in 171 files, which might take some time to analyze. In addition, `devm` frees the memory it manages in a last-in-first-out order, which might cause differences. So she rewrote her semantic patch to avoid any value-returning function calls before the `kzalloc()` in the probe function or any remove functions that made calls after `kfree()`. That reduced the search space to 51 files. 

Similarly, the checking for blocking functions under a spinlock produced lots of false positives. Conditionals in the code may cause the lock status to be inaccurate in the analysis. To avoid that, she changed the test to look for places where the blocking calls appear after the lock release, then looked closely at places where that wasn't true. 

The last lesson she imparted was to exploit information from other tools. The constification effort used GCC to determine if the change would build. But, ideally, there would be ways to warn users about potentially difficult cases. In order to do that, you might want to gather information about the structures (are some already `const`, which might indicate that making them all `const` might work?) and about the kernel build with regard to the structure (are all of instances of the structures in files that are built with the current configuration?). Those would indicate structures that might be easier or harder to change and verify to help prioritize which ones to tackle. That kind of information can be gathered by Python or OCaml programs to help users narrow in on good candidates. 

In conclusion she said that the tool is useful for a number of different kinds of problems. Coccinelle "compromises on both soundness and completeness, but it is still useful in practice". Developers would be well-served by adding it to their arsenal. 

[I would like to thank the Linux Foundation for travel support to attend the Linux Security Summit in Toronto.]
