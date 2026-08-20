---
title: A couple of graphical git front ends
url: https://lwn.net/Articles/140333/
date: "July 4, 2005"
category: "Development tools-Git"
author: 
---

The development of git is a classic free software success story, even at this early stage. When Linus Torvalds needed a new tool for managing source code revisions, he sat down for a week and bashed out something which was enough for other people to play with. It didn't do much, but it contained enough in the way of good ideas and functionality that a group of developers quickly gathered around it and started making it better. Git may still have a number of rough edges, but, to a great extent, it has taken BitKeeper's place in the development process. 

One of those rough edges has been the lack of graphical tools for working with the repository. Things are happening in this area as well, however, and there are now two tools available which warrant a look. These tools (being gitk and qgit) both function as patch browsers; neither, currently, can actually make any changes to a git repository. In other words, capabilities like three-way patch conflict resolution are still missing. But you have to start somewhere; either of these tools will be helpful for anybody who wishes to look at the path the kernel took to get to the current point. 

gitk is a Tk-based utility written by Paul Mackerras; the current release is [version 1.1](<http://lwn.net/Articles/138160/>). The build [![\[gitk screenshot\]](https://static.lwn.net/images/ns/kernel/gitk-sm.png)](<https://lwn.net/Articles/140350/>) process for gitk is about the easiest your editor has ever encountered; since it is written in Tcl, installation is simply a matter of putting the `gitk` script somewhere in your path. 

Running gitk yields a three-paned window. At the top is a reverse-time list of committed patches, along with a graphical trace showing which tree each patch was merged from. Tags in the repository are indicated by a cute little yellow tag in this pane. The bottom left shows the selected patch in a fairly gaudy, colorful form. The commit text and the patch itself are run together, just as they would appear in an emailed patch. On the lower right is a list of files touched by the current patch; clicking on the name of a file narrows the view to the corresponding portion of the patch. 

There is a simple "find" function which can search for patches whose description or author information match a given string; searching with regular expressions is possible. If you know the SHA tag for a given commit, you can type (or paste, presumably) it into a blank and view the corresponding patch. gitk also stores the SHA ID of the current commit in the X selection, allowing it to be pasted into git commands if need be. 

The alternative to gitk is [qgit](<http://sourceforge.net/projects/qgit>), a Qt-based application currently at version 0.6. The initial experience with qgit is a little rougher; the application uses [SCons](<http://www.scons.org/>) for building instead of make. Since most systems tend not to have SCons [![\[qgit screenshot\]](https://static.lwn.net/images/ns/kernel/qgit-sm.png)](<https://lwn.net/Articles/140351/>) installed, the qgit tarball includes a prebuilt version of SCons and a script to hack up the build file to use it. One thinks that, for a relatively simple application, it might have been easier to just toss in a makefile. 

The initial qgit window looks very similar to gitk. The lower left pane shows only the commit text, however; the actual patch is nowhere to be seen. A single click on the filenames on the right appears to do nothing; a double click will pop up a separate window with the full text of the file. It turns out that one has to double-click on the appropriate line in the top pane to get a separate window with the patch itself. Once the window is up, it will be updated with the body of any patch selected in the main window. The presentation of the patches is a bit nicer than in gitk; the use of color is a bit more restrained, and patches are shown in the `diff -up` format that developers are used to reading. The patch window, however, has the obnoxious feature that it is permanently on top of any other window on the screen, regardless of the user's desires. 

Nicely, in version 0.6, the qgit file browser window includes line-by-line annotation which makes it easy to figure out which commit modified a specific piece of code. This feature is enough to make one wish that the kernel.org git repository contained more than a few months of history. 

qgit also makes it possible to search for specific patches; entering a string in the blank and clicking "filter" will narrow the patch view to the patches containing the string. It is not obvious, but the way back to the full listing is to hit "filter" again. 

Once upon a time, your editor asked the BitKeeper folks how to determine which tree was the source of any particular changeset in the mainline. The answer that came back was that this information was not available - BitKeeper did not store it. So it is pleasing to see this information so readily in evidence in either git front end. We really do have a better handle on the development history than we did even a few months ago. 

Both tools can be a bit rough to use at times, and their features are limited. When one considers that, back at the beginning of March, none of this software existed at all, it is hard not to be impressed. There is a lot happening around git, and, chances are, we've just barely seen the beginning of it.
