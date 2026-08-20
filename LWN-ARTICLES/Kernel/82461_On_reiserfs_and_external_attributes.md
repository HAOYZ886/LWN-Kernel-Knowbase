---
title: On reiserfs and external attributes
url: https://lwn.net/Articles/82461/
date: "April 28, 2004"
category: "Filesystems-Extended attributes; Filesystems-Reiser4"
author: 
---

[The patch](<https://lwn.net/Articles/81978/>) seemed relatively straightforward; Chris Mason had sent out a set of reiserfs changes which include `data=journal` support, an improved block allocator, metadata readahead, and external attribute support. One of those changes, however, does not sit well with Hans Reiser, the original creator of reiserfs. 

External attributes are just a way of attaching extra metadata to files; they are used for things like access control lists and SELinux context information. Most of the standard Linux filesystems support external attributes in 2.6, but reiserfs does not yet have that capability. Given that features like SELinux will not work without external attributes, adding this capability has been high on the wish lists of many users and developers. 

When the external attribute patch was posted, however, Hans Reiser sent out [a protest](<https://lwn.net/Articles/82480/>) asking that the patch not be applied. Those who have followed Hans's work over the years will know what his objection is: external attributes live in their own name space. Hans has dedicated much effort to the task of moving everything into the filesystem name space; he says: 

The expressive power of an operating system is NOT proportional to the number of components, but instead is proportional to the number of possible connections between its components. If you fragment the namespaces of an OS, you reduce each component to effective interactions with only those components in its reduced size namespace. Designing the namespaces of an OS so that they possess closure and are unified may seem like a lot of effort, but it is very cost effective compared to building many times more other OS components to get the same expressive power. 

The upcoming [Reiser4 filesystem](<http://namesys.com/v4/v4.html>) implements Hans's vision of how external attributes should be implemented; essentially, each attribute just looks like a small file containing the attribute value. The solution is fast and elegant; it may well be the way things are done in the future. For the moment, however, there are a few problems: 

  * Reiser4 is still in beta testing, and has not yet been submitted for inclusion into the 2.6 kernel. Once it is submitted, it is not certain that it will be accepted quickly. 

  * The Reiser4 external attribute API is different from the API used in the 2.6 kernel. Applications, to use this API, will have to be rewritten to use the special-purpose `reiser4()` system call. 

  * Some users of reiserfs ("Reiser3") might be a little nervous about making an immediate jump to a completely new filesystem. They just might want to be able to continue using their existing filesystems and, simultaneously, make use of external attributes. 

The solution seems reasonably clear: Reiser4, once it's ready, can be merged with its new ways of doing things. The existing reiserfs filesystem, meanwhile, can be augmented with the capabilities that its users would like to have now. This approach would seem to offer the best of both worlds. Mr. Reiser disagrees; he would rather not have (what he sees as) an inelegant hack grafted onto reiserfs to satisfy immediate needs. When code is released as free software, however, not even its creator can prevent its development in certain directions if that's what its users want.
