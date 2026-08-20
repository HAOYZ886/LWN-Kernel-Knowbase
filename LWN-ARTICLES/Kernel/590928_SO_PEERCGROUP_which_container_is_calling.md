---
title: "SO_PEERCGROUP: which container is calling?"
url: https://lwn.net/Articles/590928/
date: "March 18, 2014"
category: Containers
author: "By Jonathan Corbet March 18, 2014"
---

By **Jonathan Corbet**  
March 18, 2014

As various container solutions on Linux approach maturity, distribution developers are thinking more about the infrastructure needed to manage a system full of containers. Toward that goal, Vivek Goyal recently posted [a patch](<https://lwn.net/Articles/590910/>) allowing a process to determine which control group contains a process at the other end of a Unix-domain socket. The patch is relatively simple, but it still kicked off a lengthy discussion making it clear that, among other things, there is still resistance to using modern Linux kernel facilities to implement new features. 

The patch in question adds a new command (`SO_PEERCGROUP`) to the `getsockopt()` system call. A process can invoke this command on an open Unix-domain socket and get back the name of the control group containing the process at the other end. Or something close to that: what is returned is the control group the peer process was in when the connection was established; that process may have moved in the meantime. The information may thus be a bit outdated, but `SO_PEERCGROUP` mirrors the existing `SO_PEERCRED` command in this regard. Connection-time information is deemed to be good enough for the targeted use case, which is allowing the [system security services daemon (SSSD)](<https://lwn.net/Articles/457415/>) to make policy decisions based on which container it is talking to. 

The main critic of this patch was Andy Lutomirski, who had a number of complaints with it. In the end, though, the key point may have been described in [this message](<https://lwn.net/Articles/590933/>): 

My a priori opinion is that this is a terrible idea. cgroups are a nasty interface, and letting knowledge of cgroups leak into the programs that live in the groups (as opposed to the cgroup manager) seems like a huge mistake to me. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Part of this complaint was a bit off the mark: the idea is to not require awareness of control groups for processes running inside containers. But, even without that, Andy appears to be against the use of control groups in general. He is certainly not alone in that point of view. 

Andy came up with [three alternative approaches](<https://lwn.net/Articles/590984/>) by which a daemon process could identify which container is connecting to it, but those have run into resistance as well. The first of those was to put the containers inside [user namespaces](<https://lwn.net/Articles/532593/>). The user-ID mapping performed by user namespaces would then allow each connecting process to be identified with the existing `SO_PEERCRED` mechanism or with an `SCM_CREDENTIALS` control message. Adding user namespaces to the mix should also make containers more secure, he said. 

The objection to this approach was best [summed up](<https://lwn.net/Articles/590985/>) by Vivek: 

Using user namespaces sounds like the right way to do it (at least conceptually). But I think hurdle here is that people are not convinced yet that user namespaces are secure and work well. IOW, some people don't seem to think that user namespaces are ready yet. 

Simo Sorce [echoed](<https://lwn.net/Articles/590986/>) these concerns and also added that he is not in a position to make the target container mechanism (Docker) use user namespaces. Eric Biederman, the developer of user namespaces, [asked for specifics](<https://lwn.net/Articles/590987/>) of any problems and observed: ""It seems strange to work around a feature that is 99% of the way to solving their problem with more kernel patches."" 

Strange or not, there does not appear to be a lot of interest in exploring the use of user namespaces as a solution to this particular problem. Like control groups, user namespaces are a relatively new, Linux-specific mechanism; getting developers to adopt such features is often a challenge. In this case, concerns about a lack of maturity can only serve to deprive user namespaces of testing, prolonging any such immaturity further. 

Andy's second suggestion was to get the container information out of `/proc`, using the process ID of the connecting process. Simo responded that use of process IDs can suffer from race conditions; processes can come and go quickly on some systems. The third idea was to just keep a separate socket open into each container; this idea was dismissed as being on the messy and inelegant side, but nobody said that it wouldn't work. 

The end result was a conversation that, by all appearances, convinced nobody. In the process, it has highlighted a question that often comes up in the kernel community: once we add interesting new features, to what extent can we integrate those features with others or expect developers to use them? Expect to see this kind of debate more often as the kernel continues to develop and acquires more features that were never envisioned by any of the Unix standards bodies. A lot of work is going into adding new capabilities to the kernel; it would seem strange if we were so unconvinced by our own work that we did not expect others to make use of it.
