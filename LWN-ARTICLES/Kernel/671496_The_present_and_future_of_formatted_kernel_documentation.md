---
title: The present and future of formatted kernel documentation
url: https://lwn.net/Articles/671496/
date: "January 13, 2016"
category: Documentation
author: "By Jonathan Corbet January 13, 2016"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
January 13, 2016

The kernel source tree comes with a substantial amount of documentation, believe it or not. Much of that can be found in the `Documentation` tree as a large set of rather haphazardly organized plain-text files. But there is also quite a bit of documentation embedded within the source code itself that can be extracted and presented in a number of formats. There has been an effort afoot for the better part of a year to improve the capabilities of the kernel's formatted-documentation subsystem; it's a good time for a look at the current state of affairs and where things might go. 

Anybody who has spent much time digging around in the kernel source will have run across the specially formatted comments used there to document functions, structures, and more. These "kerneldoc comments" tend to look like this: 

```
/**
         * list_add - add a new entry
         * @new: new entry to be added
         * @head: list head to add it after
         *
         * Insert a new entry after the specified head.
         * This is good for implementing stacks.
         */
```

This comment describes the `list_add()` function and its two parameters (`new` and `head`). It is introduced by the "`/**`" marker and follows a number of rules; see [`Documentation/kernel-doc-nano-HOWTO.txt`](<https://lwn.net/Articles/671502/>) for details. Normal practices suggest that these special comments should be provided for all functions meant to be used outside of the defining code (all functions that are exported to modules, for example); some subsystems also use kerneldoc comments for internal documentation. 

The documentation subsystem is able to extract these comments and render them into documents in a number of formats, including plain text, man pages, HTML, and PDF files. This can be done in a kernel source tree with a command like "`make mandocs`" or "`make pdfdocs`". There is also [a copy of the formatted documentation](<https://www.kernel.org/doc/htmldocs/>) on kernel.org; the end result for the comment above can be found on [this page](<https://www.kernel.org/doc/htmldocs/kernel-api/API-list-add.html>), for example. The results are not going to win any prizes for beautiful design, but many developers find them helpful. 

#### Inside kernel-doc

The process of creating formatted documents starts with one of a number of "template files," found in the `Documentation/DocBook` directory. These files (there are [a few dozen of them](<http://lxr.free-electrons.com/source/Documentation/DocBook/>)) are marked up in the [DocBook](<http://www.docbook.org/>) format; they also contain a set of specially formatted (non-DocBook) lines marking the places where documentation from the source should be stuffed into the template. Thus, for example, `kernel-api.tmpl` contains a line that reads: 

```
!Iinclude/linux/list.h
```

The `!I` directive asks for the documentation for all functions that are _not_ exported to modules. It is used rather than `!E` (which grabs documentation for exported functions) because the functions, being defined in a header file, do not appear in an `EXPORT_SYMBOL()` directive. 

Turning a template file into one or more formatted documents is a lengthy process that starts with a utility called `docproc`, found in the `scripts` directory. This program (written in C) reads the template file, finds the special directives, and, for each of those directives, it does the following: 

  * A pass through named source file is made, and each of the `EXPORT_SYMBOL()` directives found therein is parsed and the named function added to the list of exported symbols. 

  * A call is made to `scripts/kernel-doc` (a 2,700-line Perl script) to locate all of the functions, structures, and more that are defined in the source file. `kernel-doc` tries to parse the C code well enough to recognize the definitions of interest; in the process, it attempts to deal with some of the kernel's macro trickery without actually running the source through the C preprocessor. It will output a list of the names it found. 

  * `docproc` calls `kernel-doc` again, causing it to parse the source file a second time; this time, though, the output is the actual documentation for the functions of interest, with some minimal DocBook formatting added. 

The formatted output is placed into the template file in the indicated spot. If the target format is HTML, the `kernel-doc-xml-ref` script is run to generate cross-reference links. This feature, only added in 4.3, can only generate links within one template file; cross-template links are not supported. 

The final step is to run the documentation-formatting tool to actually create the files in the format of interest. Most of the time, the `xmlto` tool is used for this purpose, though there are some provisions in the makefile for using other tools. 

In other words, this toolchain looks just like what one might expect from a documentation system written by kernel developers. It gets the basic job done but it is not particularly pretty or easy to use. It is somewhat brittle, making it easy for developers to break the documentation build without knowing it. Numerous developers have said that they have given up on trying to actually get formatted output from it; depending on one's distribution, getting all of the pieces is place is not always easy. And a lot of potentially desirable features, like cross-file links, indexing, or formatting within the in-source comments, are not present. 

#### Formatted comments

The latter issue — adding formatting to the kerneldoc comments — has been the subject of some work in recent times. Daniel Vetter has a long-term goal of putting much more useful graphics-subsystem information into those comments, but has found the lack of formatting to be an impediment once one gets beyond documenting function prototypes. To fix that, Intel funded some work that, among other things, produced [a patch set](<https://lwn.net/Articles/652234/>) allowing markup in the comments. Nobody really wants to see XML markup in C source, though, so the patch took a different approach, allowing markup to be done using the [Markdown](<https://en.wikipedia.org/wiki/Markdown>) language. Using Markdown allowed a fair amount of documentation to be moved to the source from the template file, shedding a bunch of ugly XML markup on the way. 

This work has not yet been merged into the mainline. Daniel [has his own hypothesis](<http://blog.ffwll.ch/2016/01/better-markup-for-kernel-gpu-docbook.html>) as to why: 

Unfortunately it died in a bikeshed fest due to an alliance of people who think docs are useless and you should just read the code, and others who didn't even know how to convert the kerneldoc into something pretty. 

Your editor (who happens to be the kernel documentation maintainer, incidentally), has a different hypothesis. Perhaps this work remains outside because: (1) it is a significant change affecting all kernel developers that shouldn't be rushed; (2) it used [pandoc](<http://pandoc.org/>), requiring, on your editor's Fedora test box, the installation of 70 Haskell dependencies to run; (3) it had [unresolved problems](<http://lists.freedesktop.org/archives/dri-devel/2015-September/090841.html>) stemming from disagreements between `pandoc` and `xmlto` regarding things like XML entity escaping; and (4) a certain natural reluctance to add another step to the kernel documentation house of cards. All of these concerns led to [a discussion at the 2015 Kernel Summit](<https://lwn.net/Articles/662930/>) and a lack of enthusiasm for quick merging of this change. 

All that notwithstanding, there is no doubt that there is interest in adding formatting to the kernel's documentation comments. Your editor thinks that there might be a better way to do so, perhaps involving the removal of `xmlto` (and DocBook) entirely in favor of a Markdown-only solution or a system like [Sphinx](<http://www.sphinx-doc.org/en/stable/>). Unfortunately, your editor has proved to be thoroughly unable to find the time to actually demonstrate that such an approach might work, and nobody else seems ready to jump in and do it for him. Meanwhile, the Markdown patches have been reworked to use [AsciiDoc](<http://www.methods.co.nz/asciidoc/>) (which can be thought of as a rough superset of Markdown) instead. That change gets rid of the Haskell dependency (replacing it with a Python dependency) and improves some formatting features at the cost of slowing the documentation build considerably. Even if it is arguably not the best solution, it is out there and working now. 

As a result, these patches will probably be pulled into the documentation tree (and, thus, into linux-next) in the next few weeks, with an eye toward merging in 4.6 if all looks well. It has been said many times that a subsystem maintainer's first job is to say "no" to changes. Sometimes, though, the right thing is to say "yes," even if said maintainer thinks that a better solution might be possible. A good-enough solution that exists now should not be held up overly long in the hopes that vague ideas for something else might turn into real, working code.
