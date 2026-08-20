---
title: Rich access control lists
url: https://lwn.net/Articles/661357/
date: "October 20, 2015"
category: Access control lists; RichACLs
author: "By Jonathan Corbet October 20, 2015"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
October 20, 2015

Linux has had support for POSIX access control lists (ACLs) since the [2.5.46](<https://lwn.net/Articles/14568/>) development kernel was released in 2002 — despite the fact that POSIX has never formally adopted the ACL specification. Over time, POSIX ACLs have been superseded by other ACL mechanisms, notably the ACL scheme adopted with the NFSv4 protocol. Linux support for NFSv4 ACLs is minimal, though; there is no support for them at all in the virtual filesystem layer. The Linux NFS server supports NFSv4 ACLs by mapping them, as closely as possible, to POSIX ACLs. Chances are, that will end with the merging of Andreas Gruenbacher's [RichACLs patch set](<https://lwn.net/Articles/661078/>) for the 4.4 kernel. 

The mode ("permissions") bits attached to every file and directory on a Linux system describe how that object may be accessed by its owner, by members of the file's group, and by the world as a whole. Each class of user has three bits regulating write, read, and execute access. For many uses, that is all the control a system administrator needs, but there are times where finer-grained access control is useful. That is where ACLs come in; they allow the specification of access-control policies that don't fit into the nine mode bits. There are different types of ACLs, reflecting their evolution over time. 

#### POSIX ACLs

POSIX ACLs are clearly designed to fit in with traditional Unix-style permissions. They start by implementing the mode bits as a set of implicit ACL entries (ACEs), so a file with permissions like: 

```
$ ls -l foo
        -rw-rw-r--  1 linus penguins  804 Oct 18 09:40 foo
```

Has a set of implicit ACEs that looks like: 

```
$ getfacl foo
        user::rw-
        group::rw-
        other::r--
```

The `user` and `group` ACEs that contain empty name fields ("`::`") apply to the owner and group of the file itself. The administrator can add other `user` or `group` ACEs to give additional permissions to named users and groups. The actual access control is implemented in a way similar to how the mode bits are handled. If one of the `user` entries matches, the associated permissions are applied. Otherwise, if one of the `group` entries matches, that entry is used; failing that, the `other` permissions are applied. 

There is one little twist: the traditional mode bits still apply as well. When ACLs are in use, the mode bits define the _maximum_ permissions that may be allowed. In other words, ACLs cannot grant permissions that would not be allowed by the mode bits. The reason for this behavior is to avoid unpleasant surprises for applications (and users) that do not understand ACLs. So a file with mode `0640` (`rw-r-----`) would _not_ allow group-write access, even if it had an ACE like: 

```
group::rw-
```

If a particular process matches a named ACE (by either user or group name), that process is in the group class and is regulated by the group mode bits on the file. The owning group itself can be given fewer permissions than the mode bits would otherwise allow. See [this article](<http://users.suse.com/~agruen/acl/linux-acls/online/>) for a detailed description of how it all works. 

#### NFSv4 ACLs

When the NFS community took on the task of defining an ACL mechanism for the NFS protocol, they chose not to start with POSIX ACLs; instead, they started with something that looks a lot more like Windows ACLs. The result is a rather more expressive and flexible ACL mechanism. With one obscure exception, all POSIX ACLs can be mapped onto NFSv4 ACLs, but the reverse is not true. 

NFSv4 ACLs do away with the hardwired evaluation order used by POSIX ACLs. Instead, ACEs are evaluated in the order they are defined. Thus, for example, a group ACE can override an owner ACE if the group ACE appears first in the list. NFSv4 ACEs can explicitly deny access to a class of users. Permissions bits are also additive in NFSv4 ACLs. As an example of this, consider a file with these ACLs: 

```
group1:READ_DATA:ALLOW
        group2:WRITE_DATA:ALLOW
```

These ACEs allow read access to members of `group1` and write access to members of `group2`. If a process that is a member of _both_ groups attempts to open this file for read-write access, the operation will succeed. When POSIX ACLs are in use, instead, the requested permissions must all be allowed by a single ACE. 

NFSv4 ACLs have a lot more permissions that can be granted and denied. Along with "read data," "write data," and "execute," there are independent permissions bits allowing append-only access, deleting the file (regardless of permissions in the containing directory), deleting any file contained within a directory, reading a file's metadata, writing the metadata (changing the timestamps, essentially), taking ownership of the file, and reading and writing a file's ACLs. 

There is a set of bits controlling how ACLs are inherited from their containing directories. ACEs on directories can be marked as being inheritable by files within those directory; there is also a bit to mark an ACE that should only propagate a single level down the hierarchy. When a file is created within the directory, it will be given the ACLs that are marked as being inheritable in its parent directory. This behavior conflicts with POSIX, which requires that any "supplemental security mechanisms" be disabled for new files. 

ACLs can have an "automatic inheritance" flag set. When an ACL change is made to a directory, that change will be propagated to any files or directories underneath that have automatic inheritance enabled — unless the "protected" flag is also set. Setting the "protected" flag happens whenever the ACL or mode of the file have been set explicitly; that keeps inheritance from overriding permissions that have been intentionally set to something else. The interesting twist here is that there is no way in Linux for user space to create a file without explicitly setting its mode, so the "protected" bit will always be set on new files and automatic inheritance simply won't work. NFS does have a way to create files without specifying the permissions to use, though, so automatic inheritance will work in that case. 

NFSv4 ACLs also differ in how permissions are applied to the world as a whole. The "other" class is called "`EVERYONE@`", and it means truly everyone. In normal POSIX semantics, if a process is in the "user" or "group" class, the "other" permissions will not even be consulted; that allows, [for example](<https://lwn.net/Articles/621612/>), a specific group to be blocked from a file that is otherwise world accessible. If a file is made available to everyone in an NFSv4 ACL, though, it is truly available to everyone unless a specific "deny" ACE earlier in the list says otherwise. 

#### RichACLs

The RichACLs work tries to square NFSv4 ACLs with normal POSIX expectations. To do so, it applies the mode bits in the same way that POSIX ACLs do — the mode specifies the maximum access that is allowed. Since there are far more access types in NFSv4 ACLs than there are mode bits, a certain amount of mapping must be done. So, for example, if the mode denies write access, that will be translated to a denial of related capabilities like "create file," "append data," "delete child," and more. 

The actual relationship between the ACEs and the mode is handled via a set of three masks, corresponding to owner, group, and other access. If a file's mode is set to deny group-write access, for example, the corresponding bits will be cleared from the group mask in the ACL. Thereafter, no ACE will be able to grant write access to a group member. The original ACEs are preserved when the mode is changed, though; that means that any additional access rights will be returned if the mode is made more permissive again. The masks can be manipulated directly, giving more fine-grained control over the maximum access allowed to each class; tweaking the masks can cause the file's mode to be adjusted to match. 

There are some interesting complications in the relationship between the ACEs, the masks, and the actual file mode. Consider an example (from [this document](<http://www.bestbits.at/richacl/draft-gruenbacher-nfsv4-acls-in-posix-00.html>)) where a file has this ACL: 

```
OWNER@:READ_DATA::ALLOW
        EVERYONE@:READ_DATA/WRITE_DATA::ALLOW
```

This ACL gives both read and write access to the owner. If, however, the file's mode is set to `0640`, the mask for `EVERYONE@` will be cleared, denying owner-write access even though there is nothing in the permissions that requires that. Fixing this issue requires a special pass through the ACL to grant the `EVERYONE@` flags to other classes where the mode allows it. 

A similar problem comes up when an `EVERYONE@` ACE grants access that is denied by the owner or group mode bits. Handling this case requires inserting explicit DENY ACEs for `OWNER@` (or `GROUP@`) ahead of the `EVERYONE@` ACE. 

The RichACLs patch set implements all of this and more. See [this page](<http://www.bestbits.at/richacl/>) and the [richacl.7 man page](<http://www.bestbits.at/richacl/man/richacl.7.txt>) for more details. As of this writing, though, there is still an open question: how to handle RichACL support in Linux filesystems. 

At the implementation level that question is easily answered; RichACLs are stored as extended attributes, just like POSIX ACLs or SELinux labels. The problem is one of backward compatibility: what happens when a filesystem containing RichACLs is mounted by a kernel that does not implement them? Older kernels will not corrupt the filesystem (or the ACLs) in this case, but neither will they honor the ACLs. That can result in access being granted that would have been denied by an ACL; it also means that ACL inheritance will not be applied to new files. 

To prevent such problems, Andreas [requested](<https://lwn.net/Articles/661364/>) that a feature flag be added to the ext4 filesystem; that flag would prevent the filesystem from being mounted by kernels that do not implement RichACLs. There was some discussion about whether this made sense; ext4 maintainer Ted Ts'o [felt](<https://lwn.net/Articles/661365/>) that the feature flags were there to mark metadata changes that the `e2fsck` utility needed to know about to avoid corrupting the filesystem. RichACLs do not apply, since filesystems don't pay attention to the contents of extended attributes. 

Over the course of the conversation, though, a consensus seemed to form around the idea that the use of RichACLs is a fundamental filesystem feature. So it appears that once they are enabled for an ext4 filesystem (either at creation time, or via `tune2fs`), that filesystem will be marked as being incompatible with kernels that don't implement RichACLs. Something similar will likely be done for XFS. 

If things go as planned, this work will be mainlined during the 4.4 merge window. At that point, NFS servers should be able to implement the full semantics of NFSv4 ACLs; the feature should also be of use to people running Samba servers. This patch set, the culmination of several years' work, should provide a useful capability to server administrators who need fully supported access control lists on Linux.
