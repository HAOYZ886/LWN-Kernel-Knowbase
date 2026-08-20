---
title: "The twilight of the version-1 memory controller"
url: https://lwn.net/Articles/974575/
date: "May 23, 2024"
category: "Control groups-Memory controller; Memory management-Control groups"
author: "By Jonathan Corbet May 23, 2024 LSFMM+BPF"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
May 23, 2024

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2024/>)

Almost immediately after the merging of [control groups](<https://lwn.net/Articles/603762/>), kernel developers set their sights on reimplementing them properly. The second version of the control-group API started trickling into the kernel around the 3.16 release in 2014 and users have long since been encouraged to migrate, but support for (and users of) the initial API remain. At the [2024 Linux Storage, Filesystem, Memory-Management and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>), memory-management developers discussed whether (and when) it might be possible to remove the version-1 memory controller. The session was led by Shakeel Butt and (participating remotely) Roman Gushchin. 

#### Deprecation process

The first step toward an eventual deprecation is to move the version-1 code into a separate file with its own configuration option. That option would also control the presence of some internal structure fields. Michal Hocko immediately suggested making the old version disabled by default; if it remains enabled, he said, the community will never manage to get rid of it. There are, he said, two classes of users for the old interface: intentional users who have a reason to stick with it, and accidental users who are unaware that a better interface exists. Disabling the old interface will motivate the second group to migrate away from it. 

[![\[Shakeel Butt\]](https://static.lwn.net/images/conf/2024/lsfmm/ShakeelButt-sm.png)](<https://lwn.net/Articles/974586/>) Most distributions, Butt continued, are using systemd these days, so they could easily handle a deprecation of the old interface. Hocko cautioned, though, that there are a lot of containers out there stuck on version 1 that nobody has ever bothered to fix. David Hildenbrand said that there was no need to worry about distributions; they will enable this option if it is needed. Gushchin said that, if the option is disabled by default, kernel developers will have to pay special attention to avoid breaking it while making changes elsewhere. 

The proposed deprecation process, once the code separation is done, involves adding a warning to be emitted when the old interface is used; that code would be backported to the stable kernels as well. The next step is "wait a while", defined as two or three long-term-support cycles (referring to the long-term stable kernels, which have a one-year cycle). After that, the interface would remain, but it would not actually do anything; the code behind it would be removed. 

Hocko worried that, no matter how long the warning period is, it would not be enough. He also said that turning the interface into a no-op is a risky approach that could cause systems to fail silently. Rather that do that, he suggested just setting the version 1 code aside and letting it slowly decay. David Rientjes said that this is the real point: how should features be deprecated? It is necessary to make users take some manual action to continue using the deprecated feature, or it will never go away; he suggested adding a sysctl knob to enable the old interface. 

A participant pointed out that there are still features provided by the old interface that are unavailable in version 2\. At the top of the list was combined accounting of memory and swap usage; without that, he said, applications simply cannot know how much swap space they need. Gushchin said that there are a number of version-1 features that are no longer used, and that not all features are equally important. The best approach might be to deprecate only those features; that would reduce the pressure to get rid of the rest. 

#### Features specific to version 1

That led to a discussion of specific version-1 features, most of which are tersely described in [this document](<https://docs.kernel.org/admin-guide/cgroup-v1/memory.html>). The first on the chopping block was the `move_charge_at_immigrate` knob, which controls how accounting for memory is done. A deprecation warning was added in the 6.3 kernel and backported to older ones; should it be turned into a no-op this year? Hocko again wondered if that was the right approach, saying that it might be better to simply fail if somebody tries to use that knob. One way or the other, it was agreed that this feature makes maintenance of the memory controller harder and should be removed. 

Then, there is TCP memory accounting, which is controlled by four knobs. This is a separate, opt-in accounting feature. Butt made the claim that nobody is using TCP memory accounting, its performance is terrible, and that the version-2 implementation is far better. Nobody disagreed with that assessment. This set of knobs should be relatively easy to remove; the group agreed to start the deprecation process for them. 

Next is soft limits, which are controlled by the `soft_limit_in_bytes` knob. The old version is broken (and disabled) for systems using realtime preemption. The version-2 API has better-defined semantics, and provides both best-effort and hard protections. Nobody objected to the removal of this feature either. 

The `failcnt` interfaces can be read to see how many times a given control group has run into its limits; they are not exposed in the version-2 interface, and it is not clear that anybody is using them. It would be easy to add `failcnt` to version 2, but there should be a use case defined first. Hocko said that this feature is not useful, but it is almost free to support and not worth the trouble to remove. 

There are a number of notification variables (including `usage_in_bytes` and `oom_control`) that notify a registered user when usage goes above a given threshold. They are disabled for realtime, and are not useful for driving the behavior of a process since notification happens before reclaim. But, evidently, Google uses them internally for job control and exposes them to workloads there. This functionality could be had with BPF, but applications would have to explicitly migrate over to that approach. 

The `oom_control` knob also allows disabling the out-of-memory (OOM) killer and reading its status. Its presence enables the creation of user-space OOM killers. The new API provides some of this functionality via the `memory.events` knob, but does not give a way to disable the OOM killer. The version 2 `memory.high` knob ([documented here](<https://docs.kernel.org/admin-guide/cgroup-v2.html#memory-interface-files>)) can be used to similar effect, though perhaps less reliably. Johannes Weiner said that Meta is using it that way, and it works; evidently Android also uses `memory.high` for this purpose. 

Hocko said that the `oom_control` knob has been broken for years. It only controls OOM handling in the page-fault path. It is not a big deal to support, he said, since the overhead is small, but nothing like it should be provided in the version 2 API. There _is_ a need for better control over the OOM killer, he said; perhaps that could be provided as a hook for a BPF program. That approach would allow controlling the OOM killer globally as well. 

The next version 1 feature considered was memory-pressure notifications. This feature is not reliable, it assumes that there is reclaimable memory, which might not be the case. Unfortunately, the network-memory-pressure notification has leaked into the version 2 interface. The [pressure-stall information API](<https://lwn.net/Articles/759781/>) is sufficient for most use cases, but there does need to be an alternative for network-memory pressure in particular. 

Toward the end of the session, attention turned to the combined accounting of memory and swap usage. This feature has been an area of concern for some time; it was [discussed](<https://lwn.net/Articles/753162/>) at the 2018 summit. Google is still using this feature, though, and there does not seem to be a way to create a good replacement. Hocko said that he hoped Google would eventually move to the version 2 interface for the "great features" it provides, and will find a way to move to a newer swap model in the process. There was a suggestion for a "`-google`" mount option for the cgroupfs filesystem to make this feature appear in version 2, but Hocko said that would cause it to never go away. 

The final knob discussed was `swappiness`, which controls the relative attention paid by the reclaim mechanism to anonymous and file-backed pages. Hocko said that users complain that the knob doesn't work; it can be changed, but the changes do not propagate through the control-group hierarchy, creating confusion. He would rather not see that confusion repeated in the version 2 interface. Weiner disagreed, though, saying that it is possible to define good hierarchical semantics for `swappiness`. Before proceeding, though, it will be necessary to define the use cases for this knob.
