---
title: The guts of git
url: https://lwn.net/Articles/131657/
date: "April 12, 2005"
category: "Development tools-Git; Git"
author: 
---

Now that BitKeeper is no more, how will the kernel development process function? In the short term, the answer is "painfully." The rest of the 2.6.12 process looks like the good old days: patches emailed to Linus, who will apply them (hopefully) and occasionally release a snapshot tree. That mode might work for the short term, since only bug fixes should be merged before 2.6.12 comes out, but nobody wants to try to run the process that way for any period of time. The kernel team needs much better patch and workflow support if it is going to sustain a reasonable development pace. So a replacement for BitKeeper will have to come from somewhere. 

For a while, the leading contender appeared to be [monotone](<http://www.venge.net/monotone/>), which supports the distributed development model used with the kernel. There are some issues with monotone, however, with performance being at the top of the list: monotone simply does not scale to a project as large as the kernel. So Linus has, in classic form, gone off to create something of his own. The first version of the tool called "git" was [announced](<https://lwn.net/Articles/131312/>) on April 7\. Since then, the tool has progressed rapidly. It is, however, a little difficult to understand from the documentation which is available at this point. Here's an attempt to clarify things. 

Git is _not_ a source code management (SCM) system. It is, instead, a set of low-level utilities (Linus compares it to a special-purpose filesystem) which can be used to construct an SCM system. Much of the higher-level work is yet to be done, so the interface that most developers will work with remains unclear. 

At the lower levels, Git implements two data structures: an object database, and a directory cache. The **object database** can contain three types of objects: 

  * **Blobs** are simply chunks of binary data - they are the contents of files. One blob exists in the object database for every revision of every file that git knows about. There is no direct connection between a blob and the name (or location) of the file which contains that blob. If a file is renamed, its blob in the object database remains unchanged. 

  * **Trees** are a collection of blobs, along with their file names and permissions. A tree object describes the state of a directory hierarchy at a particular given time. 

  * **Commits** (or "changesets") mark points in the history of a tree; they contain a log message, a tree object, and pointers to one or more "parent" commits (the first commit will have no parent). 

The object database relies heavily on SHA hashes to function. When an object is to be added to the database, it is hashed, and the resulting checksum (in its ASCII representation) is used as its name in the database (almost - the first two bytes of the checksum are used to spread the files across a set of directories for efficiency). Some developers have [expressed concerns](<https://lwn.net/Articles/131752/>) about hash collisions, but that possibility does not seem to worry the majority. The object itself is compressed before being checksummed and stored. 

It's worth repeating that git stores every revision of an object separately in the database, addressed by the SHA checksum of its contents. There is no obvious connection between two versions of a file; that connection is made by following the commit objects and looking at what objects were contained in the relevant trees. Git might thus be expected to consume a fair amount of disk space; unlike many source code management systems, it stores whole files, rather than the differences between revisions. It is, however, quite fast, and disk space is considered to be cheap. 

The **directory cache** is a single, binary file containing a tree object; it captures the state of the directory tree at a given time. The state as seen by the cache might not match the actual directory's contents; it could differ as a result of local changes, or of a "pull" of a repository from elsewhere. 

If a developer wishes to create a repository from scratch, the first step is to run `init-db` in the top level of the source tree. People running PostgreSQL want to be sure not to omit the hyphen, or they may not get the results they were hoping for. `init-db` will create the directory cache file (`.dircache/index`); it will also, by default, create the object database in `.dircache/objects`. It is possible for the object database to be elsewhere, however, and possibly shared among users. The object database will initially be empty. 

Source files can be added with the `update-cache` program. `update-cache --add` will add blobs to the object database for new files and create new blobs (leaving the old ones in place) for any files which have changed. This command will also update the directory cache with entries associating the current files' blobs with their current names, locations, and permissions. 

What `update-cache` will _not_ do is capture the state of the tree in any permanent way. That task is done by `write-tree`, which will generate a new tree object from the current directory cache and enter that object into the database. `write-tree` writes the SHA checksum associated with the new tree object to its standard output; the user is well-advised to capture that checksum, or the newly-created tree will be hard to access in the future. 

The usual thing to do with a new tree object will be to bind it into a commit object; that is done with the `commit-tree` command. `commit-tree` takes a tree ID (the output from `write-tree`) and a set of parent commits, combines them with the changelog entry, and stores the whole thing as a commit object. That object, in essence, becomes the head of the current version of the source tree. Since each commit points to its parents, the entire commit history of the tree can be traversed by starting at the head. Just don't lose the SHA checksum for the last commit. Since each commit contains a tree object, the state of the source tree at commit time can be reconstructed at any point. 

The directory cache can be set to a given version of the tree by using `read-tree`; this operation reads a tree object from the object database and stores it in the directory cache, but does not actually change any files outside of the cache. From there, `checkout-cache` can be used make the actual source tree look like the cached tree object. The `show-diff` tool prints the differences between the directory cache and what's actually in the directory tree currently. There is also a `diff-tree` tool which can generate the differences between any two trees. 

An early example of what can be done with these tools can be had by playing with the [git-pasky](<https://lwn.net/Articles/131560/>) distribution by Petr Baudis. Petr has layered a set of scripts over the git tools to create something resembling a source management system. The git-pasky distribution itself is available as a network repository; running "`git pull`" will update to the current version. 

A "pull" operation, as implemented in git-pasky, performs these steps: 

  * The current "head" commit for the local repository is found; git-pasky keeps the SHA checksum for the current commit in `.dircache/HEAD`. 

  * The current head is obtained from the remote repository (using `rsync`) and compared with the local head. If the two are the same, no changes have been made and the job is done. 

  * The remote object database is downloaded, again with `rsync`. This operation will add any new objects to the database. 

  * Using `diff-tree`, a patch from the previous (local) version to the current (remote) version is generated. That patch is then applied to the current directory's contents. The patch technique is used to help preserve, if possible, any local changes to the files. 

  * A call to `read-tree` updates the directory cache to match the current revision as obtained from the remote repository. 

Petr's version of git adds a number of other features as well. It is a far cry from a full-blown source code management system, since it lacks little details like release tagging, merging, graphical interfaces, etc. A beginning structure is beginning to emerge, however. 

When this work was begun, it was seen as a sort of insurance policy to be used until a real source management system could be found. There is a good chance, however, that git will evolve into something with staying power. It provides the needed low-level functionality in a reasonably simple way, and it is blindingly fast. Linus [places a premium on speed](<https://lwn.net/Articles/131669/>): 

If it takes half a minute to apply a patch and remember the changeset boundary etc (and quite frankly, that's _fast_ for most SCM's around for a project the size of Linux), then a series of 250 emails (which is not unheard of at all when I sync with Andrew, for example) takes two hours. 

As if on cue, Andrew [announced](<https://lwn.net/Articles/131754/>) a set of 198 patches to be merged for 2.6.12: 

This is the first live test of Linus's git-importing ability. I'm about to disappear for 1.5 weeks - hope we'll still have a kernel left when I get back. 

If this test (and the ones that come after) goes well, and the resulting system evolves to where it meets Linus's needs, he may be unlikely to switch to yet another system in the future. So git is worth watching; it could develop into a powerful system in a hurry.
