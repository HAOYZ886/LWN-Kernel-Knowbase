---
title: Making stable kernels more stable
url: https://lwn.net/Articles/769253/
date: "October 24, 2018"
category: "Development model-Stable tree"
author: "By Jonathan Corbet October 24, 2018 Maintainers Summit"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
October 24, 2018

* * *

[Maintainers Summit](<https://lwn.net/Articles/769260/>)

Improving the quality of stable kernel releases is a perennial subject at the Kernel and Maintainers Summit events, and this year was no exception. This session, led by Fedora kernel maintainer Laura Abbott, discussed a range of ideas but found no silver bullets. There is, it seems, not much that can be done to create better stable kernels except to perform more and better testing. 

Abbott's objective in running this session was to discuss ideas for reducing regressions in stable kernels. Those kernels are, after all, supposed to be stable; if they break, users will suffer and their trust in the entire process will be reduced. In the discussions prior to the summit, she had suggested that perhaps stable releases should sit in a release-candidate state for one week prior to release as a way of shaking out any bugs; that idea was not particularly well received. But we should do something, she said; if we are going to tell people that they should be running stable kernels, those people should not need to employ "an army of engineers" to debug those kernels. The stable kernels we are releasing now, she said, are not ready for production use. 

Peter Zijlstra started the discussion with an assertion that the problem will never be solved. The only way anybody can ever really know that a kernel will work for their particular combination of hardware and production workload is to try it. Rafael Wysocki said that there is a fundamental conflict here: users want fixes to be aggressively applied to stable kernels, but they also want those kernels to be mature. The end [![\[Laura Abbott\]](https://static.lwn.net/images/conf/2018/ms/LauraAbbott-sm.jpg)](<https://lwn.net/Articles/769257/>) result, Jiri Kosina said, is that the distributors are not using the stable kernel releases anymore. 

Ted Ts'o told the group that part of the problem is that the long-term support (LTS) kernels are too successful, so the regular stable kernels are not being used anymore. The support period for those kernels is simply too short. Supporting them for a longer period would help, but that would, of course, increase the amount of work required. So the non-LTS kernels are unlikely to ever be useful for distributors. Those that have tried to use them (he mentioned CoreOS in particular) have ended up shipping regressions to users, who were naturally displeased with what they got. 

Greg Kroah-Hartman, the maintainer for most of the stable kernels out there, noted that CoreOS never told him about the problems it was having, so there was not much that he could have done about them. Other stable-kernel users have a different experience. Google, for example, runs each release candidate through "a zillion tests" and, as a result, is able to push updates out to users quickly. But, it was pointed out, obtaining this kind of result requires operating a large test infrastructure. Linaro is building something like it, Kroah-Hartman said, and Red Hat too. This is the only way the use of stable kernels by a distributor can really work, he said. 

Abbott pointed out that big companies have the resources to put together this kind of infrastructure, but that is not true of all would-be stable-kernel users. Sasha Levin said that the [KernelCI testing project](<https://kernelci.org/>) is evolving to the point where small groups should be able to make use of it. Kroah-Hartman said that KernelCI is a Linux-Foundation project now, and that it is working to add more tests; Mark Brown cautioned that KernelCI still needs resources to be able to grow, though. ~~and that it's a bit too soon to advertise it as being ready for widespread use~~

When Ts'o asked Abbott about the bugs reported by Fedora users, she replied that most of them turn up either in the graphics drivers or the KVM virtualization subsystem. Graphics, she noted, has been getting better recently; Kroah-Hartman replied that KVM is "a black hole" in this regard. Linus Torvalds said that Intel graphics, in particular, has improved a lot recently, but there is more to graphics than Intel. Abbott added that AMD graphics seems to be the source of many recent regressions. 

Returning to one of her original points, Abbott asked whether companies need to be active in the kernel community to be able to use the stable releases effectively; Kroah-Hartman responded that not all users are active kernel contributors. Zijlstra said that companies don't need experts; they just need to test their workloads on the release candidates and report any bugs they find. Ts'o thought that the core problem might be a documentation issue; if users knew that they needed to test the release candidates, they might do more of it. 

Kees Cook, instead, said that if the community is seeing holes that bugs are slipping through, the right response would be to add tests that might catch them — assuming such tests exist. Paul McKenney pointed out that a lot of the existing tests out there are proprietary; in such cases, it's up to the company that owns the tests to run them and report the results. Some companies do indeed do that, Kroah-Hartman said. 

Arnd Bergmann observed that more patches seem to be going into the stable releases than was once the case. Kroah-Hartman said that a lot of work has gone into getting maintainers to tag fixes for the stable releases; that work is bearing fruit. But, Bergmann said, many of those "fixes" appear to be bending the rules that had been put in place for the stable kernels. The rules, Kroah-Hartman responded, are there to allow the maintainers to say "no" to specific patches, but he will generally accept a much broader range of patches for stable releases if the maintainers agree. Bergmann asked whether the rules stretch to adding fixes for warnings generated by new compilers; Kroah-Hartman said "no", that the line has to be drawn somewhere. Fixes to disable those warnings in stable-kernel builds might be accepted, though. 

Toward the end, Kroah-Hartman was asked if he uses the "Fixes" tag to select patches for backporting to the stable releases; he answered that he does not have the time to do that. Levin's [automatic patch-selection code](<https://lwn.net/Articles/764647/>) can make use of it, though. Ts'o said that he has started getting CVE numbers for applicable patches for a novel reason: the presence of a CVE number will cause others to do the work backporting the patches to older kernels for him. With regard to the original topic, though, the conclusion reached by the group was clear enough: if we want better stable-kernel releases, there is really no substitute for better testing. 

[Thanks to the Linux Foundation, LWN's travel sponsor, for supporting my travel to the Maintainers Summit.]
