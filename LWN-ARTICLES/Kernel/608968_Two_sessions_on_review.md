---
title: Two sessions on review
url: https://lwn.net/Articles/608968/
date: "August 20, 2014"
category: "Development model-Code review"
author: "By Jonathan Corbet August 20, 2014 Kernel Summit 2014"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
August 20, 2014

* * *

[Kernel Summit 2014](<https://lwn.net/Articles/KernelSummit2014/>)

Like almost every other free software project out there, the kernel has a fundamental problem: not enough reviewers. Some areas of development need more review than others; at the top of the list must be the creation of binary interfaces for user space, since those interfaces must be maintained for a very long time. But the problem goes beyond ABI definition. Two sessions at the 2014 Kernel Summit looked at the issue of review and how it can be improved. 

#### ABI changes

The first session, led by Michael Kerrisk and Andy Lutomirski, was concerned with the ABI issue in particular. Michael started by saying that, whenever he gets around to testing a new system call, he finds a bug 50% of the time. Christoph Hellwig remarked that the 50% number meant he wasn't looking hard enough. The point being made was that there is clearly not enough review and testing of code that is going out the door in a stable release. Indeed, sometimes there has been no testing at all. The `recvmmsg()` system call, Michael said, was implemented with a timeout value that cannot possibly have done anything sensible in the initial release. 

Sometimes we change ABIs as well, often without meaning to. For example, with the inotify interface, the `IN_ONESHOT` option in early kernels did not trigger the `IN_IGNORED` option; in later kernels that behavior has changed. 

There is, he said, no specification for new ABIs, a fact which makes them hard to review and test. The lack of a specification also leads to subtle implementation problems. Again citing inotify as an example, Michael talked about the difficulties involved in tracking files that move between [![\[Michael and Andy\]](https://static.lwn.net/images/conf/2014/ks/Michael+Andy-sm.jpg)](<https://lwn.net/Articles/608970/>) directories; see [this article](<https://lwn.net/Articles/605128/>) for details. There are no man pages for most new system calls and not enough reviewers, with the result that questionable designs get through. The `O_TMPFILE` option serves as a good example, Michael said; beyond [its other problems](<https://lwn.net/Articles/562294/>), there should also have been consideration given to implementing that functionality as a separate system call. 

Andy added that a specification is good, but a unit test for a new ABI is also a good thing to have. At this point, Peter Zijlstra asked if maybe the [Linux Test Project](<https://github.com/linux-test-project/ltp>) would be a better place than the kernel tree for unit tests. But LTP is concerned with testing more than just system calls, and many kernel developers see the LTP system as a big extra thing that they would have to install. 

Ted Ts'o observed that developers must have tests for the new features they have. Otherwise they would not be exercising the expected level of diligence in their work. Dave Airlie added that it would be good to have those tests in the kernel tree. He also suggested that, perhaps, the community should insist on the existence of a man page for any new system call before it could be added to the mainline. Michael responded that this has been tried before, without much success. But, it was pointed out, all four new system calls added in 3.17 came with manual pages. 

Ben Herrenschmidt pointed out that system calls are just the tip of the iceberg. The kernel's ABI includes other aspects, such as `ioctl()` calls, sysfs, netlink, and more. 

A topic that came up repeatedly is that any patch changing the kernel ABI should be copied to the linux-api mailing list. Perhaps, it was suggested, it should be the subsystem maintainers' duty to ensure that this copying happens when needed. Josh Triplett suggested that the `get_maintainer` script could be modified to make that happen, but that idea was not received all that warmly. That script tends to add lots of unrelated recipients to patch mailings and is not universally popular among kernel developers. 

Peter Anvin made the claim that the linux-api list simply is not working. Perhaps it would be better, he said, to just merge the man pages into the kernel tree so that the code and documentation could be patched together. Michael responded that this idea [has come up before](<https://www.kernel.org/doc/man-pages/todo.html#migrate_to_kernel_source>). There are advantages and disadvantages to it. On the down side, there is a lot of material in the man pages that does not describe the kernel interface. The man pages are documentation for application developers, not for kernel developers, so they cover a lot of glibc interfaces, wrappers around system calls, and more. 

The discussion wound down with some repeated suggestions that maintainers should insist on man pages before accepting new system calls into the kernel. The plea to copy the linux-api list for changes affecting the kernel ABI was also repeated. For a while, at least, kernel developers may try to do better, but no true solution to the problem was found at this session. 

#### Getting more reviewers

James Bottomley stepped up to run a more general discussion on the problem of patch review. How, he asked, can we increase the amount of review that code going into the mainline gets? Are there any new ideas out there on how the kernel's review process can be improved? No answers resulted, but the session did cover some of the mechanics of how review is performed. 

Peter Zijlstra said that he has been getting patches with bogus `Reviewed-by` tags. "Bogus," in this case, means that no in-depth review of the code has actually happened. Often, these tags come from [![\[James Bottomley\]](https://static.lwn.net/images/conf/2014/ks/JamesBottomley-sm.jpg)](<https://lwn.net/Articles/608972/>) other developers working for the same company as the patch author, but not always. James said that he ignores `Reviewed-by` tags unless they are offered in an email that has substantive comments in it. 

Darren Hart said, though, that these tags can be the result of internal review that happens before patches are posted on a public list. In some companies, at least, this review process is serious and rigorous; it makes sense to document those reviews in the patch. Dave asked why the process has to be done internally; why not expose the whole process on the public lists? Darren answered that, with almost any project, it is natural to go for "small-group consensus" before facing the wider world. 

James added that he is often suspicious of same-vendor reviews, but they are not necessarily invalid. It is really a matter of how much one trusts the specific reviewers involved. 

He went on to ask a general process question: how much can a patch change before a `Reviewed-by` tag needs to be reviewed? A white-space change probably should not bring about a need for a new review, but substantial changes to how the patch works should. There were some differences of opinion in the room about just where the line should be drawn; in the end it seems to come down to common sense and the subsystem maintainer's opinion on the matter. 

The session ended with Linus jumping in and saying that, in the end, the `Reviewed-by`, `Acked-by`, and `Cc` tags all mean the same thing: the person named in the tag will be copied on the report if the patch turns out to be buggy. Some developers use one tag, while others use a different one, but there is no real difference between them. The session closed with some general disagreement over the meanings of the different tags — and no new ideas on how to get more review of kernel code. 

**Next** : [One year of Coverity work](<https://lwn.net/Articles/608992/>).
