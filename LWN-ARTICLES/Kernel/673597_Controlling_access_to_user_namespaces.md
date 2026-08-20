---
title: Controlling access to user namespaces
url: https://lwn.net/Articles/673597/
date: "January 27, 2016"
category: "Namespaces-User namespaces; Security-Namespaces"
author: "By Jonathan Corbet January 27, 2016"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
January 27, 2016

The [user namespaces](<https://lwn.net/Articles/532593/>) feature holds an interesting promise for system security: users can be confined within a namespace, given full root privileges within that namespace, and still be unable to adversely affect the system as a whole. The path to better security has, perhaps predictably, proved to be a bit rocky, however. In response, there is now an effort to make the feature configurable by system administrators, but this new configuration knob is proving to be a harder sell than one might expect. 

User namespaces are created by passing the `CLONE_NEWUSER` flag to the `clone()` or `unshare()` system calls. Administrators who are nervous about allowing access to this feature currently only have one option: configure out support at kernel build time. That option is not easily available to the many systems running distribution-built kernels, though. Kees Cook set out to create an easier way with [this patch set](<https://lwn.net/Articles/673261/>) creating a new sysctl knob to control access to the user-namespace feature, saying: 

There continues to be unexpected side-effects and security exposures via CLONE_NEWUSER. For many end-users running distro kernels with CONFIG_USER_NS enabled, there is no way to disable this feature when desired. As such, this creates a sysctl to restrict CLONE_NEWUSER so admins not running containers or Chrome can avoid the risks of this feature. 

In particular, the patch adds a knob called `/proc/sys/kernel/userns_restrict`. When it is set to the default value (zero), user namespaces are unrestricted. Setting it to one allows only privileged users to create user namespaces; a setting of two disables user namespaces altogether. In that final case, it is not possible to re-enable user namespaces without rebooting the system. 

One of the first issues to be aired had to do with naming: it turns out that Debian currently carries a similar patch, but, on Debian systems, the knob is called `unprivileged_userns_clone` and doesn't support the "privileged users only" setting. Ben Hutchings [agreed](<https://lwn.net/Articles/673602/>) that the new naming was probably better and said that, should Kees's patch go upstream, Debian would slowly move over to it. 

Some developers worried that allowing user namespaces to be turned off would slow the process of finding and fixing any remaining security issues. Additionally, Serge Hallyn [suggested](<https://lwn.net/Articles/673604/>) that, if application developers could not count on the availability of user namespaces, they wouldn't use them at all. He suggested that, if the knob is accepted, it be marked as a short-term workaround that would eventually be removed. 

The strongest opposition, though, came from Eric Biederman, the creator of user namespaces and also the developer who has done the most work on the sysctl code in recent times. He [stated](<https://lwn.net/Articles/673610/>) flat out that ""the code is buggy, and poorly thought through"" and would not be merged. In [another message](<https://lwn.net/Articles/673611/>) he described his objections in detail, starting with a challenge to the idea that user namespaces are a security risk at all: 

I don't actually think there do continue to be unexpected side-effects and security exposures with CLONE_NEWUSER. It takes a while for all of the fixes to trickle out to distros. At most what I have seen recently are problems with other kernel interfaces being amplified with user namespaces. 

Others, though, seem to think that, if problems elsewhere are being "amplified," there is indeed a security exposure. Andy Lutomirski [described](<https://lwn.net/Articles/673613/>) some concerns of his own: 

I consider the ability to use CLONE_NEWUSER to acquire CAP_NET_ADMIN over /any/ network namespace and to thus access the network configuration API to be a huge risk. For example, unprivileged users can program iptables. I'll eat my hat if there are no privilege escalations in there. 

Eric echoed the point that making it possible to disable user namespaces would be a net loss in security, since the feature would not be available on all systems. He cited web browsing with Chrome as a use case; Kees [responded](<https://lwn.net/Articles/673614/>) that this patch wasn't really aimed at desktop systems in the first place. 

Next on Eric's list was a complaint that a system-wide knob was too coarse; he suggested that perhaps the `seccomp()` mechanism should be used instead if access to user namespaces must really be restricted. Kees's answer here is that it's not really possible to set a global `seccomp()` policy, that performance would suffer in any case, and that `seccomp()` is meant for developers to use rather than system administrators. ""It's an extraordinarily big hammer for wanting to turn off a single area of the kernel with a long history of problems."" He noted that trying to use a Linux security module to achieve this end would have a number of similar problems. 

Then, Eric said, the sysctl knob could create ""a false sense of security"" since it would have no effect on processes that are already running in a user namespace. If a security issue comes to light, just turning off the knob will not be enough to protect a system; a reboot will also be necessary. Eric [returned](<https://lwn.net/Articles/673618/>) to this point later, calling the patch ""fatally flawed"" as a result of the ""subtlety and nuance"" involved in using it. 

Kees [acknowledged](<https://lwn.net/Articles/673620/>) the "corner case" in the sysctl implementation, one that, he said, applies to a number of other, existing knobs as well. But, he said, it really does not matter to an administrator who simply wants to disable the feature outright as a way of reducing the attack surface of a system. Even so, he allowed: ""I'm open to having this sysctl kill all CLONE_NEWUSERed process trees"", without noting that having a sysctl knob kill off processes might pose some interesting "subtlety and nuance" of its own. 

As a sort of postscript, Eric suggested that, perhaps, the desired restriction could be implemented as a resource limit controlling the number of user namespaces that any user would be allowed to create. Setting that number to zero would effectively disable the feature. Kees indicated a willingness to look at this idea; it is the end result he wants, rather than the sysctl knob itself. 

There is an evident desire for the ability to turn off access to user namespaces; various other developers spoke in its favor over the course of the discussion. But this desire is clearly not universal and, as a result, the current patches do not appear to have an easy path into the mainline. It is entirely possible that the concerns blocking this feature may eventually be addressed and overcome, but it also seems possible that, in the end, this knob ends up being part of the patch set carried by distributors and users. It seems that getting security-related changes into the kernel is still a difficult task.
