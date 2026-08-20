---
title: More notes on reiser4
url: https://lwn.net/Articles/100148/
date: "September 1, 2004"
category: "Filesystems-Reiser4; Named streams"
author: 
---

The article on reiser4 which appeared here [last week](<https://lwn.net/Articles/98509/>) drew a number of comments. [One comment from Hans Reiser](<https://lwn.net/Articles/99878/>) took LWN to task for not having started with a kernel tarball which was created from a reiser4 filesystem to begin with. It seems that reiser4 is highly sensitive to the order in which files are created, and using the wrong order does not show the filesystem in its best light. 

Here is last week's table, with a new line for tests done starting with a reiser4-built tarball: 

Filesystem | Test  
---|---  
Untar | Build | Grep | find (name) | find (stat)  
ext3 | 55/24 | 1400/217 | 62/8 | 10.4/1.1 | 12.1/2.5  
reiser4 | 67/41 | 1583/386 | 78/12 | 12.5/1.3 | 15.2/4.0  
reiser4 (new) | 57/35 | 1445/393 | 58/9.9 | 8.4/1.3 | 11.1/4.0  
  
The results do show a significant difference in performance when the files are created in the right order - and the differences carry through all of the operations performed on the filesystem, not just the untar. In other words, the performance benefits of reiser4 are only fully available to those who manage to create their files in the right order. Future plans call for a "repacker" process to clean up after obnoxious users who insist on creating files in something other than the optimal order, but that tool is not yet available. (For what it's worth, restoring from the reiser4 tarball did not noticeably change the ext3 results). 

Last week, the discussion about reiser4 got off to a rather rough start. Even so, it evolved into a lengthy but reasonably constructive technical conversation touching on many of the issues raised by reiser4. 

At the top of the list is the general question of the expanded capabilities offered by this filesystem; these include transactions, the combined file/directory objects (and the general representation of metadata in the filesystem namespace), and more. The kernel developers are nervous about changes to filesystem semantics, and they are seriously nervous about creating these new semantics at the filesystem level. The general feeling is that any worthwhile enhancements offered by reiser4 should, instead, be implemented at the virtual filesystem (VFS) level, so that more filesystems could offer them. Some developers want things done that way from the start. If there is a consensus, however, it would be along the lines [laid out](<https://lwn.net/Articles/100150/>) by Andrew Morton: accept the new features in reiser4 for now (once the other problems are addressed) with the plan of shifting the worthwhile ones into the VFS layer. The reiser4 implementation would thus be seen as a sort of prototype which could be evolved into the true Linux version. 

Hans Reiser [doesn't like this idea](<https://lwn.net/Articles/100154/>): 

Look guys, in 1993 I anticipated the battle would be here, and I build the foundation for a defensive tower right at the spot MS and Apple are now maneuvering towards. Help me get the next level on the tower before they get here. It is one hell of a foundation, they won't be able to shake it, their trees are not as powerful. Don't move reiser4 into vfs, use reiser4 as the vfs. Don't write filesystems, write file plugins and disk format plugins and all the other kinds of plugins, and you won't be missing any expressive power that you really want.... 

Somehow, over the years, Hans has neglected to tell the developers that he was, in fact, planning to replace the entire VFS. That plan looks like a difficult sell, but reiser4 could become the platform that is used to shift the VFS in the directions he sees. 

Meanwhile, the reiser4 approach to metadata has attracted a fair amount of attention. Imagine you have a reiser4 partition holding a kernel tree; at the top of that tree is a file called `CREDITS`. It's an ordinary file, but it can be made to behave in extraordinary ways: 

> 

```
>     $ **tree CREDITS/metas**
>     CREDITS/metas
>     |-- bmap
>     |-- gid
>     |-- items
>     |-- key
>     |-- locality
>     |-- new
>     |-- nlink
>     |-- oid
>     |-- plugin
>     |   |-- compression
>     |   |-- crypto
>     |   |-- digest
>     |   |-- dir
>     |   |-- dir_item
>     |   |-- fibration
>     |   |-- file
>     |   |-- formatting
>     |   |-- hash
>     |   |-- perm
>     |   `-- sd
>     |-- pseudo
>     |-- readdir
>     |-- rwx
>     |-- size
>     `-- uid
>      
>     1 directory, 24 files
>
```

You can also type "`cd CREDITS; cat .`" to view the file. (One must set execute permission on the file before any of this works). 

What appears to be a plain file also looks like a directory containing a number of other files. Most of these files contain information normally obtained with the `stat()` system call: `uid` is the owner, `size` is the length in bytes, `rwx` is the permissions mask, etc. Some of the others (`bmap`, `items`, `oid`) provide a window into how the file is represented inside the filesystem. This is all part of Hans Reiser's vision of moving everything into the namespace; rather than using a separate system call to learn about a file's metadata, just access the right pseudo file. 

One branch of the discussion took issue with the "`metas`" name. Using reiser4 means that you cannot have any file named `metas` anywhere within the filesystem. Some people would like to change the name; ideas like `..metas`, `...`, and `@` have been tossed around, but Hans seems uninclined to change things. 

Another branch, led by Al Viro, worries about the locking considerations of this whole scheme. Linux, like most Unix systems, has never allowed hard links to directories for a number of reasons; one of those is locking. Those interested in the details can see [this rather dense explanation](<https://lwn.net/Articles/100183/>) from Al, or [a translation](<https://lwn.net/Articles/100184/>) by Linus to something resembling technical English. Linus's example is essentially this: imagine you have a directory "`a`" containing two subdirectories `dir1` and `dir2`. You also have "`b`", which is simply a link to `a`. Imagine that two processes simultaneously attempt these commands: 

Process 1|     | Process 2  
---|---|---  
`mv a/dir1 a/dir2/newdir`|  | `mv b/dir2 b/dir1/newdir`  
  
Both commands cannot succeed, or you will have just tied your filesystem into a knot. So some sort of locking is required to serialize the above actions. Doing that kind of locking is very hard when there are multiple paths into the same directory; it is an invitation to deadlocks. The problem could be fixed by putting a monster lock around the entire filesystem, but the performance cost would be prohibitive. The usual approach has been to simply disallow this form of aliasing on directory names, and thus avoid the problem altogether. 

In the reiser4 world, all files are also directories. So hard links to files become hard links to directories, and all of these deadlock issues come to the foreground. The concerns expressed by the kernel developers - which appear to be legitimate - is that the reiser4 team has not thought about these issues, and there is no plan to solve the problem. Wiring the right sort of mutual exclusion deeply into a filesystem is a hard thing to do as an afterthought. But something will have to be done; Al Viro has made it clear that he will oppose merging reiser4 until the issue has been addressed, and it is highly unlikely that it would go in over his objections ([Linus](<https://lwn.net/Articles/100271/>): ""This means that if Al Viro asks about locking and aliasing issues, you don't ignore it, you ask 'how high?'"") 

One way of dealing with the locking issues (and various other bits of confusion) would be to drop the "files as directories" idea and create a namespace boundary there. Files could still have attributes, but an application which wished to access them would use a separate system call to do so. The `openat()` interface, which is [how Solaris solves the problem](<http://docs.sun.com/db/doc/816-0220/6m6nkorp9?a=view>), seems like the favored approach. Pushing attributes into their own namespace breaks the "everything in one namespace" idea which is so fundamental to reiser4, but it would offer compatibility with Solaris and make many of the implementation issues easier to deal with. On the other hand, applications would have to be fixed to use `openat()` (or be run with `runat`). 

Another contingent sees the reiser4 files-as-directories scheme as the way to implement multi-stream files. Linux is one of the few modern operating systems without this concept. The Samba developers, in particular, would love to see a multi-stream implementation, since they have to export a multi-stream interface to the rest of the world. There are obvious simple applications of multi-stream files, such as attaching icons to things. Some people are ready to use the reiser4 plugin mechanism and go nuts, however; they would like to add streams which present compressed views of files, automatically produce and unpack archive files, etc. Linus [draws the line](<https://lwn.net/Articles/100185/>) at that sort of stuff, though: 

Which means that normally we really don't _want_ named streams. In 99% of all cases we can use equally good - and _much_ simpler - tool-based solutions. 

Which means that the only _real_ technical issue for supporting named streams really ends up being things like samba, which want named streams just because the work they do fundamentally is about them, for externally dictated reasons. Doing named streams for any other reason is likely just being stupid. 

Once you do decide that you have to do named streams, you might then decide to use them for convenient things like icons. But it should very much be a secondary issue at that point. 

Yet another concern has to do with how user space will work with this representation of file metadata. Backup programs have no idea of how to save the metadata; `cp` will not copy it, etc. Fixing user space is certainly an issue. The fact is, however, that, if reiser4 or the VFS of the future changes our idea of how a file behaves, the applications will be modified to deal with the new way of doing things. Meanwhile, it has been pointed out that reiser4-style metadata is probably easier for applications to work with than the current extended attribute interface, which is also not understood by most applications. 

The discussion looks likely to continue for some time. Regardless of the outcome, Hans Reiser will certainly have accomplished one of his goals: he has gotten the wider community to start to really think about our filesystems and how they affect our systems and how we use them.
