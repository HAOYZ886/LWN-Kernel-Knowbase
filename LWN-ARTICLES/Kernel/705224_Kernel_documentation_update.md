---
title: Kernel documentation update
url: https://lwn.net/Articles/705224/
date: "November 2, 2016"
category: Documentation
author: "By Jake Edge November 2, 2016 2016 Kernel Summit"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
November 2, 2016

* * *

[2016 Kernel Summit](<https://lwn.net/Articles/KernelSummit2016/>)

The kernel's documentation "subsystem" has undergone some changes over the past few releases as we [reported on](<https://lwn.net/Articles/704613/>) in late October. The author of that report and the kernel's documentation maintainer, Jonathan Corbet, gave a presentation at this year's Kernel Summit to describe some of those changes. He was joined by Mauro Carvalho Chehab, who has done much of the work (along with Daniel Vetter, Jani Nikula, and Markus Heiser) to make it all happen. 

[ ![\[Jonathan Corbet\]](https://static.lwn.net/images/2016/ks-corbet-sm.jpg) ](<https://lwn.net/Articles/705305/>)

Corbet started by noting the [4.8-rc1 release announcement](<https://lwn.net/Articles/696633/>), where Linus Torvalds highlighted that ""over 20% of the patch is documentation updates, due to conversion of the drm and media documentation from docbook to the Sphinx doc format"". Those changes were unusual in that documentation changes have never been anywhere near that large in previous merge windows. 

Corbet set out on the Sphinx transition with several goals. The first was to eliminate the hand-rolled DocBook-based toolchain that was being used to generate the documentation. At a Kernel Summit a few years ago, he asked kernel developers how many had gotten the toolchain set up and less than half indicated that they had. In the end, it is simply "the wrong way to go"; the kernel project should not be developing its own tools for creating documentation, it should use something that is developed and maintained by others. 

Another goal was to have integrated documentation with nice output formatted in multiple ways. But, he wanted to be able to do that without a complicated markup language and a bunch of toolchain dependencies. Beyond that, he wanted to clean up the `Documentation` directory so that it "doesn't look like my daughter's bedroom", he said, complete with a photo of said messy bedroom. All of that will make it easier for developers to write documentation, which should, in theory, lead to better documentation. 

So, starting in the 4.7 cycle, the documentation began being switched to use the [Sphinx documentation generator](<http://www.sphinx-doc.org/en/1.4.8/>), which uses the [reStructuredText](<http://docutils.sourceforge.net/rst.html>) markup language. There are, he said, LWN articles about the [history of the change](<https://lwn.net/Articles/692704/>) and [how it all works](<https://lwn.net/Articles/692705/>). In addition, the kernel documentation now has a [Linux Kernel Documentation book](<https://www.kernel.org/doc/html/latest/kernel-documentation.html>) that describes how to build (and write) documentation for the kernel. 

#### Open questions

There are, of course, some open questions. The organization of the documentation tree leaves a lot to be desired. It used to have around 300 files in the top-level directory, but he has slowly been moving things around. One move that he has been nervous about is the `SubmittingPatches` file, which is being incorporated into the [development-process book](<http://static.lwn.net/kerneldoc/process/index.html>). Chehab has submitted a patch to move the file and leave a three-line file pointing to the new location in its place, but Corbet is worried about dangling references to the file. He asked if there were objections to making that move. 

At that point, Torvalds said with a grin: "No one in this room has ever read anything in the `Documentation` directory." He said it was really up to the users of the kernel and its documentation to decide if the move made sense. There were murmurs of disagreement in the room and Darren Hart said: "I do use it, read it, and cite it by section." He said that he liked what had been done so far, especially that he could now cite sections by URL. 

That last piece is thanks to kernel.org maintainer Konstantin Ryabitsev, who built the documentation from the tree and [put it up at kernel.org](<https://www.kernel.org/doc/html/latest/index.html>), Corbet said. Furthermore, a look at mailing list postings shows that the documentation is cited rather frequently. "So they may not read it, but they tell everyone else to read it", he said. Based on the reaction in the room, it appears that no one is "too upset" with moving `SubmittingPatches`, so he will leave that patch in for 4.10. The goal is to eventually have a top-level `Documentation` directory that looks like all of the others in the kernel tree. 

Olof Johansson asked about having stub files pointing to the proper place, as was done for `SubmittingPatches`. Corbet replied that he has done that for some of the more important files, but not for every one. David Howells also cautioned against moving `memory-barriers.txt`. Corbet said that when he had broached the subject with Paul McKenney, he was told to "keep away", so he plans to work with McKenney on that down the road. 

For something perhaps a bit more controversial, Corbet noted that there is only one directory in the top-level kernel directory that is capitalized: `Documentation`. Since part of the reorganization will be adding more subdirectories, thus lengthening the path names for files of interest—in addition to an already-long top-level name—it has led some to ask that he consider renaming the directory to `doc` . 

That immediately led to discussion of tab-key-based auto-completion, as well as bikeshedding over a new name. But, as was also pointed out, those names are often used in places (e.g. email) where auto-completion does not work. H. Peter Anvin noted that files like `README` are capitalized in part to help newbies, who will often be attracted to those files because their names stand out. 

After some more discussion, it was suggested that Corbet call for a vote, which he did. Roughly half of those assembled voted against, while about the same voted for the change. That made it obvious there was "no clear consensus" on the question, so things would stay the way they are. Shuah Khan was glad to hear that; she voted against changing the name because of the large number of blog posts and other types of information that she and others have written that would suddenly become outdated. 

#### Adding complications

Moving to another topic, Corbet said he had set out to get to something simpler and the community had accomplished that, but now things have started to get more complicated again. A change that was made for 4.9 meant that LaTeX is required to build the HTML version of the documentation. He will be pushing to get that particular problem fixed. 

There are number of files that some want to pre-process to get them into the Sphinx format. There was a request to add a Sphinx directive that would run an arbitrary shell command as part of the documentation build process, but he rejected that particular mechanism. It has also been suggested that the `MAINTAINERS` file be processed into the Sphinx format. 

[ ![\[Mauro Carvalho Chehab\]](https://static.lwn.net/images/2016/ks-chehab-sm.jpg) ](<https://lwn.net/Articles/705305/>)

Since the media subsystem is where some of the push for pre-processing is coming from, he asked Chehab to explain what he would like to do there. Chehab has a patch that takes the ABI files from the media subsystem to convert them into the Sphinx format, which allows creating documentation that is sorted and arranged in various ways. That is useful for distributions and others, he said; "it adds value" to the documentation. 

For the `MAINTAINERS` file, it would be nice if interested users could find out where they can get the latest development tree for a subsystem, which could be added into the information already there. It makes it easier for users if it is part of the documentation and it "comes almost for free", he said. 

Corbet said that a decision will need to be made about how much more complicated the documentation toolchain should be allowed to get. Nikula has suggested that any changes made for the kernel should be upstreamed into Sphinx. While that is a nice idea, Corbet said, some of the changes are pretty kernel-specific so it may be hard to convince the Sphinx developers to accept them. 

Another area of disagreement is about what to do with old and obsolete documentation, much of which has not even seen typo fixes in the Git era. For example, some instructions from Larry McVoy in 1996 on how to manually bisect a problem in the 1.3 kernel seem like they are past their prime. We don't keep old code around, Corbet said, so we should do the same with documentation. 

Torvalds wryly noted that pull requests that remove lines from the kernel get high priority. But he had a different complaint as well: "Can we get rid of PDF in the kernel source?" It is, he said, "binary crap" and those files are simply PDF versions of the SVG files sitting right next to them. 

Chehab said that the media subsystem needed some PDF files for the DocBook version of its documentation. Those may not be needed for Sphinx, he said. There are roughly ten PDF files that showed up recently, Torvalds said. Those files are not editable and have no reason to be in the kernel source. 

Image files are similar, Torvalds said after a question from Corbet. A binary file that no one can edit should not be in the tree, Torvalds said. He suggested putting them on a web site, but that there is a reason the kernel tree is called a _source_ tree. It was agreed that solutions could be found to have images with the documentation without requiring binary images in the kernel tree. 

Rafael Wysocki asked that Corbet consult with him before moving any files in the power management part of the documentation tree. That is standard operating procedure, Corbet said. He will let the appropriate maintainer know what he is planning to do and won't do it over the objection of that maintainer. 

Another request came from Hannes Reinecke, who would like to see the return values of kernel functions get added into the documentation. Right now, some free-form text could be added to the kerneldoc comments associated with the function, Corbet said, and something more structured could be worked out later. But, in order to get a full list of the return values, the entire set of kernel functions needs to be annotated, David Woodhouse said, so that return values from functions that are called can be incorporated into the list. But it was suggested that even just annotating the leaf functions (those that call no others) would be a good place to start. At that point, things kind of wound down; Corbet and Chehab left the stage in favor of Batman.
