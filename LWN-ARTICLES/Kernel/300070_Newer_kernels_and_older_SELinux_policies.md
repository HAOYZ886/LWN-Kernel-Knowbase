---
title: Newer kernels and older SELinux policies
url: https://lwn.net/Articles/300070/
date: "September 24, 2008"
category: SELinux
author: "By Jake Edge September 24, 2008"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
September 24, 2008

A subtle change in 2.6.25 recently left Andrew Morton with a less than completely functioning system, but it also demonstrated a user-space interface that may sometimes be overlooked: SELinux. The problem stemmed from a change to facilitate containers by making `/proc/net` into a symbolic link, which tripped up SELinux policies that had been written for earlier kernels. Putting policy into user space is a guiding principle of kernel development, but that can sometimes lead to an unexpected synchronization required between those policies and the kernel. 

The change itself was fairly minor, making `/proc/net` be a symbolic link to `/proc/self/net` so that containers would only see their network devices, rather than those of the enclosing system. But when Morton ran a recent kernel on his Fedora Core 5 and 6 systems, he [got](<https://lwn.net/Articles/300185/>): 

```
sony:/home/akpm> ifconfig -a
        Warning: cannot open /proc/net/dev (Permission denied). Limited output.
```

Further investigation found that even `ls` got permission errors when looking at `/proc/net`. As is usual with mysterious "permission denied" errors, SELinux was the underlying cause. 

When the change was made, back in March, it was reviewed by the SELinux developers, but no one noticed that it would cause an additional permission check--on the symbolic link itself. So, when resolving things like `/proc/net/dev` or other entries in that directory, the "labels" on the symbolic link were checked. Of course, `/proc` is a synthetic filesystem, so the labels are generated from SELinux code rather than retrieved from extended attributes (xattrs). 

Distributions have updated their policies to allow access to the symbolic link--probably by noticing the SELinux denial in log messages--so most folks never saw the problem. As Morton found out, though, existing distribution policy files (those shipped with FC5 and FC6 for example) would still disallow the access. Morton regularly runs newer kernels with older distributions to try to catch exactly this kind of error; he is probably one of very few, perhaps the only one, doing that. 

Because the distribution-supplied kernel was being changed, some argued that requiring users to update their SELinux policies is not an onerous requirement. Paul Moore [puts it this way](<https://lwn.net/Articles/300218/>): 

Maybe I'm in the minority here, but in my mind once you step away from the distro supplied kernel (also applies to other packages, although those are arguably less critical) you should also bear the responsibility to make sure you upgrade/tweak/install whatever other bits need to be fixed. 

Morton did not buy that argument [saying](<https://lwn.net/Articles/300219/>): 

Nope. Releasing a non-backward-compatible kernel.org kernel is a big deal. 

We'll do it sometimes, with long notice, much care and much deliberation. 

We did it this time by sheer accident. That's known in the trade as a "bug". 

But SELinux developer Stephen Smalley points out that permissions checks are not normally considered part of the kernel to user space interface. It is something of a gray area, though. Clearly the standard UNIX permission checks _are_ part of that interface, at least partially because the kernel does handle the policy for those checks. Since the policies that govern the decisions about SELinux access denial come from user space, it is a bit hard to argue that changes to the kernel will not ripple out. Smalley [describes](<https://lwn.net/Articles/300223/>) the problem: 

I should note here that for changes to SELinux, we have gone out of our way to avoid such breakage to date through the introduction of compatibility switches, policy flags to enable any new checks, etc (albeit at a cost in complexity and ever creeping compatibility code). But changes to the rest of the kernel can just as easily alter the set of permission checks that get applied on a given operation, and I don't think we are always going to be able to guarantee that new kernel + old policy will Just Work. 

One possible solution to the immediate problem was floated by Smalley: SELinux could change the label that it returns for symbolic links under `/proc`. It is not clear that anyone really wants that change, and there has been no movement to add it. As Morton [says](<https://lwn.net/Articles/300238/>), ""people who are shipping 2.6.25- and 2.6.26-based distros probably wouldn't want such a patch in their kernels anyway"". 

Longer term, Eric Biederman [asks](<https://lwn.net/Articles/300239/>) about supporting xattrs for `/proc`. That would allow user space to label the proc filesystem appropriately, removing one of the special cases. Unfortunately, doing so would create yet another incompatibility between newer kernels and older user spaces. 

In the end, because the bug was only seen by Morton, many months after it was introduced, it may just be ignored. The larger issue of how permissions checks fit into the kernel to user space interface, though, may rear its head again.
