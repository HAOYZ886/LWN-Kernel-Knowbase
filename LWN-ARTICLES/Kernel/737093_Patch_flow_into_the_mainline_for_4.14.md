---
title: Patch flow into the mainline for 4.14
url: https://lwn.net/Articles/737093/
date: "October 24, 2017"
category: "Releases-4.14; Security-Patch verification"
author: "By Jonathan Corbet October 24, 2017"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
October 24, 2017

There is a lot of information buried in the kernel's Git repositories that, if one looks closely enough, can yield insights into how the development community works in the real world. It can show how the idealized hierarchical model of the kernel development community matches what actually happens and provide a picture of how the community's web of trust is used to verify contributions. Read on for an analysis of the merge operations that went into the 4.14 development cycle. 

The diagram to the right was generated from the commits merged for the 4.14 release, through 4.14-rc5. It is unfortunately dense; click on the image to get a version that has a chance of being legible. In short, it shows [![\[Patchflow diagram\]](https://static.lwn.net/images/2017/patchflow-4.14.svg)](<https://lwn.net/Articles/737094/>) all of the subsystem trees that were pulled into the mainline and the number of patches that flowed out of each. 

LWN has posted these diagrams a couple of times in the past, for the [2.6.29](<https://lwn.net/Articles/318699/>) and [4.4](<https://lwn.net/Articles/670209/>) development cycles. They have always shown a structure that is far flatter than the hierarchical maintainer model would suggest. In the real world, mid-level maintainers are relatively rare; most maintainers send pull requests directly to Linus Torvalds. Doing so helps to get changes into the mainline more quickly; that is why, for example, some security-module maintainers recently [decided](<https://lwn.net/Articles/737254/>) to bypass the security maintainer and push their trees directly to Torvalds. 

That said, the hierarchy shows more clearly than it has in past years. A number of subsystems are growing to the point where there needs to be some overall higher-level coordination. So there are more two and three-level trees than there used to be. As the kernel community continues to grow, it will almost certainly need to add more mid-level maintainers. 

#### Signing of pull requests

Diagrams like this one can be interesting to look at just to see how work is flowing through the system. But they can also be used to reveal semi-hidden aspects of how that work is being done. This time around, your editor has decided to put a focus on the security of the process. 

Shortly after the 3.0 kernel was released, it was revealed that kernel.org, where many kernel developers (including Torvalds) keep their repositories, had been broken into. This episode brought the merging of patches to a halt for some time and delayed the 3.1 release by some months; it also created a great deal of concern over the possibility that somebody's repository might have been corrupted in an attempt to get malicious code into the mainline kernel. No evidence of that happening ever turned up, but the realization that it maybe could have happened drove a number of changes in the development community. 

One of those changes was the establishment of a web of trust among kernel developers; at the [2011 Kernel Summit](<https://lwn.net/Articles/KernelSummit2011/>) in Prague, an initial key-signing ritual was held to bootstrap that web. The ability to GPG-sign commits and tags was added to Git. One need merely tag the commit at the head of a series to be pulled with a command like: 

```
git tag -s fixes-for-linus
```

and request that the `fixes-for-linus` tag be pulled. If the receiving maintainer pulls with the `--verify-signatures` option, Git will ensure that a valid signature exists before doing the merge. 

The idea was that developers would sign their repositories before sending pull requests, allowing upstream maintainers to verify that those pull requests corresponded to legitimate streams of development. Even if an attacker could put up a convincing copy of a developer's repository (or somehow add a malicious commit to a real repository) and send a fake pull request, the attack would not succeed because the attacker would not be able to attach a proper signature to the relevant tag. 

This system has been in place for six years now, and many developers routinely sign tags for outgoing commits and verify signatures when pulling from others. But do they all do so? It is possible to find out. When a signed commit or tag is pulled into a repository, the signature is stashed into the merge commit, allowing the provenance of the changes to be verified at a later date. That also makes it possible to examine the merges in the kernel repository and see how many of them carry signature information. 

Referring back to the tree plot on the right, one will see that some repositories are shown in black boxes, while others use red boxes. The repositories in red are those from which no signed merges happened during the period in question. The results show that, while many developers do sign their tags before sending changes upstream, quite a few do not. 

More to the point, the repository that sends more traffic into the mainline than any other — networking — makes almost no use of signatures anywhere in the chain. The "tip" tree (containing x86 and core-kernel work) is another significant tree that does not employ signatures, as is the linux-block tree. Neither the security tree nor the crypto tree employ cryptographic signatures. Pull requests from the graphics tree into the mainline are signed, but many of the trees feeding into graphics do not use signatures. On the other hand, some high-volume trees, such as arm-soc, have almost complete signature coverage from the leaves up to the mainline. 

Years of traffic on the kernel mailing lists suggests that maintainers rarely ask for signatures to be added to pull requests that lack them. Torvalds will typically demand it when the tree being pulled is hosted on a public service like GitHub, but is otherwise happy to pull from unsigned tags. He does verify signatures when they do exist, though. Few other maintainers require (or even mention) signatures at all. 

Your editor asked around a bit to get a sense for why some maintainers are not using signed tags. The answer was typically along the lines of "I never got around to incorporating them into my workflow". One maintainer admitted that he had probably forgotten the passphrase for his GPG key by now and would have to create a new one to be able to start signing tags. The problem, if there is one, is not any real hostility to the idea of signed commits. It is just that, since signatures are not required, many busy subsystem maintainers have not made the effort to start using them. 

The result is that the kernel has a web of trust that, one might fairly conclude, is not really protecting much. It's nice to have the verification on pull requests that do carry signatures but, since those signatures seem to be almost entirely optional at present, they offer little protection against a malicious pull request. 

If the intent of signed tags is limited to enabling developers to host repositories on untrusted services, then perhaps signature checking as it is practiced now is sufficient. Perhaps the threat model need not include more sophisticated attackers trying to sneak vulnerabilities into the kernel via some developer's tree on a well-run site. After all, kernel.org itself seems relatively well protected these days, and kernel developers have demonstrated that, like developers of most other projects, they are entirely capable of introducing security bugs at a sufficient rate without external assistance. 

But if the intent is to make the kernel development process resilient against attacks on developers' machines or kernel.org, then there is some work yet to be done. It is worth remembering that the web of trust came about as a response to a compromise of kernel.org, after all. If we want to prepare for a recurrence of that sort of incident, the actual threat model needs to be defined, and the use of protective techniques like signed tags should probably not be optional. Partially implemented security mechanisms have a distressing tendency to fail when put to the test. 

(The plot in this article was generated with the `treeplot` tool, which is part of the gitdm collection of hacks hosted at `git://git.lwn.net/gitdm.git`).
