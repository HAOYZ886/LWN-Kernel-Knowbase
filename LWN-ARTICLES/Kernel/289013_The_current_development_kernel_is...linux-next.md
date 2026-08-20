---
title: "The current development kernel is...linux-next?"
url: https://lwn.net/Articles/289013/
date: "July 8, 2008"
category: "Development model-linux-next; linux-next"
author: "By Jonathan Corbet July 8, 2008"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
July 8, 2008

One of the development process advantages brought by git (and by BitKeeper before it) is the ability to see the up-to-the-second, bleeding-edge status of Linus's tree. So any developer who wants to know where the front edge of development lies can grab that tree and make patches fit into it. But the value of the mainline repository for development would appear to be less than it once was. The mainline is no longer where the action is. 

Consider, for example, [this response from Andrew Morton](<https://lwn.net/Articles/289015/>) after finding that a patch posted to linux-kernel would not compile for him: 

I assume this patch was prepared against some ancient out-of-date kernel such as current Linus mainline. Guys, we have a new development tree now. 

He followed up with [this statement](<https://lwn.net/Articles/289016/>): 

But what I am repeatedly seeing is people cheerfully raising 2.6.27 patches against the 2.6.26 tree when we have a nice 2.6.27 tree for developing against. Those days are over, guys. 

So the message would appear to be clear: development work should be done against the linux-next tree rather than against the mainline kernel. There are some clear advantages to having work done in this way. Patches developed against linux-next should merge cleanly during the next merge window. Developers will be testing each other's trees as they work, causing bugs to turn up earlier in the process. And, of course, Andrew won't have to complain about patches which fail to build for him - at least, not as often. 

Linux-next is a somewhat strange base on which to try to develop, though. It is built anew every day from over 100 subsystem trees, each of which can, itself, change from one day to the next. So linux-next is a moving target, just like the mainline is. But, unlike the mainline, linux-next has no consistent or coherent history. Every day's linux-next tree is a completely new creation with a unique - and transient - history. 

Consider a developer who bases some work on a mainline release - 2.6.26-rc9, say. That developer's work will be derived from a specific commit in the mainline tree, known as b7279469d66b55119784b8b9529c99c1955fe747 in this case. The history from 2.6.26-rc9 is well defined, and that series of patches can be merged into any other repository which also contains 2.6.26-rc9; the identity of that commit is consistent and immutable across all repositories. With such a development tree, it is (relatively) easy to track the mainline as it advances, and to merge one's work when the time comes. A git tree based on the mainline sits on a solid foundation. 

It is not possible to base a tree on linux-next in the same way. Development can begin at a specific commit, but tomorrow's linux-next tree may not contain that commit at all. The various component trees will have advanced independently of the previous day's linux-next tree, which can, in itself, complicate things. But the process of making all those trees come together can involve tasks like moving patches from one tree to another, or fixing intermediate patches which break things. That makes the end result better, but at the cost of rebasing those trees. Rebasing completely rewrites the development history, causing the old history to disappear from the tree. So a patch series based on the previous history loses its foundation. 

And, since linux-next is built from its components every day, a patch developed on top of linux-next may, when integrated into that tree, be merged somewhere in the middle of the sequence; in other words, the patch will be merged into a tree which differs considerably from the tree on which it was developed. As Stephen Rothwell, the maintainer of the linux-next tree, [put it](<https://lwn.net/Articles/289019/>): 

One downsides of the way linux-next works is that, because it is recreated every day, you cannot really base anything on it that is to be merged into it. 

Another interesting aspect of linux-next development involves API changes. The longstanding rule in kernel development is that internal kernel interfaces can be changed if there is a good reason to do so, but that the person making the change is obligated to fix all in-tree code broken by that change. If an API change is introduced into linux-next, though, the developer is simply not able to fix any code which enters linux-next by way of the other subsystem trees. If the developer does get patches into those trees for the API change, they can no longer be built on top of kernels which lack that change - the mainline, for example. API changes have, in other words, become harder to do - a situation which some may see as a good thing. 

What all this means is that API changes must be handled through techniques like the creation of backward-compatibility layers; those layers can then be removed a development cycle or two later once the transition is complete. Or changes can be split up and added to individual subsystem trees; that, however, can lead to interesting ordering dependencies between the trees. In some cases, we are seeing 2.6.27 changes [being merged into 2.6.26](<https://lwn.net/Articles/289023/>) in stub form as a way of making all of the pieces fit together. 

Then, there is the simple matter that developers like to have a stable base upon which to create their code. The linux-next tree, since it contains large amounts of relatively new code, will also contain its share of new bugs. That makes developers, who are often having enough trouble just tracking down their own bugs, somewhat grumpy. Development against the mainline tends to have a lower probability of forcing developers to look for bugs which are not of their own making. 

Many of these complaints have an easy answer: the pain which comes from making all the pieces fit together in linux-next must be faced at some point anyway. The real difference is that linux-next allows those problems to be dealt with at leisure, while the older "merge everything in the mainline" model compressed much of that work into the merge window. How beneficial that really is will be seen for the first time in the 2.6.27 merge window; if linux-next is serving its intended function, 2.6.27 should come together with rather less hassle than its immediate predecessors did. 

But, regardless of the value provided by linux-next for integration and testing purposes, the fact remains that it is a difficult platform upon which to develop patches. That process is somewhat like building a house on a sand bar; overnight the tide comes in and completely reshapes the land underneath you. That is why most (possibly all) of the subsystem trees used to assemble linux-next are, themselves, based on the mainline. 

The solution to that problem will have to evolve over time. The linux-next tree is a new institution which is still finding its proper place in the development process. Easier ways to develop patches against the linux-next tree will certainly be worked out; it may well turn out that [quilt](<http://savannah.nongnu.org/projects/quilt/>)-like tools work better for this task than git. But, for now, linux-next is an excellent integration and testing resource, but it has not quite yet managed to become the true Linux kernel development tree.
