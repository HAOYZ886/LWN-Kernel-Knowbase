---
title: COW Links
url: https://lwn.net/Articles/77972/
date: "March 29, 2004"
category: "COW links; Filesystems-COW links"
author: 
---

Free software hackers often find themselves cloning a large tree full of source files; with a duplicate tree, it is easy to see which files have been changed and to generate patch files. Creating such a tree can be easy as typing: 

```
cp -rl old-tree new-tree
```

This technique works well if you use a tool (emacs, say) which moves files aside before rewriting them. By moving the file, emacs breaks the link and leaves the original copy (in the old tree) unchanged. If, however, the tool rewrites the file in place (as vi tends to do), the file, as seen in both trees, will be changed. 

As a solution to this problem, Jorn Engel has been working on [a patch](<https://lwn.net/Articles/76616/>) which implements "cowlinks." The idea behind a COW (copy-on-write) link is that, if the file linked to is written to, a copy will be made (thus breaking the link) and the write will be performed on the copy. With this capability, somebody wishing to duplicate and modify a tree of files could use COW links; the duplicate files would share the same blocks on disk until one was modified. And it would all work regardless of the tool being used to perform the modifications. 

In fact, COW links could be used for any copy operations within the same filesystem. The result would be faster copies and, perhaps, substantial savings of disk space. 

The current cowlink patch does not actually implement this behavior, however. It implements a COW bit in the inode structure, but, rather than actually perform the copy, it simply fails any attempt to write a file with more than one link. User space is then expected to notice the error and do the right thing. This is not the long-term planned behavior; from a comment in the code: 

Yes, this breaks the kernel interface and is simply wrong. This is intended behaviour, so Linus will not merge the code before it is complete. Or will he? 

The full behavior has not yet been implemented because it requires some tricky filesystem-level programming. There is also the issue that the right behavior for COW links has not, yet, been worked out. One obvious implementation would have COW links behave just like regular, "hard" links, with the file being truly copied when the first write is done. With that approach, however, the file will change its inode number after the writing application has opened it. That is just the sort of anomalous, nonstandard behavior that can break applications in strange and unexpected places. 

An alternative would be for two COW-linked files to have separate inode numbers from the beginning, even though they share the same on-disk data. If COW links are implemented this way, no application will notice when the link is broken. What _will_ break, however, is any application which depends on inode numbers to detect identical files. Recursive diffs will be much slower, "du" will give wrong numbers, and tar could do the wrong thing. Fixing all of these applications would require the addition of a nonstandard system call and fixing the programs to use it. 

Linus has [made his opinion known](<https://lwn.net/Articles/77974/>): 

I think the correct thing to do is to just admit that cowlinks aren't POSIX, and instead see the inode number as a way to see whether the link has been broken or not. Ie just accept the inode number potentially changing. 

That opinion makes it likely that development will go in that direction, but, until the code shows up, nobody knows for sure.
