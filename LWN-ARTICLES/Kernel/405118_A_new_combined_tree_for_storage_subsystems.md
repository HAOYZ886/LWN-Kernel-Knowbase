---
title: A new combined tree for storage subsystems
url: https://lwn.net/Articles/405118/
date: "September 15, 2010"
category: Development model
author: "By Jake Edge September 15, 2010"
---

By **Jake Edge**  
September 15, 2010

One of the outcomes from this year's Linux Storage and Filesystem Summit was a plan to create a combined tree to help ease the process of integrating changes to various storage subsystems. At the summit, James Bottomley ["volunteered" himself](<http://lwn.net/Articles/399148/#lightning>) to put the tree together, and that came to fruition with his [announcement](<http://lwn.net/Articles/404484/>) of the tree on September 10. Paralleling the discussion at the summit, there is still the lingering belief that more than just an automatically generated tree may be needed. 

The tree currently collects patches from several subsystem trees, scsi, libata, and block, along with patches from the dm quilt repository. It is being automatically pulled and built nightly, much like linux-next. It will also be rebased daily against the mainline which will make it somewhat harder for kernel hackers to use--also like linux-next. Because of that, Dave Chinner didn't really [see](<https://lwn.net/Articles/405137/>) the storage-tree as being all that useful: ""I really don't see a tree like this getting wide use - if I enjoyed the pain of rebasing against throw-away merge trees every day, then I'd already be using linux-next."" 

Bottomley [acknowledged](<https://lwn.net/Articles/405139/>) that complaint, noting that using linux-next had been suggested at the summit, but pointed out that the storage-tree is a much smaller target than linux-next: ""The diffs to mainline in the current storage tree are still under a megabyte."" Bottomley also noted that the summit participants were a bit skeptical that a tree without a "storage maintainer" to oversee it (_a la_ Dave Miller's networking tree) might not prove to solve the problem, which was one of Chinner's concerns as well. 

But there are political considerations too. ""Unlike net, storage has never had a single maintainer, so it's a bit more political than just doing that by fiat"", Bottomley said. Chinner was of the [opinion](<https://lwn.net/Articles/405143/>) that the summit is the obvious place to have made a decision to appoint a storage maintainer, even if all of the current maintainers of the storage subsystems were not present. But its clear that those who were present wanted to move slowly, as Bottomley [described](<https://lwn.net/Articles/405144/>): 

This sort of thing doesn't get decided by fiat. If you can't get all of the relevant parties to agree, you have to demonstrate the need. So doing a rollup tree to test how much of the problem is solvable that way seems like a reasonable first step. 

The tree is available at `git://git.kernel.org/pub/scm/linux/kernel/git/jejb/storage-tree`. The nightly diffs from the mainline and log of the pull script are [available](<http://www.kernel.org/pub/linux/kernel/people/jejb/storage-tree/>) as well. It is likely to take a bit of time to see if the storage-tree solves the problem with integration of cross-storage-subsystem changes, but it does provide a good starting point.
