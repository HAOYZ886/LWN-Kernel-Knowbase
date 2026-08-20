---
title: Coccinelle for Rust
url: https://lwn.net/Articles/991399/
date: "September 30, 2024"
category: "Development tools-Coccinelle"
author: "By Daroc Alden September 30, 2024 Kangrejos 2024"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Daroc Alden**  
September 30, 2024

* * *

[Kangrejos 2024](<https://lwn.net/Articles/990496/>)

Tathagata Roy has been working to make the [ Coccinelle](<https://coccinelle.gitlabpages.inria.fr/website/>) tool that is used (among other things) to automate the refactoring of C code work on Rust code as well. Roy gave a presentation at [Kangrejos](<https://kangrejos.com/>) about that work, including the creative approaches necessary to work with Rust's more complicated control flow and syntax. 

Roy opened by describing the purpose of [ Coccinelle within the kernel](<https://docs.kernel.org/dev-tools/coccinelle.html>). Linux's code is huge, and frequently a change to some internal API will need to be reflected across a large number of drivers or other kernel components; Coccinelle allows maintainers to [ write patches in a special language](<https://lwn.net/Articles/315686/>) that can be automatically applied across the entire code base, making it easier to make changes that have a broad impact. This kind of change requires a specialized tool since parsing C source code is not trivial. 

Although there is much less Rust code in the kernel, it would be nice if the same tool that existing kernel maintainers are familiar with also worked on Rust code. Roy presented an example of what that might look like; in short, authors would follow the same process as for C code. To construct a semantic patch for Coccinelle, the programmer picks a typical diff for the desired change, and then makes it more generic by adding "expression variables". An example of a patch for a Rust API change might look like this: 

```
@@
        expression tcx, arg;
        @@
    
        - tcx.type_of(arg)
        + tcx.bound_type_of(arg).subst_identity()
```

That patch finds uses of `.type_of()` and rewrites them to call a different function, even if the method is being invoked on or with complex expressions. 

Under the surface, however, correctly supporting Rust requires a lot of thought. The biggest issue Roy highlighted is more complex control flow. In order to apply patches correctly, Coccinelle needs to match patch expressions to the abstract syntax tree (AST) of the program. This requires knowledge of the program's control flow, Roy explained. In Rust, this includes wrinkles such as if expressions. It's valid to write code like this in Rust: 

```
println!("{}", if boolean { "string 1" } else { "string 2" })
```

This can lead to some counterintuitive control-flow graphs. The center of Roy's recent work has been getting the representation of Rust programs into a form that can be processed by Coccinelle's existing pattern-matching algorithm. His approach works by doing a preorder traversal of the AST, converting it into a representation of the control flow in the program, and then matching on that, instead of on the raw AST. This approach works, but still has some problems, he said. It results in "huge" graphs, which take time for Coccinelle to process, and make debugging difficult. 

The graphs can be compressed by 4-5 times by eliminating intermediate nodes with only one child, but that does not present a complete solution. Another issue with this representation is dealing with metavariables (such as `tcx` and `arg`, above), Roy explained. Metavariables can stand in for any expression, which makes matching them to parts of the control-flow graph somewhat tricky. So, to make matching easier, the graph is also decorated with extra edges that point to the next child (or closest cousin) of a node. This makes it much easier to efficiently determine where an expression ends. 

[ ![\[Tathagata Roy\]](https://static.lwn.net/images/2024/tathagata-roy-small.png) ](<https://lwn.net/Articles/991562>)

This new internal representation isn't the only work-in-progress for Coccinelle. Roy explained that Coccinelle for Rust had recently added an ellipsis operator that can stand in for any control-flow path connecting two nodes. This makes it easier to write rules that match two different call sites separated by some arbitrary code. 

Matching a pattern that contains an ellipsis requires considering different potential matches, which means considering multiple alternatives, so it is a form of disjunction. Previously, Coccinelle only permitted disjunctions of expressions; now, Roy is working on generalizing disjunctions to include arbitrary statements. This has proved particularly difficult for Rust code, because of Coccinelle's approach to parsing. Coccinelle's C parser is hand-written, and can therefore include extensions for parsing Coccinelle-specific constructs in semantic patches into the same structure as the adjacent C code. Coccinelle's Rust parser, on the other hand, is the stock parser provided by the [ rust-analyzer project](<https://rust-analyzer.github.io/>) — because Roy did not want to rewrite an entire Rust parser and worry about keeping it up to date. This makes parsing Coccinelle's syntax for disjunctions tricky. 

The solution Roy has selected is to convert disjunctions into Rust macros by simple textual substitution, parse the resulting Rust code with rust-analyzer, and then recognize the macro name and treat it specially. This has the advantage that it permits easier parsing, but the disadvantage that it doesn't allow disjunctions of attributes, enum alternatives, or other constructs that can't be wrapped in macros. Roy emphasized that support for disjunctions was still a work in progress. 

There are two other problems that Roy wants to see resolved before calling the work complete: allowing Coccinelle to work on actual macros (which he described as "a pain in the AST") and processing patches in parallel. When these are finished, Coccinelle will hopefully prove to be as useful a tool for Rust programmers as it is for C programmers. People wishing to use the current (experimental) support for Rust can find [ Roy's development version](<https://gitlab.inria.fr/coccinelle/coccinelleforrust/-/tree/ctl2?ref_type=heads>) in the [ CoccinelleForRust repository](<https://gitlab.inria.fr/coccinelle/coccinelleforrust/>) — although the linked branch does not include the in-progress support for disjunctions.
