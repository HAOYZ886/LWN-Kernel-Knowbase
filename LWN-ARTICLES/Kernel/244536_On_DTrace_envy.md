---
title: On DTrace envy
url: https://lwn.net/Articles/244536/
date: "August 7, 2007"
category: SystemTap; Tracing
author: "By Jonathan Corbet August 7, 2007"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
August 7, 2007

When Sun looks to highlight the strongest features of the Solaris operating system, DTrace always appears near the top of the list. Your editor recently had a conversation with an employee of a prominent analyst firm who was interested, above all else, in when Linux would have some sort of answer to DTrace. There is a common notion out there that tracing tools is one place where Linux is significantly behind the state of the art. So your editor decided to take a closer look. 

The Linux tool which is most similar to DTrace is [SystemTap](<http://sourceware.org/systemtap/>). This development is supported by a number of high-profile companies, including Red Hat, Intel, IBM, and Hitachi. Most distributions have SystemTap packages somewhere in their systems of repositories, making it readily available to Linux users. DTrace supporters have been known to say that SystemTap is merely a [knock-off of DTrace](<http://blogs.sun.com/ahl/entry/dtrace_knockoffs>), and a badly-done one at that. SystemTap proponents will counter that it is an independent development which can hold its own. 

Both tools are based on the insertion of probe points in the system kernel. Whenever a thread of execution hits one of those probe points, some sort of action - as described in the tool's C-like language - is run. That action can be as simple as printing a message, or it can be significantly more complicated than that. 

DTrace comes with a large set of pre-defined probe points wired into the Solaris kernel - seemingly tens of thousands of them. These points are [well documented](<http://docs.sun.com/app/docs/doc/817-6223>) and cover most of the kernel. Some simple wildcarding is implemented for the selection of multiple probe points. It is claimed that the run-time overhead of unused probe points is negligible. [**Update** : see the comments for some useful clarification on the use of dynamic probe points in DTrace.] 

SystemTap, instead, does not depend on static probe points within the kernel; that capability exists, but nobody has much interest in maintaining all of those points. Instead, SystemTap uses dynamic probes (implemented with [kprobes](<http://sourceware.org/systemtap/kprobes/>)) which are inserted into the kernel at run time. A flexible language can enable probes to be easily inserted anywhere in the kernel, with fairly complete wildcard support which allows, for example, all functions within a source file or subsystem to be instrumented with a single statement. Unused probe points do not exist at all, and so cannot affect system performance. 

There are a couple of advantages to the DTrace approach. The probe points exist and can be easily found in the manuals; a SystemTap user, instead, is required to have a certain amount of familiarity with the kernel source code. DTrace probe points are fixed at locations where it is known to be safe to interrupt the execution of the kernel. The SystemTap documentation, instead, comes with warnings that placing probes in the wrong places can cause system crashes and mutterings about the possibility of implementing blacklists in the future. The number of "wrong places" appears to be quite small, but that is of limited comfort for an administrator trying to observe the operation of a production system - something which is supposed to be possible with either system. There is a set of predefined points provided in the "tapsets" packaged with SystemTap, but it is small. 

The "D" language provided with DTrace is more restricted than the SystemTap language, though it does have a few features - like the ability to print stack traces - which appear to be missing in SystemTap. The D language has no flow control or looping constructs. Instead, the code associated with a probe has a predicate expression determining whether that code is executed when the probe is hit. Thus each selected probe point can be thought of as having a single, controlling "`if`" statement around it, with no further flow control possible afterward. 

SystemTap's language, instead, has conditionals, loops, and the ability to define functions. It also has, for those who like to live dangerously, the ability to embed C code. There are clear advantages to a more powerful scripting language, but hazards as well: SystemTap must, for example, carry extra code to keep infinite loops in scripts from bringing down the system. 

D is, like Java, compiled to a special virtual machine and interpreted at run time. SystemTap, instead, compiles directly to C. So SystemTap code may execute more quickly, but D may benefit from the additional safety checks which a virtual machine allows. 

DTrace has the ability to work with user-space probes. As with the kernel, developers are required to insert the probe points before DTrace can use them; it is not clear that large amounts of user-space code have been so instrumented. There is clear elegance to the idea, though, and this capability may prove genuinely useful in the future as more applications are equipped with probe points. SystemTap does not currently have this capability. 

In practice, simply getting SystemTap to work can be a challenge - even when a distributor-supported package is available. SystemTap is clearly its own development which must be (somewhat painfully) integrated with a specific kernel. DTrace can be expected to simply work out of the box. 

And that is perhaps the biggest difference between the two tracing systems. SystemTap would appear to have all of the capabilities it really needs to be a powerful system tracing tool - at least on the kernel side. DTrace features which are missing - [speculative tracing](<http://docs.sun.com/app/docs/doc/817-6223/6mlkidli7?a=view>), for example - could certainly be added if there were demand for it. Evidently user-space tracing is in the works. But what SystemTap really needs is more basic than that. What's missing is the degree of maturity exhibited by DTrace. 

SystemTap needs to simply work on most systems - and be usable by the system administrators. To a great extent, the "simply work" part is something that the distributors must address. Current SystemTap packages as tested by your editor have the look of an edge-of-the-repository afterthought. They do not have the dependencies to bring in the needed kernel information, requiring a fair amount of manual "what does it need now?" administrative work. Even then, performance is spotty at best; the SystemTap utilities just do not have access to the sort of information (uncompressed kernel images, for example) that they need to operate correctly. Until an administrator can simply tell the package management system to install SystemTap and expect to have it work thereafter, it will be hard to convince anybody that we have a mature tracing tool. 

On the development side, there should be an extensive set of well-documented trace points which can be used without having to go into the kernel source. Digging deeply into the system in a flexible way is always going to require a certain amount of skill, but SystemTap all but requires its users to be kernel hackers. The hard work of making a tool which can match - and, in places, exceed - DTrace has been done. What remains is a large (but relatively straightforward) job: making this tool usable by a much wider set of system administrators. Until that is done, DTrace envy will remain with us.
