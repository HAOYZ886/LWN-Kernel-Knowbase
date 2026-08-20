---
title: A very quick guide to starting with git
url: https://lwn.net/Articles/132796/
date: "April 20, 2005"
category: "Development tools-Git; Git"
author: 
---

Linus has posted a git archive containing the 2.6.12-rc2 kernel source with a small series of patches. His current plan is to _not_ populate that repository with the full development history reclaimed from BitKeeper. Adding the history would massively bloat the size of the repository, and git currently lacks the tools to do anything interesting with that history anyway. So the repository starts with a clean slate and goes from there. 

If you want to experiment with the new setup, the steps are relatively simple. The first of which is to be sure that you are sufficiently interested to pull down a 120MB repository and play with bleeding-edge tools; in many cases, it might be better to wait a little longer. Should you choose to continue, the first step is to grab the latest git-pasky distribution, found at <http://pasky.or.cz/~pasky/dev/git/>. Untar it, and go through a series of steps like: 

```
make
        git pull pasky
        make
```

That will yield the current git, with Petr's added tools. Put said tools into your path, create a directory for the kernel tree, and run: 

```
git init rsync://kernel.org/pub/linux/kernel/people/torvalds/linux-2.6.git/
```

The command will appear to do nothing for quite some time; it will eventually pull down the entire repository and check out a copy. You'll now have a copy of the current Linus mainline tree. 

Typing "`git log`" will print out the checkin log messages in reverse chronological order. "`git pull`" will update the tree to the current mainline. Just typing "`git`" will yield a list of possible commands. The capability is there, at this point, to check in changes, merge changes from other trees, generate patches, etc. Enjoy, but expect things to continue to change in a hurry.
