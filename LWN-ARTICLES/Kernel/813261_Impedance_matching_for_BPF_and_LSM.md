---
title: Impedance matching for BPF and LSM
url: https://lwn.net/Articles/813261/
date: "February 26, 2020"
category: "BPF-Security; Security-Security modules"
author: "By Jake Edge February 26, 2020"
---

By **Jake Edge**  
February 26, 2020

The "kernel runtime security instrumentation" (KRSI) patch set has been making the rounds over the past few months; the idea is to use the Linux security module (LSM) hooks as a way to detect, and potentially deflect, active attacks against a running system. It does so by allowing BPF programs to be attached to the LSM hooks. That has caused [some concern](<https://lwn.net/Articles/808048/>) in the past about exposing the security hooks as external kernel APIs, which makes them potentially subject to the "don't break user space" edict. But there has been no real objection to the goals of KRSI. The fourth version of the patch set was [posted](<https://lwn.net/ml/linux-kernel/20200220175250.10795-1-kpsingh@chromium.org/>) by KP Singh on February 20; the concerns raised this time are about its impact on the LSM infrastructure. 

The main change Singh made from the previous version effectively removed KRSI from the standard LSM calling mechanisms by using [BPF "fexit" (for function exit) trampolines](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=fec56f5890d9>) on the LSM hooks. That trampoline can efficiently call any BPF programs that have been placed on the hook without the overhead associated with the normal LSM path; in particular, it avoids the cost of the [retpoline](<https://support.google.com/faqs/answer/7625886>) mitigation for the Spectre hardware vulnerability. The KRSI hooks are enabled by [static keys](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/static-keys.txt>), which means they have zero cost when they are not being used. But it does mean that KRSI looks less like a normal LSM as Singh acknowledged: ""Since the code does not deal with security_hook_heads anymore, it goes from 'being a BPF LSM' to 'BPF program attachment to LSM hooks'."" 

Casey Schaufler, who has done a lot of work on the LSM infrastructure over the last few years, [objected](<https://lwn.net/ml/linux-kernel/0ef26943-9619-3736-4452-fec536a8d169@schaufler-ca.com/>) to making KRSI a special case, however: ""You aren't doing anything so special that the general mechanism won't work."" Singh [agreed](<https://lwn.net/ml/linux-kernel/20200221114458.GA56944@google.com/>) that the standard LSM approach would work for KRSI, ""but the cost of an indirect call for all the hooks, even those that are completely unused is not really acceptable for KRSI’s use cases"". 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Kees Cook [focused](<https://lwn.net/ml/linux-kernel/202002211946.A23A987@keescook/>) on the performance issue, noting that making calls for each of the hooks, even when there is nothing to do, is common for LSMs. Of the 230 hooks defined in the LSM interface, only SELinux uses more than half (202), Smack is next (108), and the rest use less than a hundred—several use four or less. He would like to see some numbers on the performance gain from using static keys to disable hooks that are not required. It might make sense to use that mechanism for all of LSM, he said. Singh [agreed](<https://lwn.net/ml/linux-kernel/20200224172309.GB21886@chromium.org/>) that it would be useful to have some performance numbers; ""I will do some analysis and come back with the numbers."" It is, after all, a bit difficult to have a discussion about improving performance without some data to guide the decision-making. 

There are several intertwined pieces to the disagreement. The LSM infrastructure has generally not been seen as a performance bottleneck, at least until the advent of KRSI; instead, over recent years, the focus has been on generalizing that infrastructure to support [arbitrary stacking of multiple LSMs](<https://lwn.net/Articles/804906/>) in a running system. That has improved the performance of handling calls into multiple hooks (when more than one LSM defines one for a given operation) over the previous mechanism, but it was not geared to the high-performance requirements that KRSI is trying to bring to the LSM world. 

In addition, the stacking work has made it so that LSMs can be stacked in any order; each defined hook for a given operation is called in order, the first that denies the operation "wins" and the operation fails without calling any others. That infrastructure is in place, but KRSI upends it to a certain extent. KRSI comes from the BPF world, so the list traversal and indirect calls used by the LSM infrastructure are seen as performance bottlenecks. KRSI always places itself (conceptually) last on the list and uses the BPF trampoline to avoid that overhead. That makes it a special case, unlike the other "normal" stackable LSMs, but that may be a case of a premature optimization, as Cook noted. 

BPF developer Alexei Starovoitov [does not see it as "premature"](<https://lwn.net/ml/linux-kernel/20200223220833.wdhonzvven7payaw@ast-mbp/>) at all, however. ""I'm convinced that avoiding the cost of retpoline in critical path is a requirement for any new infrastructure in the kernel."" He thought that the LSM infrastructure should consider using static keys to enable its hooks, and that the mechanism employed by KRSI should be used there: 

Just compiling with CONFIG_SECURITY adds "if (hlist_empty)" check for every hook. Some of those hooks are in critical path. This load+cmp can be avoided with static_key optimization. I think it's worth doing. 

[...] I really like that KRSI costs absolutely zero when it's not enabled. Attaching BPF prog to one hook preserves zero cost for all other hooks. And when one hook is BPF powered it's using direct call instead of super expensive retpoline. 

But the insistence on treating KRSI differently than the other LSMs means that perhaps KRSI should go its own way—or work on improving the LSM infrastructure as a whole. Schaufler [said](<https://lwn.net/ml/linux-kernel/c5c67ece-e5c1-9e8f-3a2b-60d8d002c894@schaufler-ca.com/>) that he had not ""gotten that memo"" on avoiding retpolines and that the LSM infrastructure is not new. He is interested in looking at using static keys, but is concerned that the mechanism is too specific to use in the general case, where multiple LSMs can register hooks to be called: 

I admit to being unfamiliar with the static_key implementation, but if it would work for a list of hooks rather than a singe hook, I'm all ears. 

The new piece is not KRSI per se, Singh [said](<https://lwn.net/ml/linux-kernel/20200224171305.GA21886@chromium.org/>), but the ability to attach BPF programs to the security hooks _is_ new. There are techniques available to make that have zero cost, so it makes sense to use them: 

There are other tracing / attachment [mechanisms] in the kernel which provide similar [guarantees] (using static keys and direct calls) and it seems regressive for KRSI to not leverage these known patterns and sacrifice performance [especially] in hotpaths. This provides users to use KRSI alongside other LSMs without paying extra cost for all the possible hooks. 

[...] My analogy here is that if every tracepoint in the kernel were of the type: 

```
if (trace_foo_enabled) // <-- Overhead here, solved with static key
       trace_foo(a);  // <-- retpoline overhead, solved with fexit trampolines
```

It would be very hard to justify enabling them on a production system, and the same can be said for BPF and KRSI. 

The difficulty is that the LSM interface came about under a different set of constraints, Schaufler [said](<https://lwn.net/ml/linux-kernel/00c216e1-bcfd-b7b1-5444-2a2dfa69190b@schaufler-ca.com/>). Those constraints have changed over time and the infrastructure is being worked on to improve its performance, but it still needs to work with the existing LSMs: 

The LSM mechanism is not zero overhead. It never has been. That's why you can compile it out. You get added value at a price. You get the ability to use SELinux and KRSI together at a price. If that's unacceptable you can go the route of seccomp, which doesn't use LSM for many of the same reasons you're on about. 

When LSM was introduced it was expected to be used by the lunatic fringe people with government mandated security requirements. Today it has a much greater general application. That's why I'm in the process of bringing it up to modern use case requirements. Performance is much more important now than it was before the use of LSM became popular. 

[...] If BPF and KRSI are that performance critical you shouldn't be tying them to LSM, which is known to have overhead. If you can't accept the LSM overhead, get out of the LSM. Or, help us fix the LSM infrastructure to make its overhead closer to zero. Whether you believe it or not, a lot of work has gone into keeping the LSM overhead as small as possible while remaining sufficiently general to perform its function. 

The goal of eliminating the retpoline overhead is reasonable, Cook [said](<https://lwn.net/ml/linux-kernel/202002241136.C4F9F7DFF@keescook/>), but the LSM world has not yet needed to do so. ""I think it's a desirable goal, to be sure, but this does appear to be an early optimization."" He noted there is something of an impedance mismatch; the BPF developers do not want to see any performance hit associated with BPF, but the LSM developers ""do not want any new special cases in LSM stacking"". So he suggested adding a "slow" KRSI that used the LSM stacking infrastructure as it is today, followed by work to optimize that calling path. 

But Starovoitov [thought](<https://lwn.net/ml/linux-kernel/20200225054125.dttrc3fvllzu4mx5@ast-mbp/>) that KRSI should perhaps just go its own way. He proposed changing the BPF program type from `BPF_PROG_TYPE_LSM` to `BPF_PROG_TYPE_OVERRIDE_RETURN` and moving KRSI completely out of the LSM world: ""I don't see anything in LSM infra that KRSI can reuse."" He does not see a slow KRSI as an option and suggested that perhaps the LSM interface and the new BPF program type should be made mutually exclusive at kernel build time: 

It may seem as a downside that it will force a choice on kernel users. Either they build the kernel with CONFIG_SECURITY and their choice of LSMs or build the kernel with CONFIG_BPF_OVERRIDE_RETURN and use BPF_PROG_TYPE_OVERRIDE_RETURN programs to enforce any kind of policy. I think it's a pro not a con. 

There are, of course, lots of users of the LSM interface, including most distributions, so it might difficult to go that route, Schaufler [said](<https://lwn.net/ml/linux-kernel/4b56177f-8148-177b-e1e5-c98da86b3b01@schaufler-ca.com/>). But Singh [noted](<https://lwn.net/ml/linux-kernel/20200226051535.GA17117@chromium.org/>) that the users of a `BPF_PROG_TYPE_OVERRIDE_RETURN` feature may be highly performance-sensitive such that they already disable LSMs ""because of the current performance characteristics"". But Singh did [think](<https://lwn.net/ml/linux-kernel/20200225193108.GB22391@chromium.org/>) that `BPF_PROG_TYPE_OVERRIDE_RETURN` might be useful on its own, separate from the KRSI work; he plans to split that out into its own patch set. He [agreed](<https://lwn.net/ml/linux-kernel/20200225192913.GA22391@chromium.org/>) with Cook's approach, as well, and plans to re-spin the KRSI patches to use the standard LSM approach as a starting point; ""we can follow-up on performance"". 

The clash here was the classic speed versus generality tradeoff that pops up in the kernel (and elsewhere) with some frequency. The BPF developers are laser-focused on their "baby"—and squeezing every last drop of performance out of it—but there is a wider world in kernel-land, some parts of which have different requirements. It would seem that a reasonable compromise has been found here. Preserving the generality of the LSM approach while gaining the performance improvements that the BPF developers have been working on would be a win for both, really—taking the kernel and its users along for the ride.
