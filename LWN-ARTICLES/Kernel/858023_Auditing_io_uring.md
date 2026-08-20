---
title: Auditing io_uring
url: https://lwn.net/Articles/858023/
date: "June 3, 2021"
category: "Auditing; io uring-Security"
author: "By Jonathan Corbet June 3, 2021"
---

By **Jonathan Corbet**  
June 3, 2021

The [io_uring subsystem](<https://lwn.net/Articles/776703/>), first introduced in 2019, has quickly become the leading way to perform high-bandwidth, asynchronous I/O. It has drawn the attention of many developers, including, more recently, those who are focused more on security than performance. Now some members of the security community are lamenting a perceived lack of thought about security support in io_uring, and are trying to remedy that shortcoming by adding audit and Linux security module support there. That process is proving difficult, and has raised the prospect of an unpleasant fallback solution. 

The Linux [audit](<https://man7.org/linux/man-pages/man8/auditd.8.html>) mechanism allows the monitoring and logging of all significant activity on the system. If somebody wants to know, for example, who looked at a specific file, an audit-enabled system can provide answers. This capability is required to obtain any of a number of security certifications which, in turn, are crucial if one wants to deploy Linux in certain types of security-conscious settings. It is probably fair to say that a relatively small percentage of Linux systems have auditing turned on, but distributors, almost without exception, enable auditing in their kernels. 

The audit mechanism relies, in turn, on a large array of hooks sprinkled throughout the kernel source. Whenever an event that may be of interest occurs, it is reported via the appropriate hook to the audit code. There, a set of rules loaded from user space controls which events are reported to user space. 

When io_uring was being developed (which is still happening now, of course), the developers involved were deeply concerned about performance and functionality. Supporting security features like auditing was not at the top of their list, so they duly neglected to add the needed hooks — or to think about how auditing could be supported in a way consistent with the performance goals. Now that io_uring is showing up in more distributor kernels (and, in particular, the sorts of kernels where auditing is relatively likely to be enabled), security-oriented developers are starting to worry about it. Having io_uring serve as a way to circumvent the otherwise all-seeing audit eye does not seem like a good way to maintain those security certifications. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

#### Adding security support

In late May, Paul Moore (a maintainer of the audit subsystem) posted [a set of patches](<https://lwn.net/ml/linux-fsdevel/162163367115.8379.8459012634106035341.stgit@sifl/>) adding Linux security module (LSM) and audit capabilities to io_uring. The LSM side is relatively straightforward; the operations performed by io_uring are already covered by LSM hooks, so all that was needed was a pair of new hooks to pass judgment on io_uring-specific actions. Specifically, these hooks control the sharing (between processes) of credentials that are stored with the ring buffer that is used to communicate operation requests to the kernel; see [this patch](<https://lwn.net/ml/linux-fsdevel/162163382536.8379.3124023175473604584.stgit@sifl/>) for details. This part of the patch set does not seem to be controversial. 

The audit code is another story. The core io_uring code has been carefully optimized to dispatch requests and their results as quickly as possible. Use cases for io_uring can involve performing millions of I/O operations per second, so any added overhead will prove most unwelcome. Adding the audit hooks to cover operations submitted through the ring slowed down one of the most performance-critical parts of io_uring, leading Pavel Begunkov to [react negatively](<https://lwn.net/ml/linux-fsdevel/f07bd213-6656-7516-9099-c6ecf4174519@gmail.com/>): ""So, it adds two if's with memory loads (i.e. current->audit_context) per request in one of the hottest functions here... No way, nack"". 

Begunkov suggested that perhaps the audit hooks could be patched in at run time when they are actually enabled, the way tracepoints and kprobes work. Moore [responded](<https://lwn.net/ml/linux-fsdevel/CAHC9VhRjzWxweB8d8fypUx11CX6tRBnxSWbXH+5qM1virE509A@mail.gmail.com/>) that the audit subsystem doesn't support that sort of patching, and that doing so could raise problems of its own: ""I fear it would run afoul of the various security certifications"". So that does not appear to be the route to a possible solution. 

Meanwhile, Jens Axboe, the io_uring maintainer, [ran some tests](<https://lwn.net/ml/linux-fsdevel/9e69e4b6-2b87-a688-d604-c7f70be894f5@kernel.dk/>). A simple random-read test slowed down by nearly 5% with the audit hooks installed, even in the absence of any actual audit rules. Various other benchmarks, even when run with an updated version of the patch set (which was not posted publicly), gave the same results. Kernel developers can work for months for a 5% performance gain; losing that amount to audit hooks is a bitter pill for them to swallow. 

Axboe pointed out that read and write operations are not audited when they are initiated through the older asynchronous I/O system calls. ""In the past two decades, I take it that hasn't been a concern?"" He agreed that some operations (such as opening or removing files) should be audited, but said that auditing read and write operations was ""just utter noise and not useful at all"". Since those operations are the ones where performance matters the most, taking the audit hooks out of the fast path for them might be a possible solution. 

Moore [suggested](<https://lwn.net/ml/linux-fsdevel/CAHC9VhRZEwtsxjhpZM1DXGNJ9yL59B7T_p2B60oLmC_YxCrOiw@mail.gmail.com/>) an approach based on that idea; only a specific, carefully chosen set of operations would have the audit hooks applied. There is a handy `switch` statement in the io_uring dispatcher that makes it easy to instrument just the desired operations. He asked for feedback, but has not gotten much so far. The important question, as Begunkov [pointed out](<https://lwn.net/ml/linux-fsdevel/94e50554-f71a-50ab-c468-418863d2b46f@gmail.com/>), is which operations in particular need audit support. Adding an audit call when opening a file is unlikely to bother anybody; a call added to, say, a poll operation would be another story. Moore has posted [an initial set of operations](<https://lwn.net/ml/linux-fsdevel/CAHC9VhS7Vhby4YR94U2YOwMtva-rc=_ifRcZYi1YVPwfi+Xuzg@mail.gmail.com/>) that he thinks merit auditing. 

#### Threats and grumbles

With luck, that solution will prove acceptable to everybody. The alternative to adding audit support to io_uring is, [according to Moore](<https://lwn.net/ml/linux-fsdevel/CAHC9VhTS_Yt0PzG_WjsgUA04inHa=N8+OjWju9waefP==Di39A@mail.gmail.com/>), not particularly pleasant: 

> If we can't find a solution here we would need to make io_uring and audit mutually exclusive and I don't think that is in the best interests of the users, and would surely create a headache for the distros. 

"Headache" is not really the word for it. If the two features are made exclusive, then it will not be possible to configure a kernel containing both of them. So distributors would have to either ship two different kernels (something they will go far out of their way to avoid) or pick one of the two features to support. Hopefully it will not come to that. 

Meanwhile, there has been some disgruntlement expressed by developers on both sides, but the security developers have made it especially clear that they would have liked to see audit designed into io_uring from the beginning. As Casey Schaufler [put it](<https://lwn.net/ml/linux-fsdevel/cba563e3-60da-3edf-e5fe-e409415ce3cc@schaufler-ca.com/>): 

> It would have been real handy had the audit and LSM requirements been designed into io_uring. The performance implications could have been addressed up front. Instead it all has to be retrofit. 

Richard Guy Briggs also [complained](<https://lwn.net/ml/linux-fsdevel/20210528223544.GL447005@madcap2.tricolour.ca/>) that ""multiple aspects of security appear to have been a complete afterthought to this feature, necessitating it to be bolted on after the fact"". The implication in both cases is that, with adequate forethought, the difficulties being encountered now could have been avoided. 

That is arguably a fair criticism. Kernel developers working on new features often leave security as something to be thought about later; that is especially true for relatively niche features like auditing, which is unlikely to be enabled on development systems. The kernel community can be a bit unfriendly toward its security developers, characterizing them as prioritizing security above anything else (and above performance in particular). Such an environment seems like a recipe for leaving security concerns by the wayside, to be fixed up later. 

It is also fair to point out, though, that io_uring has been developed in public since early 2019. It has been heavily discussed on the mailing lists (and in LWN), but the security community did not see that as their cue to make suggestions on how features like auditing could be supported. It is a rare kernel developer who can summon the focus to implement a million-operation-per-second I/O subsystem while simultaneously making provisions for security hooks that won't kill performance. Perhaps the io_uring developers should have been considering security from the beginning, but they should also have had help from the beginning. 

The kernel community has surprisingly few rules regarding the addition of new features like io_uring. In theory, new system calls should come with manual pages, but it's somewhat surprising when that actually happens. In a project with a more bureaucratic process, it would make sense to insist that new features do not go in until they have proper support for mechanisms like LSM and auditing. That might force earlier interactions with security developers and avoid this kind of problem. 

That is not the world that we live in, though; there is nobody with a checklist making sure that all of the relevant boxes have been marked before a new subsystem can be merged. So the kernel community will have to continue to muddle along, supporting the needed features as best it can. This is not the last time that a security mechanism will have to be retrofitted into an existing kernel feature. It's arguably not the best approach, but it generally gets the job done in the end.
