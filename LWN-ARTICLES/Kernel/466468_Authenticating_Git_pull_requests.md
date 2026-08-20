---
title: Authenticating Git pull requests
url: https://lwn.net/Articles/466468/
date: "November 9, 2011"
category: Git
author: "By Jake Edge November 9, 2011"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jake Edge**  
November 9, 2011

One of the [outcomes](<https://lwn.net/Articles/464233/>) from the kernel.org compromise is the increased use of GPG among kernel developers. GPG keys are now required to get write access to the kernel.org Git repositories, and folks are starting to think about how to use those keys for other things. Authenticating pull requests made by kernel hackers to Linus Torvalds are one possible use. But, as the discussion on the linux-kernel mailing list shows, there are a few different use-cases that might benefit from cryptographic signing. 

Most of the code that flows into the kernel these days comes from Git trees that various lieutenants or maintainers manage. During the merge window (and at other times), Torvalds is asked to "pull" changes from these trees via an email from the maintainer. In the past, Torvalds has used some _ad hoc_ heuristics to determine whether to trust that the request (and the tree) are valid, but, these days, stronger assurances are needed. That's where GPG signing commits and tags may be able to help. 

Conceptually the idea is simple: the basic information required to do a pull (location and branch of the Git tree along with the commit ID of its head) could be signed by the developer requesting the pull. Torvalds could then use GPG with his keyring of kernel developer public keys to verify that the signature is valid for the person who sent the request. That would ensure that the pull request is valid. It could all be done manually, of course, but it could also be automated by making some changes to Git. 

The discussion on how to do that automation started after a [signed pull request](<https://lwn.net/Articles/466549/>) for libata updates was posted by Jeff Garzik. The entire pull request mail (some 3200+ lines including the diffs and diffstat) was GPG signed, which mangled the diff output as Garzik noted. Beyond that, though, it is unwieldy for Torvalds to check the signature, partly because he uses the GMail web interface. In order to check it, he has to cut and paste the entire message and feed it to GPG, which is labor intensive and might be prone to the message being mangled--white space or other changes--that would lead to a false negative signature verification. As Torvalds [noted](<https://lwn.net/Articles/466550/>): ""We need to automate this some sane way, both for the sender and for the recipient."" 

The initial goal is just to find a way to ensure that Torvalds knows who the pull request is coming from and where to get it, all of which could be handled outside of Git. Rather than signing the entire pull request email, just a small, fixed-format piece of that mail could be signed. In fact, Torvalds [posted](<https://lwn.net/Articles/466573/>) a patch to `git-request-pull` to do just that. It still leaves the integrator (either Torvalds or a maintainer who is getting a pull request from another developer) doing a cut-and-paste into GPG for verification, however. 

There are others who have an interest in a permanent trail of signatures that could be audited if the provenance of a particular part of the kernel needs to be traced. That would require storing the signatures inside the Git tree somehow, so that anyone with a copy of Torvalds's tree could see any of the commits that had been signed, either by Torvalds or by some other kernel hacker. But, as Torvalds [pointed out](<https://lwn.net/Articles/466576/>), that information is only rarely useful: 

Having thought about it, I'm also not convinced I really want to pollute the "git log" output with information that realistically almost nobody cares about. The primary use is just for the person who pulls things to verify it, after that the information is largely stale and almost certain to never be interesting to anybody ever again. It's *theoretically* useful if somebody wants to go back and re-verify, but at the same time that really isn't expected to be the common case. 

Torvalds's idea is that the generation of the pull request is the proper time for a developer to sign something, rather than having it tied to a specific commit. His example is that a developer or maintainer may wish to push the tree out for testing (or to linux-next), which requires that it be committed, but then request a pull for that same commit if it passes the tests. Signing before testing has been done is likely to be a waste of time, but signing the commit later requires amending the commit or adding a new empty commit on top, neither of which were very palatable. Git maintainer Junio C. Hamano is [not convinced](<https://lwn.net/Articles/466578/>) that ephemeral signatures (i.e. those that only exist for the pull-request) are the right way to go, though: ""But my gut feeling is that 'usually hidden not to disturb normal users, but is cast in stone in the history and cannot be lost' strikes the right balance."" 

The conversation then turned toward tags, which can already be signed with a GPG key. One of the problems is that creating a separate tag for each commit that gets signed rapidly becomes a logistical nightmare. If you just consider the number of trees that Torvalds pulls in a normal merge window (hundreds), the growth in the number of signed tags becomes unwieldy quickly. If you start considering all of the sub-trees that get pulled into the trees that Torvalds pulls, it becomes a combinatorial explosion of tags. 

What's needed is an automated method of creating tag-like entries that live in a different namespace. That's more or less what Hamano [proposed](<https://lwn.net/Articles/466579/>) by adding a `refs/audit` hierarchy into the `.git` directory data structures. The audit objects would act much like tags, but instead carry along information about the signature verification status of the merges that result from pulls. In other words, a `git-pull` would verify the signature associated with the remote tag (which are often things like "for-linus" that get reused over and over) and create an entry in the local audit hierarchy that recorded the verification. Since the audit objects wouldn't pollute the tag namespace, and would be pulled and created automatically, they will have much less of an impact on users and existing tools. In addition, the audit objects could then be pushed into Torvalds's public tree so that audits could be done. 

So far, Hamano has [posted](<https://lwn.net/Articles/466517/>) a patch set that implements parts of his proposed solution. In particular, it allows for signing commits, verifying the signatures, and for pulling signed tags. Other pieces of the problem are still being [worked on](<https://lwn.net/Articles/466582/>). 

As is often the case in our communities, adversity results in pretty rapid improvements. For the kernel, the SCO case brought about the Developer's Certificate of Origin, the relicensing of BitKeeper gave us Git, the kernel.org break-in brought about a closer scrutiny of security practices, and the adoption of GPG keys because of that break-in will likely lead to even better assurances of the provenance of kernel code. While we certainly don't want to court adversity, we certainly do take advantage of it when it happens.
