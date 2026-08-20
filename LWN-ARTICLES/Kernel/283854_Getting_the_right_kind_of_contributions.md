---
title: Getting the right kind of contributions
url: https://lwn.net/Articles/283854/
date: "May 28, 2008"
category: Development model
author: "By Jake Edge May 28, 2008"
---

By **Jake Edge**  
May 28, 2008

Most free software projects encourage contributors--it is the rare project that has an overabundance--but contributions vary greatly in quality. Encouraging good submissions, or those likely to lead to useful contributions down the road, is an important part of any project. But it is a delicate balance. It can be difficult to determine the kinds of tasks suitable for new contributors that will lead to more important contributions later. 

The flip side of that coin is how to handle contributions that appear to lead elsewhere. Just wading through the significant submissions on a large project's mailing list--linux-kernel being an excellent example--is extremely time consuming; adding noise, in the form of less-than-completely-useful patches, only makes that job harder. New contributors generally want to start with something relatively easy, though, which leads to the tension. 

Discouraging patches that aren't particularly useful in a way that won't chase off prospective kernel hackers is hard. Al Viro's rather intemperate [call for discussion of a linux-wanking mailing list](<https://lwn.net/Articles/283964/>) on linux-kernel is probably not the right approach. He was responding to a patch that reformatted a kernel header file to line up the arguments. Viro is not known for his diplomatic skills, but he was responding to a problem that he and other kernel hackers see. There is an increasing amount of trivial cleanup work being submitted that is not translating to more substantial, useful contributions later on. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

In a followup post, [Viro explains his concern](<https://lwn.net/Articles/283968/>): 

We are getting another self-contained area. Namely, "pick a pointless mechanical work out of ever-growing pile, do it, learn nothing, pick more, maybe look into finding new classes of such mindless stuff". Of course it always had been there; what changes is that now it's not just a transient state one might hit on the way in to be slightly embarrassed about years later. It gets more visible, it gets self-sustained and it gets more and more sticky - it became a subculture in its own right and as far as I can see it is offering more and more incentives to stay in it instead of moving on. 

There is a real cost associated with posts to linux-kernel. It is the main communication mechanism for kernel development so those involved need to work through the posts there. David Miller [laments](<https://lwn.net/Articles/284097/>) the time he spends sorting through it all: 

After deleting all of the noise posted here, I'm often too burnt out to do real work with what's left and just delete that too. :-/ It's worse than the postmaster and list owner mail I process each day for vger.kernel.org 

Wouldn't you like me to instead have the energy left to review some useful patches? 

The kernel project provides a number of resources for people who are interested in getting involved but don't know where and how to start. The [Kernel Newbies](<http://kernelnewbies.org/>) effort is specifically designed to help people get started with the kernel by running a wiki, mailing list, and IRC channel that are focused on the needs of, well, newbies. The idea is to provide information and mentoring that will lead to useful contributions to the kernel. 

A subproject is the [Kernel Janitors](<http://janitor.kernelnewbies.org/>) who focus on cleaning up kernel code: 

We go through the Linux kernel source code, doing code reviews, fixing up unmaintained code and doing other cleanups and API conversions. It is a good start to kernel hacking. 

Both of these efforts are targeted at getting people up to speed so that the kernel as a whole improves. All of the work is important, but there are many other kernel tasks that are not getting done, possibly because contributors are concentrating on cleanups. Andrew Morton has some [suggestions](<https://lwn.net/Articles/283982/>) for interested folks: 

One could understand a developer deciding to write a do-nothing whitespace patch as a general throat-clearing exercise, but when asked, I recommend against that. I generally recommend that people just download and test the latest -rc, linux-next and -mm kernels and build and run them. Because they surely will find things which need fixing. Often simple little things like compilation errors, sometimes things which need a bisection search. 

One problem, though, is that much of that work is more difficult than a whitespace cleanup. For those who are interested in getting their name "up in lights"--in the form of a kernel commit message--the trivial patch path appears easier. Responses like Viro's may deter them, but it risks making linux-kernel look like a hostile place that does not encourage new developers. 

Some extremely important kernel tasks often do get little or no recognition. Submitting detailed bug reports, bisecting the kernel to find the patch that broke things, or testing proposed fixes go unrecognized--at least in the kernel commit log. There have been thoughts of adding tags to the patches that would note these contributions, but no concrete proposal has been made. 

Two other documentation efforts are underway to assist new kernel developers. Jesper Juhl is working on a [Kernel Newbies Guide](<https://lwn.net/Articles/284099/>) to be included into the kernel `Documentation` tree. It may get folded into `Documentation/HOWTO` or as a separate file, but the idea is to steer folks in the right direction--and away from the kinds of patches that raise the ire of kernel hackers. LWN's Jonathan Corbet also [mentioned](<https://lwn.net/Articles/284101/>) a longer document he is working on with support from the Linux Foundation that should be ready for review in June. 

There may be some rudeness or hostility towards new developers on linux-kernel, but it rarely rises to the level seen on the openbsd-misc mailing list last October. In response to a query about a list of less complicated tasks for OpenBSD--similar to what the Linux Kernel Janitors maintain--project leader Theo de Raadt, who is _really_ not noted for his diplomacy, [blasts](<https://lwn.net/Articles/284035/>): 

Surely they are too busy whining at us for lists, to actually search for the lists. 

I'll say it again more clearly -- all of you whiners just plain suck. We know you'll never write diffs, and it is up to you to prove us wrong. If you don't write diffs, we have a difficult time feeling any loss. 

This is sort of the extreme end of the "show us the code" attitude, but, in his own inimitable way, de Raadt is reacting to the same problem. It takes time and effort to shepherd new kernel hackers. Spending time mentoring folks who will never end up contributing is a waste; that time is better spent finding, fixing, or adding bugs. As Linux hacker Ted Ts'o [puts it](<https://lwn.net/Articles/284039/>): 

The real question is whether people who are wanking about whitespace and spelling fixes in comments will graduate to writing real, useful patches. If they won't, there's no point to encouraging them. 

How does a project determine which newly interested people will end up being useful contributors versus those that will not? It is a difficult problem that warrants some thought. It surely isn't just kernel projects that have it, as any large, high-profile project will have both a fairly high barrier to entry along with some developers who should be discouraged. 

Obviously there will never be a clear-cut "future contributor" test, but there may be ways to get a better idea. In the meantime, flaming well-meaning folks to a crisp is unlikely to get there. Referring inappropriate patches to Linux Newbies or something similar--on the off chance the person can be redirected--might be a start.
