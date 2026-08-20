---
title: The trouble with dropping groups
url: https://lwn.net/Articles/621612/
date: "November 19, 2014"
category: "Namespaces-User namespaces; Negative groups"
author: "By Jonathan Corbet November 19, 2014"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
November 19, 2014

Linux, like all but the earliest of Unix-like systems, allows a process to be a member of multiple groups at any given time. Any of those group memberships can be used to make access-control decisions. Changing the list of groups a process belongs to is a privileged operation for obvious reasons. Recently, though, a developer started to think about letting processes drop some groups from their lists. On its face, allowing a process to discard credentials seems like it should be a safe thing to do. In practice, it turns out that there are some surprises waiting for anybody wanting to give the idea a try. 

Josh Triplett would like to make it easy for unprivileged users to run programs in a sandboxed mode. As part of that work, he put together [a patch](<https://lwn.net/Articles/621617/>) allowing an unprivileged process to drop its membership in one or more groups. The idea is that a user could fire off a sandboxed process with minimal credentials, including a reduced set of group memberships, without having to resort to privileged helper commands. That should reduce the level of privilege in the system overall and, hopefully, make things more secure. 

But it seems that no Unix-like system anywhere has made it possible for unprivileged processes to drop group membership. The reason may be surprising, but it's there nonetheless: some systems use group membership as a way of _reducing_ privilege; such schemes would no longer work if users could discard groups at will. 

Casey Schaufler [jumped in](<https://lwn.net/Articles/621624/>) early with the assertion that Tizen is one of the systems using groups in this way. The specific mechanism he is worried about is access control lists (ACLs). An ACL can make either positive or negative access decisions, so one can write an ACL to deny access to a resource if the accessing process is a member of a given group. It makes sense in a way; user applications could be run with membership in a special "untrusted" group that would be denied access to most system resources. Casey was pretty clear in his opinion that this change should not be merged: 

This is going to introduce a whole class of vulnerabilities. Don't even think of arguing that the reduction in use of privilege will make up for that. Developers have enough trouble with the difference between setuid() and seteuid() to expect them to use group dropping properly. 

Ted Ts'o [pointed out](<https://lwn.net/Articles/621626/>) that there is no need to have ACLs to use groups as a negative access indicator. Imagine you had a directory called `games` that you wanted to make available to all users except those in the `games-abusers` group. All that is needed is a command sequence like: 

```
chown bin.games-abusers games
        chmod 705 games
```

This will have the (possibly surprising) effect of blocking access to the directory for anybody in the `games-abusers` group — the "no access" group permission bits override the more open permissions for the whole world. Once again, being able to drop group membership would defeat this kind of mechanism. 

Finally, the sudo utility can also make decisions based on group membership or the lack thereof. Being able to drop group membership could thus enable a user to get privileges via sudo that would normally have been denied. 

As it happens, on a Linux system one does not actually need Josh's patch to be able to drop group membership. As Andy Lutomirski [noted](<https://lwn.net/Articles/621627/>), user namespaces already make that possible. In a user namespace, normal users can have root access and can thus call `setgroups()`. In theory, that access does not enable any expanded privilege outside of the namespace, but, as can be seen here, a few surprises still lurk in that code. Beyond `setgroups()`, though, user namespaces can be used to drop groups by simply neglecting to map them to groups outside of the namespace; see [this article](<https://lwn.net/Articles/532593/>) for details on how this mapping works. Andy sees the problem as being serious enough that he [reported](<https://lwn.net/Articles/621629/>) it to the oss-security mailing list as a vulnerability with no fix available. 

A few of the participants in the discussion seemed to feel that the idea of using credentials to reduce privilege was a bit backward. But it appears to be something that people do, so breaking it is not an option. For the case of user namespaces, some sort of fix will have to be applied; it may become impossible to drop group memberships from within a user namespace. The sudo problem can be addressed by only allowing groups to be dropped if the "no new privileges" flag (originally introduced for [system call filtering](<https://lwn.net/Articles/475678/>)) is set, but Eric Biederman [worries](<https://lwn.net/Articles/621631/>) about the additional complexity that would bring in. 

There was talk of adding a sysctl knob to control the unprivileged dropping of group membership. Such a flag would default to "off"; system administrators could turn it on if they were confident that it would not subvert the security models in use on their systems. But Casey [is not confident](<https://lwn.net/Articles/621635/>) that this option makes sense; just because a system does not use groups to restrict privilege now doesn't mean that somebody won't install a package using that approach tomorrow. Also, as he pointed out: ""The developers of user namespaces didn't notice it might be a problem. You can't count on sysadmins or distro developers to do better."" 

So, in the end, unprivileged dropping of group membership may turn out to be one of those ideas that just can't quite be shoehorned into the decades-old Unix privilege model. There is a lot of history there and no end of systems that might see surprising results coming from a change like this. If this work does go forward, expect to hear some loud complaints before it makes it into the mainline. 

(Those interested in this work may also want to have a look at [Josh's other patch](<https://lwn.net/Articles/621580/>). It allows a process to have multiple user IDs in the same way that multiple group IDs are possible now. The idea is that the process could use those supplemental IDs to run sandboxed processes each with their own user ID.)
