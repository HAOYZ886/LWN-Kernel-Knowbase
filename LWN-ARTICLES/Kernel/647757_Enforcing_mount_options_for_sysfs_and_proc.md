---
title: Enforcing mount options for sysfs and proc
url: https://lwn.net/Articles/647757/
date: "June 10, 2015"
category: Sysfs; proc
author: "By Jake Edge June 10, 2015"
---

By **Jake Edge**  
June 10, 2015

The sysfs and proc filesystems do not contain executables, setuid programs, or device nodes, but they are typically mounted with flags (i.e. `noexec`, `nosuid`, and `nodev`) that disallow those types of files anyway. Currently, those flags are not enforced when those filesystems are mounted inside of a namespace (i.e. the mount will succeed without those flags being specified). Furthermore, a sysfs or proc filesystem that has been mounted read-only in the host can be mounted read-write inside a namespace, which is a bigger problem. The others are subtle security holes, or they could be, so Eric Biederman is trying to close them, though it turns out that the fixes break some container code. 

In mid-May, Biederman [posted](<https://lwn.net/Articles/647755/>) a patch set meant to address the problems, which boil down to differences in the behavior of mounting these filesystems inside a container versus bind-mounting them. In the latter case, any restrictions the administrator has placed on the original proc and sysfs mounts will be enforced on the bind mounts. If, instead, those filesystems are directly mounted inside of a user namespace, those restrictions won't be enforced. The problem is largely moot, at least for now, for executables, setuid programs, and device nodes, but that is not so for the read-only case. If the administrator of the host has mounted `/proc` as read-only, a process running in a user namespace could mount it read-write to evade that restriction, which is clearly a problem. 

But Biederman was well aware that he might be breaking user-space applications by making this change. In particular, he was concerned about [Sandstorm](<https://sandstorm.io/>), [LXC](<https://linuxcontainers.org/>), and [libvirt LXC](<https://libvirt.org/drvlxc.html>), all of which employ user namespaces. So he put out the patches for testing (and comment). 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

That led to two reports of breakage, the [first](<https://lwn.net/Articles/647784/>) from Serge Hallyn about a problem he found using the patches with LXC. The LXC user space was not passing the three flags that restrict file types allowed for sysfs, which caused the `mount()` call to fail with `EPERM` due to Biederman's changes. The [fix](<https://lwn.net/Articles/647789/>) for LXC is straightforward but, as Andy Lutomirski [pointed out](<https://lwn.net/Articles/647790/>), Biederman's change is an ABI break for the kernel. Given that there aren't executables or device nodes on sysfs or proc, dropping enforcement of those flags from the patch would not have any practical effect, Lutomirski argued. 

Sandstorm lead Kenton Varda [suggested](<https://lwn.net/Articles/647791/>) that instead of returning `EPERM`, `mount()` should instead ignore the lack of those flags when the caller has no choice in the matter: 

That is, in cases where mount() currently fails with EPERM when not given, say, MS_NOSUID, it should instead just pretend the caller actually set MS_NOSUID and go ahead with a nosuid mount. Or put another way, the absence of MS_NOSUID should not be interpreted as "remove the nosuid bit" but rather "don't set the nosuid bit if not required". 

As Varda noted, that would fix the problem without LXC needing to change its code. He also thought it would be less confusing than getting an `EPERM` in that situation. Neither Biederman nor Lutomirski liked the implicit behavior that Varda suggested, however. 

It turns out that libvirt LXC has a similar problem, as [reported](<https://lwn.net/Articles/647796/>) by Richard Weinberger. It is mounting `/proc/sys`, but not preserving the mount flags from `/proc` in the host, thus the `mount()` was failing. Once again, there is a [simple fix](<https://lwn.net/Articles/647797/>). 

Lutomirski [suggested](<https://lwn.net/Articles/647798/>) removing the `noexec`/`nosuid`/`nodev` part, but keeping the read-only enforcement, to avoid the ABI break. Biederman [disagreed](<https://lwn.net/Articles/647801/>) with that approach. It may not matter _now_ that proc and sysfs are mounted that way, but it has [mattered in the past](<https://lwn.net/Articles/191954/>) and could again in the future: 

So I am leaning towards enforcing all of the mount flags including nosuid, noexec, and nodev. Then when the next subtle bug in proc or sysfs with respect to chmod shows up I will be able to sleep soundly at night because the mount flags of those filesystems allow a mitigation, and I did not [sabotage] the mitigation. 

Plus contemplating code that just enforces a couple of mount flags but not all of [them] feels wrong. 

He did want to avoid breaking LXC and libvirt LXC, though, at least until those programs could be fixed and make their way out to users over the next few years. So Biederman added a [patch](<https://lwn.net/Articles/647803/>) that relaxed the requirement for `noexec` and `nosuid` (`nodev` turns out to be a non-issue due to other kernel changes), but printed a warning in the kernel log. Since it is a security fix (though not currently exploitable), he targeted the stable kernels with the fix too. However, Greg Kroah-Hartman [pointed out](<https://lwn.net/Articles/647805/>) that adding warnings for things that have been working just fine is not acceptable in stable kernels. 

Though others disagree, Biederman does not see his changes as breaking the ABI. They do cause a behavior change and break two user-space programs (at least that are known so far), however. He would prefer not to break those programs, so the warning is kind of a stop-gap measure, he [argued](<https://lwn.net/Articles/647807/>). The changes are fixing security holes, though, even if it appears they are not exploitable right now: 

Given that I have not audited sysfs and proc closely in recent years I may actually be wrong. Those bugs may actually be exploitable. All it takes is chmod to be supported on one file that can be made executable. That bug has existed in the past and I don't doubt someone will overlook something and we will see the bug again in the future. 

As it stands, the changes will still allow current LXC and libvirt LXC executables to function (though the version targeting the mainline will warn about that kind of use). Biederman plans to get it into linux-next, presumably targeting 4.2. After that, he plans to remove the warning and enforce the mount options in a subsequent kernel release. It is a bit hard to argue that either of the two broken programs were actually doing what their authors intended in the `mount()` calls, even though it worked. Assuming no other breakage appears, that might be enough to get this patch added without triggering Linus Torvalds's "no regression" filter.
