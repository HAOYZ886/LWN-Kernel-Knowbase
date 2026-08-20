---
title: Supporting multiple LSMs
url: https://lwn.net/Articles/426921/
date: "February 9, 2011"
category: "Security-Security modules"
author: "By Jake Edge February 9, 2011"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jake Edge**  
February 9, 2011

With some regularity, the topic of allowing multiple Linux Security Modules (LSMs) to all be active comes up in the Linux kernel community. There have been some attempts at "stacking" or "chaining" LSMs in the past, but nothing has ever made it into the mainline. On the other hand, though, every time a developer comes up with some kind of security hardening patch for the kernel, they are generally directed toward the LSM interface. Because the "monolithic" security solutions (like SELinux, AppArmor, and others) tend to have already taken the single existing LSM slot in many distributions, these simpler, more targeted LSMs are generally unable to be used. But a discussion on the linux-security-module mailing list suggests that work is being done that just might solve this problem. 

The existing implementation of LSMs uses a single set of function pointers in a `struct security_operations` for the "hooks" that get called when access decisions need to be made. Once a security module gets registered (typically at boot time using the `security=` flag), its implementation is stored in the structure and any other LSM is out of luck. The idea behind LSM stacking would be to keep multiple versions of the `security_operations` structure around and to call each registered LSM's hooks for an access decision. While that sounds fairly straightforward, there are some subtleties that need to be addressed, especially if different LSMs give different answers for a particular access. 

This problem with the semantics of "composing" two (or more) LSMs has been discussed at various points, without any real global solution for composing arbitrary LSMs. As Serge E. Hallyn [warned](<https://lwn.net/Articles/368838/>) over a year ago: 

The problem is that composing any two security policies can quickly have subtle, unforeseen, but dangerous effects. That's why so far we have stuck with the status quo where only one LSM is 'active', but that LSM can manually call hooks from other LSMs. 

There is one example of stacking LSMs as Hallyn describes in the kernel already; the capabilities LSM is called directly from other LSMs where necessary. That particular approach is not very general, of course, as LSM maintainers are likely to lose patience with adding calls for every other possible LSM. A more easily expandable solution is required. 

David Howells posted a [set of patches](<https://lwn.net/Articles/426890/>) that would add that expansion mechanism. It does that by allowing multiple calls to the `register_security()` initialization function, each with its own set of `security_operations`. Instead of the current situation, where each LSM manages its own data for each kind of object (credentials, keys, files, inodes, superblocks, IPC, and sockets), Howell's security framework will allocate and manage that data for the LSMs. 

The `security_operations` structure gets new `*_data_size` and `*_data_offset` fields for each kind of object, with the former filled in by the LSM before calling `register_security()` and the latter being managed by the framework. The data size field tells the framework how much space is needed for the LSM-specific data for that type of object, and the offset is used by the framework to find each LSM's private data. For `struct cred`, `struct key`, `struct file`, and `struct super_block`, the extra data for each registered LSM is tacked onto the end of the structure rather than going through an intermediate pointer (as is required for the others). Wrappers are defined that will allow an LSM to extract its data from an object based on the new fields in the operations table. 

The framework then maintains a list of registered LSMs and puts the capabilities LSM in the first slot of the list. When one of the security hooks is called, the framework iterates over the list and calls the corresponding hook for each registered LSM. Depending on the specific hook, different kinds of iterators are used, but the usual iterator looks for a non-zero response from an LSM's hook, which would indicate a denial of some kind, and returns that to the framework. The other iterators are used for specialized calls, for example when there is no return value or when only the first hook found should be called. The upshot is that the hooks for registered LSMs get called in order (with capabilities coming first), and the first to deny the access "wins". Because the capabilities calls are pulled out separately, that also means that the other LSMs no longer have to make those calls themselves; instead the framework will handle it for them. 

But there are a handful of hooks that do not work very well in a multi-LSM environment, in particular the `secid` (an LSM-specific security label ID) handling routines (e.g. `secid_to_secctx()`, `task_getsecid()`, etc.). Howells's current implementation just calls the hook of the first LSM it finds that implements it, which is not going to make it possible to use multiple LSMs that all implement those hooks (currently just SELinux and Smack). Howells's [solution](<https://lwn.net/Articles/427105/>) is to explicitly ban that particular combination: 

I think the obvious thing is to reject any chosen module that implements any of these interfaces if we've already selected a module that implements them. That would mean you can choose either Smack or SELinux, but not both. 

But Smack developer Casey Schaufler isn't [convinced](<https://lwn.net/Articles/427107/>) that is the right course: ""That kind of takes the wind out of the sails, doesn't it?"" He would rather see a more general solution that allows multiple `secid`s, and the related `secctx`s (security contexts), to be handled by the framework: 

It does mean that there needs to be a standard for a secctx that allows for the presence of multiple concurrent LSMs. There will have to be an interface whereby either the composer/stacker can break a secctx into its constituent parts or with which an LSM can pull the bit it cares about out. In either case the LSMs may need to be [updated] to accept a secctx in a standardized format. 

Another interesting part of Schaufler's message is that he has been working on an ""alternative approach"" to the multi-LSM problem that he calls "Glass". The code is, as yet, unreleased, but Schaufler describes Glass as an LSM that composes other LSMs: 

The Glass security blob is an array of pointers, one for each available LSM, including commoncap, which is always in the last slot. The Glass LSM is always registered first. As subsequent LSMs register they are added to the glass LSM vector. When a hook is invoked glass goes through its vector and if the LSM provides a hook it gets called, and the return remembered. If any other LSM provided a hook the commoncap hook is skipped, but if no LSM was invoked commoncap is called. 

Unlike Howells's proposal, Glass would leave the calls to the capabilities LSM (aka commoncap) in the existing LSMs, and only call commoncap if no LSM implemented a given hook. The idea is that the LSMs already handle the capabilities calls in their hooks as needed, so it is only when none of those get called that requires a call into commoncap. In addition, Glass leaves the allocation and management of the security "blobs" (LSM-specific data for objects) to the LSMs rather than centralizing them in the framework as Howells's patches do. 

In addition to various other differences, there is a more fundamental difference in the way that the two solutions handle multiple LSMs that all have hooks for a particular security operation. Glass purposely calls each hook in each registered LSM, whereas Howells's proposal typically short-circuits the chain of hooks once one of them has denied the access. Schaufler's [idea](<https://lwn.net/Articles/427126/>) is that an LSM should be able to maintain state, which means that skipping its hooks could potentially skew the access decision: 

My dreaded case is an LSM that bases controls on statistical frequency of access to files. There is no way you could skip any of its hooks, and I don't see off hand any file access hook it wouldn't use. I have heard people (think credit card companies) suggest such things, so although I don't have use for it I can't discount the potential for it. 

There are plenty of other issues to resolve, including things like handling `/proc/self/attr/current` (which contains the security ID for the current process) because various user-space programs already parse the output of that file, though it is different depending on which LSM is active. A standardized format for that file, which takes multiple LSMs into account, might be better, but it would break the kernel ABI and is thus not likely to pass muster. Overall, though, Howells and Schaufler were making some good progress on defining the requirements for supporting multiple LSMs. Schaufler is [optimistic](<https://lwn.net/Articles/427130/>) that the collaboration will bear fruit: ""I think that we may be able to get past the problems that have held multiple LSMs back this time around."" 

So far, there is only the code from Howells to look at, but Schaufler has promised to make Glass available soon. With luck, that will lead to a multi-LSM solution that the LSM developers can coalesce behind, whether it comes from Howells, Schaufler, or a collaboration between them. There may still be a fair amount of resistance from Linus Torvalds and other kernel hackers, but the [lack](<https://lwn.net/Articles/368730/>) of any way to combine LSMs [comes up](<https://lwn.net/Articles/393008/>) too often for it to be ignored forever.
