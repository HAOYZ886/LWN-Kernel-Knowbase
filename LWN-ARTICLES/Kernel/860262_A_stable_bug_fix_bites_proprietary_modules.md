---
title: A stable bug fix bites proprietary modules
url: https://lwn.net/Articles/860262/
date: "June 21, 2021"
category: "Development model-Stable tree; Modules-Exported symbols"
author: "By Jonathan Corbet June 21, 2021"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
June 21, 2021

The kernel-development community has long had a tense relationship with companies that create and ship proprietary loadable kernel modules. In the view of many developers, such modules are a violation of the GPL and should simply be disallowed. That has never happened, though; instead, the community has pursued a policy of legal ambiguity and technical inconvenience to discourage proprietary modules. A "technical-inconvenience" patch that was merged nearly one year ago has begun to show up in stable kernel releases, leading at least one developer to complain that things have gone a little too far. 

Code that is directly linked into the kernel can access any symbol that is visible to it. Loadable modules, instead, are restricted to a smaller (though still large) set of symbols that are explicitly "exported" for that purpose. Symbols that are exported with `EXPORT_SYMBOL()` are available to all loadable modules, while those exported with `EXPORT_SYMBOL_GPL()` can only be used by modules that declare a GPL-compatible license. A non-GPL-compatible module that tries to use a GPL-only symbol will fail to load. 

The idea behind GPL-only exports is that the affected symbols are so deeply situated within the kernel that any module using them must be a derived product of the kernel and, thus, be subject to the requirements of the GPL. In practice, that sort of analysis is rarely (if ever) done, and the decision of whether to use a GPL-only export is left to individual developers. Many developers habitually use `EXPORT_SYMBOL_GPL()` for every symbol they export out of a general distaste for proprietary modules; some maintainers encourage this practice for code that passes through their hands. 

Over the years, purveyors of proprietary modules have engaged in a number of tricks to get around GPL-only exports. One of those was manually looking up symbol addresses with `kallsyms_lookup_name()`; that practice was [shut down](<https://lwn.net/Articles/813350/>) in early 2020. Another is to split a module into two, one GPL-licensed and one proprietary. The GPL-licensed module interfaces directly with the kernel, using GPL-only symbols where needed; it then calls into the proprietary module, where all the real work gets done. 

In July 2020, the posting of this kind of shim module [created a stir](<https://lwn.net/Articles/827596/>) on the mailing lists, leading to the posting by Christoph Hellwig of [a patch set](<https://lwn.net/ml/linux-kernel/20200730061027.29472-1-hch@lst.de/>) making this trick harder to exploit. Specifically, any module that uses symbols exported by a proprietary module is itself marked proprietary, regardless of the license it declares to the kernel. Modules that hook into proprietary modules, thus, will lose access to GPL-only symbols, making it impossible to perform the shim function they were created for in the first place. This series was merged for the 5.9 kernel release in October. 

That was the end of that story — until May 2021, when that patch series found its way into the large [5.4.118](<https://lwn.net/Articles/855998/>), [4.19.191](<https://lwn.net/Articles/857000/>), and [4.14.233](<https://lwn.net/Articles/857001/>) updates. It seemingly took nearly another month for 5.4.118 to find its way into at least one distribution and create trouble for users, at which point Krzysztof Kozlowski [asked](<https://lwn.net/ml/linux-kernel/8edc6f45-6c42-19c7-6f40-6f1a49cc685b@canonical.com/>) why such a change was being included in a stable update: 

> How this is a stable material? What specific, real bug that bothers people, is being fixed here? Or maybe it fixes serious issue reported by a user of distribution kernel? IOW, how does this match stable kernel rules at all? 
> 
> For sure it breaks some out-of-tree modules already present and used by customers of downstream stable kernels. Therefore I wonder what is the bug fixed here, so the breakage and annoyance of stable users is justified. 

Stable-kernel maintainer Greg Kroah-Hartman [responded](<https://lwn.net/ml/linux-kernel/YMxlP2EMTaG9+2y6@kroah.com/>): 

> It fixes a reported bug in that somehow symbols are being exported to modules that should not have been. This has been in mainline and newer stable releases for quite some time, it should not be a surprise that this was backported further as well. 

That pretty much ended the conversation; others may be unhappy about this change making it into older kernels, but it is doubtful that anybody realistically expects that it could be reverted. It might be interesting, though, to watch kernel updates from distributors to see whether this additional restriction on proprietary modules is retained or quietly removed. 

This change does show where at least one limit to the kernel's "no regressions" policy is to be found, though. The core idea behind that policy is that a kernel upgrade should never break a working system; kernel developers want users to feel confident that they can move to newer kernels without risking unpleasant surprises. This change does indeed create such a surprise for some users, according to Kozlowski. But kernel modules have never been included in the kernel's stability guarantees — not even the GPL-licensed ones. Kernel-developer tears are rarely shed when proprietary modules break, and they are not in evidence this time either. 

When Hellwig's patch series was first posted, LWN noted that it did not close all of the loopholes. Specifically, a GPL-licensed module that wraps and re-exports symbols to a proprietary module will still work as designed as long as no symbols are imported from that proprietary module; this problem was [pointed out](<https://lwn.net/ml/linux-kernel/5ac70bdf2c5b440c83f12e75ca42a107@AcuMS.aculab.com/>) by David Laight in this discussion. Kroah-Hartman responded with [a promise](<https://lwn.net/ml/linux-kernel/YMxlBCzztbWGvi/l@kroah.com/>) to ""work on fixing that up in a future patch series next week"", so there may be more unpleasant surprises in store for the creators and users of proprietary loadable kernel modules. 

The kernel community's policy on loadable modules has, over the years, drawn criticism from many sides. Some see even a grudging tolerance of proprietary modules as a weakening of the protections provided by the GPL and a cover for vendors that don't want to play by the rules. Others see it as an impediment to the use of Linux in general that reduces available hardware support and makes users jump through unnecessary hoops. The best way to judge this policy, though, is to look at what its results have been over nearly three decades. 

Proprietary modules still exist, but they are in the minority; most hardware is supported with free drivers, and the situation seems to continue to slowly improve. Vendors that have clung to proprietary modules in the past have found ways to change their approach; this might not have happened if those vendors had been excluded from the Linux community entirely. So, perhaps, making life uncomfortable for distributors of such modules while not trying to ban them outright may be the most productive policy in the long run.
