---
title: "Coming soon: eCryptfs"
url: https://lwn.net/Articles/156921/
date: "October 25, 2005"
category: Filesystems
author: 
---

eCryptfs developer Michael Halcrow recently [announced](<https://lwn.net/Articles/156922/>) that he will shortly be putting eCryptfs up for inclusion into the -mm tree. This filesystem aims to make "enterprise level" (it comes from IBM, after all) file encryption capabilities available in a secure and easy to use manner. Those who are interested in trying it out early can download it from [SourceForge](<http://sourceforge.net/projects/ecryptfs>). 

The eCryptfs developers took the stacking approach, meaning that, rather than implement its own platter-level format, eCryptfs sits on top of another filesystem. It is, essentially, a sort of translation layer which makes encrypted file capabilities available. The system administrator can thus create encrypted filesystems on top of whatever filesystem is in use locally, or even over a network-mounted filesystem. 

The design of eCryptfs envisions providing a great deal of flexibility in the use of the filesystem. Rather than encrypt the filesystem as a whole, eCryptfs deals with each file individually. Different files can be encrypted in different ways. The use of this sort of mechanism implies that eCryptfs must maintain metadata on how each file is to be handled. This metadata is placed in the first block of the file itself, meaning that the file can be backed up, copied, and even moved to another system without losing the metadata needed to decrypt it in the future. 

Plans for eCryptfs include a wide range of features. There will be dynamic, public-key encryption with each user's GPG keyring. On systems equipped with "trusted platform" (TPM) modules, the TPM will be used for its encryption capabilities and the ability to lock files to a specific system. Key escrow systems can be worked in for companies which need that feature. For the upcoming 0.1 release, however, eCryptfs will only support a single passphrase mode. The rest can be added once the initial problems have been shaken out and some policy support work has been done. 

Many of the advanced features have been implemented, however, and can be tried out by sufficiently motivated testers. The developers are interested in feedback from people who can give eCryptfs a try or look over the source. Having seen the difficulties experienced by some filesystem implementers as they tried to get their work merged, the eCryptfs hackers would, doubtless, like to get any potential issues resolved sooner rather than later.
