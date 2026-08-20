---
title: "Pathname-based hooks for SELinux?"
url: https://lwn.net/Articles/419161/
date: "December 8, 2010"
category: "SELinux; Security-Security modules"
author: "By Jake Edge December 8, 2010"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jake Edge**  
December 8, 2010

A [patch](<https://lwn.net/Articles/418849/>) that would add the last path component as a parameter to the Linux security module (LSM) hooks for inode creation raised a few eyebrows. It looked to be an attempt to add pathname-based hooks for SELinux--after many SELinux developers took strong stands against those kinds of hooks when they were proposed for AppArmor and, later, TOMOYO. But, this change would not add pathname-based access controls to SELinux, and would, instead, allow it to make decisions about the label it applies to a new inode based on the filename being created. Still, there are questions about whether this is just an _ad hoc_ change to the LSM API for SELinux, and whether there are other hooks that might benefit from similar treatment. 

The patches, which were proposed by Eric Paris on the linux-security-module mailing list, are fairly straightforward. The first simply adds a `struct qstr` pointer to the `inode_init_security()` hook and changes all the calls to it that are made, mostly in various filesystems. A `qstr` is a "quick string" object from the directory entry cache, which contains the filename and some additional information (length and hash). The other [patch](<https://lwn.net/Articles/419215/>) in the set changes SELinux so that it can use that information in its policies: 

Currently SELinux has rules which label new objects according to 3 criteria. The label of the process creating the object, the label of the parent directory, and the type of object (reg, dir, char, block, etc.) This patch adds a 4th criteria, the dentry name, thus we can distinguish between creating a file in an etc_t directory called shadow and one called motd. 

There is no file globbing, regex parsing, or anything mystical. Either the policy exactly (strcmp) matches the dentry name of the object or it doesn't. This patch has no changes from today if policy does not implement the new rules. 

But the inclusion of path information was enough to get a [rise](<https://lwn.net/Articles/419216/>) out of Casey Schaufler: ""I see. Pathname based controls. In SELinux."". He went on to note that AppArmor and TOMOYO had made similar arguments to Paris's and that there are already pathname-based hooks that were added to support those two solutions. But Paris is quick to [point out](<https://lwn.net/Articles/419217/>) that he is _not_ implementing pathname-based access controls (which is what AppArmor and TOMOYO implement), but is only adding additional information for decisions about labeling new filesystem objects: 

The intention is to remove some particularly gross userspace hacks related to new object labeling (read udev/restorecond/anything to do with /var/run, etc). It simplifies userspace, removes numerous races, and does so with no reduction in security (and theoretically the possibility of a more secure system) 

Schaufler [does not completely buy that argument](<https://lwn.net/Articles/419218/>) because of the way labels are typically maintained in an SELinux system, i.e. using user-space utilities like `restorecond` that _are_ pathname-based: ""Yes, the kernel component of SELinux relies strictly on the labels, but the reality is that SELinux is heavily dependent on the user space component to maintain the proper labels on files so that the specified policy is rational."" Stephen Smalley agreed with that to some extent, but [tries to clarify](<https://lwn.net/Articles/419219/>) the role of pathnames in SELinux: 

That fact that we are already using the parent directory context as an input in computing the security context of a new files means that our file labeling logic is already "path-based" in a certain sense. It isn't solely path-based (either before or after this change), but it is already taking into account the placement of the file when it is created. This just refines the granularity at which we can make such decisions. 

Smalley also explains more about the kinds of race conditions that the patch is trying to avoid: 

restorecond and udev relabeling of kernel-created dev nodes are inherently racy - the file is not created in the desired security context initially, and must be relabeled by some userspace component that notices that the file has been created. Kernel support for incorporating the last component name as an additional input enables us to label certain files correctly upon creation and thus avoids that problem entirely. 

Furthermore, Smalley said, the pathname-based hooks that are currently available in the LSM API are not usable to solve this problem because they don't address the issue of assigning labels to new inodes. The existing hooks are ""about enforcing access control upon file accesses based on the pathname used to reach the file"". The SELinux community has reached a consensus that the proposed change is needed, Smalley said, and the only real question in his mind was whether the changes were acceptable to the Linux virtual filesystem (VFS) and various filesystem developers. 

While Schaufler [recognizes](<https://lwn.net/Articles/419220/>) that the SELinux community is fully behind the change, he wonders if there are other hooks that could also benefit from the filename information: 

One of the concerns that has traditionally been raised when new LSM hooks or changes to existing hooks are proposed is that of generality. I can think of a number of ways in which the final component of a pathname could be used to make access control decisions, but I would not expect to be using them myself. Who else might you expect to make use of this LSM "enhancement", or is this something that only SELinux is ever going to want? Is the component something the LSM should be providing in general, or is this the only case in which it makes sense? 

He goes on to point out that the LSM API is inconsistent and arbitrary, so it would make sense to look at the "bigger picture" before hacking in a change specifically for SELinux. As an example, he posits a possible access control mechanism that uses file extensions to make decisions (""only files suffixed with '.exe' can be executed and only files suffixed with '.so' can be mmapped""). Smalley [believes](<https://lwn.net/Articles/419221/>) that kind of access control could be done with the existing pathname-based hooks, but Kyle Moffett [came up](<https://lwn.net/Articles/419223/>) with another place where the filename information might be useful, even for SELinux: 

While you of course cannot (and should not) *change* the label of a file in a link() or rename() operation, it would potentially be useful to deny an operation based on the old label and the new name that is being passed in. 

The example Moffett gives would deny a compromised web application the ability to rename or link to the `.htaccess` file in its directories. 

So far, none of the VFS or filesystem hackers have spoken up one way or another, so it is unclear whether this change will be acceptable to them. The LSM API is something of a kernel outcast--or so it appears at times--as no one is particularly satisfied with it, yet it is an integral part of the kernel security landscape. Sometimes that means that various "hacks" get added for specific security solutions, without looking at the overall picture, which is rather unfortunate. It may well be that this change is adopted, as is, without considering other potential users or consistency in the API.
