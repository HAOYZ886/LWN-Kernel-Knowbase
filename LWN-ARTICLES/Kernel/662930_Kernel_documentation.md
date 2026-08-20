---
title: Kernel documentation
url: https://lwn.net/Articles/662930/
date: "November 4, 2015"
category: Documentation
author: "By Jonathan Corbet November 4, 2015 2015 Kernel Summit"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
November 4, 2015

* * *

[2015 Kernel Summit](<https://lwn.net/Articles/KernelSummit2015/>)

Kernel documentation made the list of topics for discussion during the core day at the 2015 Kernel Summit. Your editor, who has been the documentation maintainer for about one year now, led this discussion. Among other things, that means that notes from this session are, well, nonexistent, so this writeup is entirely from memory. If any inconvenient things have been left out, it's purely accidental. 

One thing your editor has found over the course of the last year is that it's often not clear where the responsibility for documentation patches lies. Most kernel subsystems are well contained within their own directory subtrees — except that many of them also have files under `Documentation/`. Some maintainers want to manage documentation patches that relate to their subsystems, while others are happy to leave it to the documentation maintainer. There is a slow-moving effort underway to document these preferences in the `MAINTAINERS` file. If nothing else, that will help reduce your editor's email load; thanks to the wonders of `get_maintainer.pl`, he is copied on every patch that touches anything in the documentation tree. 

There was a bit of talk about whether it would make sense to split the documentation out into the various subsystem trees, but people seemed to feel that it would make things harder to find. It seems that kernel developers often use the "grep the documentation subtree" technique to search for information. 

The bulk of the session, though, was concerned with the structured documentation found in the `DocBook` subdirectory. A document here starts as a DocBook template file, which is read by the `docproc` utility to determine which source-code files to extract documentation comments from. Those files are passed to `kernel-doc`, which, using its own Perl-based C parser, finds all the symbol names of interest and passes them back to `docproc`. Then `docproc` invokes `kernel-doc` again to actually extract the documentation and do some basic markup. The end result is patched into the template file and passed to `xmlto` for formatting into HTML files, PDF files, man pages, and more. 

Various functionalities have been added to this mechanism; 4.3, for example, adds a simple facility for automatically adding cross-references within a single template file. In the end, the kernel community has spent years slowly building up its own special document-formatting system. With all due respect to the people in the room, your editor said, they just might not be the right crowd for that particular project. The whole thing is a bit of a house of cards; one need not look far to find kernel developers who have given up on making this toolchain actually work. 

There is a desire to develop things further, though; a current patch set adds the ability to format the in-code documentation as Markdown text. The patches adding this feature are relatively straightforward, but they depend on the `pandoc` utility to do the Markdown formatting. An attempt to install `pandoc` on a Fedora system led to a demand to drag in no less than 70 Haskell-language dependencies. Somehow that didn't seem like something the kernel community would be thrilled about. 

Your editor's question was: do we want to merge that work, or maybe consider more wide-ranging changes to the documentation toolchain, preferably in a direction that uses more standard tools supported by others? There was little enthusiasm for adding a `pandoc` dependency, but little consensus otherwise. Linus said that, while he finds the in-code documentation comments useful, he has never seen any real point in building the formatted manuals. As far as he is concerned, that feature could just be removed. There was some agreement with that position, but others seem to find some value in the formatted documents. 

The session was not particularly conclusive in the end. There is general agreement that documentation is good, and a certain preference for documentation that is maintained as comments within the code itself. For the most part, the community will continue to muddle along, producing documentation as well as it can with the time that is available.
