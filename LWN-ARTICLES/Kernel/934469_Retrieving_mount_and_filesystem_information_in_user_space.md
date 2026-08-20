---
title: Retrieving mount and filesystem information in user space
url: https://lwn.net/Articles/934469/
date: "June 14, 2023"
category: "Filesystems-Mounting; System calls"
author: "By Jake Edge June 14, 2023 LSFMM+BPF"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
June 14, 2023

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2023>)

In something of a follow-on from the [mount-operation monitoring session](<https://lwn.net/Articles/932648/>) the previous day, Christian Brauner led another discussion about providing user space with a mechanism to get current mount information on day two of the [2023 Linux Storage, Filesystem, Memory-Management and BPF Summit](<https://lwn.net/Articles/lsfmmbpf2023>). The session also continued on from [one at last year's summit](<https://lwn.net/Articles/897420/>)—and likely others before that. There are two separate proposals for ways to retrieve this kind of information, one from Miklos Szeredi and another from David Howells, both of whom were present this year; Brauner's intent was to try to reach some kind of agreement on the way forward in the session. 

#### Background

Brauner began by noting that user-space developers, Lennart Poettering in the back of the room in particular, have been asking him for a way to query mount and filesystem information. He said that Howells has [proposed the `fsinfo()` system call](<https://lwn.net/Articles/829212/>) and that Szeredi has [worked on `getvalues()`](<https://lwn.net/Articles/890449/>), which has shifted to use the extended attributes (xattr) interface, as was discussed last year. There were other proposals in the mix, Brauner said; "we somehow need to come to an acceptable compromise" so that things can move forward. 

People have different preferences with regard to the facility; his main one is that the information not be exposed as a filesystem, because "it is a giant pain for user space, usually". It might be good to hear Poettering's thoughts on what kind of API he would prefer, since systemd will be a user of it. 

Szeredi noted that Linus Torvalds had suggested having a [`stat()`](<https://man7.org/linux/man-pages/man2/lstat.2.html>)-like system call that reported information from `/proc/self/mountinfo`. It could return a mixture of binary and textual information, which is seen by some as problematic; the simplest thing would be to simply return the text line from the `mountinfo` file for a requested mount. Beyond that, though, is a need to be able to list the child mounts (i.e. mounts on subdirectories) for a given mount. A new system call could simply return a list of extended mount IDs (of the 64-bit variety mentioned in the earlier session) for the children. Those are simply some of his ideas, Szeredi said, perhaps others have different ideas. 

After Torvalds weighed in on the `fsinfo()` proposal, several people contacted Howells to ask that the interface not be textual because parsing text can be painful and is slow. Brauner said that he would prefer a non-textual interface as well. Jeff Layton asked for a recap of the objections to `fsinfo()`; Brauner said that he thought the main problem was that mount notification and the general query mechanism were combined in a single, huge patch set. It was a difficult patch set to review, which is part of what Torvalds was reacting to. Brauner thinks that splitting some of the functionality up over multiple system calls is not a problem; the days where it was truly difficult to add a new system call are over. 

Szeredi had no major objection to a binary interface, but there are some pieces of mount and superblock information that are hard to represent in binary. There is a textual format that is already established; even if it is difficult to parse, there are already parsers available for the format. The performance objections do not stem from parsing a single entry in `mountinfo` but from having to parse the entire file, he said. 

Poettering said that, from his perspective, the problems with textual interfaces is "that splitting out the fields is always nasty [...] because of escaping and figuring out what the delimiters are". The kernel is not good at having uniform logic for those kinds of interfaces either. If the fields are separated in the structure returned from some kind of query, a textual interface would be fine with him. 

He would like to see a single, atomic call to return information, as with [`statx()`](<https://man7.org/linux/man-pages/man2/statx.2.html>), rather than have to make several calls to retrieve what is needed. Brauner asked if that single call needed to also return the child mounts or if it would be sufficient for it to provide the mount ID that could be used in a separate call to retrieve the child mounts. That would be fine with Poettering; he would like information about a kernel object to be returned in an atomic fashion, but "I have no illusion that getting an atomic view of more than an object" is sensible. Brauner said that the mount table can be constantly changing, so getting a snapshot of it is inherently racy; Poettering agreed and said that was not what he was looking for. 

#### Extensible interface

Brauner proposed a starting point for the API to consist of `fsinfo()` (or some other name, "we can squabble about this") that had a structure with a core set of information that is useful for user space. That structure could be extensible, "we know how to do this, we've done this before", even though some people do not like extensible structures that are versioned by size. It would take a mask of what information was being requested and it would return a mask with which of those were available. There can be both textual and binary information in that structure. 

Amir Goldstein said that it is important to be able to query for filesystem-specific information, which Szeredi's xattr-based API could support. In fact, it would have been nice to have a way to do that for `statx()`, as well, since some filesystems do have inode-specific information that is exported. There are virtual xattrs that some filesystems support for that, he said. Poettering and others seemed to think that was a reasonable approach, though there may be some wrinkles to iron out with it. 

Steve French said that CIFS makes various types of filesystem-specific information available in `/proc`, though it is not clear to him how user space could go from a file descriptor to get to those entries. Ted Ts'o said that he is leery of mixing filesystem-specific information queries in with the more general query mechanism, in part because a simpler proposal will be easier to review. It is clear what the use case for the mounted filesystem information is, Ts'o continued, but much less so for all of these esoteric, filesystem-specific bits; combining the two may add more complexity for little gain. Querying for that extra information can be addressed separately. The virtual xattr approach is contentious, with some, including Christoph Hellwig, finding it to be a "radical abuse of that interface"; even if that is not a reasonable position, he would rather avoid that particular battle. 

Josef Bacik noted that the mount options are the only filesystem-specific information that would be returned from the generalized query; he wondered if the mount options, beyond attributes like read-only, were needed by user space. Poettering said that systemd is interested in the universally unique ID (UUID) for the superblock, but Brauner cautioned that exposing the UUID is more complicated than it might seem. Some filesystems generate a UUID, but others do not; some filesystems use the UUID to generate a filesystem ID (FSID), but not always. For example, XFS generates the FSID from the block-device information. So exposing that information requires additional work, but if the query mechanism is extensible, that can all come later. 

Brauner suggested that the filesystem-specific question could be set aside for now. A core structure could be defined that is generic for all filesystems, then another text-based system call for filesystem-specific options (e.g. mount options) could be added. 

Poettering would prefer to get all of the mount options together in a single call, rather than one-by-one in multiple calls. Howells said that he wanted something like that too for a "mount supervisor" that he wants to create. The supervisor would intercept mount requests in a mount namespace and allow or deny them based on the mount options; it could also be used for NFS automounts, he said. 

Poettering noted that the [util-linux](<https://github.com/util-linux/util-linux>) utilities use its libmount, which gathers up the mount options so that various tools can report them; if the idea is to support those use cases, the mount options are going to need to be available. Bacik said that it seems perfectly reasonable to him to just provide the mount options in the form of a string (or a list of single-option strings). That code is already present for the `mountinfo` file. Ts'o agreed that providing filesystem-specific mount options that way made sense. If there is a need for something with more structure for mount options, it can be added later. 

#### xattrs?

Szeredi said that there is already a system call that can be used to get filesystem-specific options: [`getxattr()`](<https://man7.org/linux/man-pages/man2/getxattr.2.html>). But Ts'o said: "I will let you fight with Christoph about using `getxattr()` for something that is not a real extended attribute". Ts'o does not think it is a good generic approach, though individual filesystems can do whatever they want. There are other problems with using `getxattr()`, Howells said, including mounts that are not reachable via a path or a file descriptor. 

From afar, Al Viro asked what a mount supervisor is going to be able to do with a mount option that refers to a file descriptor by number. Poettering added that the options listed in `mountinfo` for automounted filesystems have things like "`fd=5`", which is not meaningful to other processes. Howells said that he had added the 64-bit mount IDs that could be used to identify mounts; those could be queried using `fsinfo()` or some other interface. 

Poettering also noted that xattrs are normally a property of an inode, so making them suddenly return information about the filesystem is a bit weird and will be hard to explain; that would be another reason to find a different style of interface, he thought. Another remote participant, Darrick Wong, wondered if `fspick()` could choose a filesystem based on an FSID and if the file descriptor it returned could be used to get filesystem-specific virtual xattrs, which might actually route around Hellwig's objections. Wong was guessing that Hellwig did not like mixing the regular and virtual xattrs so that you could not tell the difference if someone had simply added a regular xattr with the same name as a virtual one. 

Howells said that `fspick()` takes a path or file descriptor, so you cannot use an FSID. Layton said that CephFS had its own xattr namespace, so virtual xattrs could be distinguished from the regular ones. But Brauner does not think the xattr API is a good one, so it is not one that should be used for this purpose. "This is a really broken API in my opinion." It is type-unsafe and convoluted; access-control list (ACLs) were moved out of xattrs, so he hopes that filesystem attributes can be moved out as well. 

Brauner suggested that converging on a slimmed-down version of `fsinfo()`, under that or some other name, and adding a separate system call to retrieve the mount options in textual form. That should provide util-linux and systemd what they need. Layton suggested adding UUID into `fsinfo()` even if all filesystems did not support them (yet); if the request/response mask is used, those filesystems can just not report it. 

Brauner said that a goal of getting that into the kernel by the end of the year, or early in 2024, seemed reasonable. It is mostly a matter of copying the code for `statx()` and hacking it up to be suitable for generic filesystem information. Goldstein added, "and make sure it's extensible", which to Brauner sounded like Goldstein was volunteering to do the work. Rapid backpedaling to general laughter was the result. 

Szeredi wondered about getting child-mount information, but Brauner thought there had been agreement on a new system call for that. There was some discussion about ways to shoehorn that information into `fsinfo()`, but Brauner and others are resistant to the idea of variable-length arrays embedded into structures. Eric Biggers asked that any new system calls that get added for this have both documentation and tests. The session wound down shortly thereafter, but not before Brauner, with a big grin, said "hopefully we can all remember the good spirit" of the session on the mailing list when patches start getting posted.
