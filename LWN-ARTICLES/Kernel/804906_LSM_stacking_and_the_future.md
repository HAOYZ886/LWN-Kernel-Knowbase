---
title: LSM stacking and the future
url: https://lwn.net/Articles/804906/
date: "November 20, 2019"
category: "Security-Security modules"
author: "By Jake Edge November 20, 2019 LSS EU"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
November 20, 2019

* * *

[LSS EU](<https://lwn.net/Archives/ConferenceByYear/#2019-Linux_Security_Summit_Europe>)

The idea of stacking (or chaining) Linux security modules (LSMs) [goes back 15 years](<https://lwn.net/Articles/114588/>) (at least) at this point; [progress](<https://lwn.net/Articles/635771/>) has definitely been made along the way, especially in the last decade or so. It has been possible to stack "minor" LSMs with one major LSM (e.g. SELinux, Smack, or AppArmor) for some time, but mixing, say, SELinux and AppArmor in the same system has not been possible. Combining major security solutions may not seem like a truly important feature, but there is a use case where it is pretty clearly needed: containers. Longtime LSM stacker (and Smack maintainer) Casey Schaufler gave a presentation at the [2019 Linux Security Summit Europe](<https://events19.linuxfoundation.org/events/linux-security-summit-europe-2019/>) to report on the status and plans for allowing arbitrary LSM stacking. 

LSMs allow adding more restrictions to Linux than those afforded by the traditional security policies. For the most part, those policies reflect the existing mechanisms, such as permissions bits on files. But there are also other security concerns, such as binding to a network socket, that are outside of the usual permissions, so mechanisms to restrict access to them have been added to the LSM interface. 

[ ![\[Casey Schaufler\]](https://static.lwn.net/images/2019/lsseu-schaufler-sm.jpg) ](<https://lwn.net/Articles/805190/>)

Prior to the advent of the [Yama LSM](<https://www.kernel.org/doc/Documentation/security/Yama.txt>), only one security module could be active in a running kernel; Yama was originally manually stacked, which "didn't really sit very well". To support adding the Yama restrictions on top of other LSMs in a dynamic fashion, lists of modules were added to the kernel, which would allow multiple LSMs to be active. But there was a problem for the "bigger" LSMs that need security "blobs"—context data associated with various kernel objects—in which to store their state. There is only one pointer available to use, so only one blob-using LSM could be active, though multiple minor LSMs that did not need the blobs could be stacked with it. 

At this point, an LSM attribute has been added to tag LSMs; `LSM_FLAG_EXCLUSIVE`. The "exclusive" tag is applied to the blob-using LSMs: SELinux, Smack, and AppArmor. The idea is to remove that tag from those LSMs over time. 

There can only be one exclusive LSM active in a running kernel. "That's bad", Schaufler said, but for a long time that was not seen as a serious problem. That was before containers became so widespread, however. Now there are some people who run, for example, Ubuntu in their data centers (with AppArmor) and who want to run Android (SELinux) containers on top. So the goal of the work he and others have been doing is to get rid of the exclusive bit for "as many modules as we possibly can". 

The 5.1 kernel added "infrastructure-managed blobs" for a number of different kernel objects: tasks, credentials, files, inodes, and the System V interprocess-communication mechanisms (semaphores, shared memory, and message queues). An LSM will tell the kernel how much space it needs to store its information and the kernel will take care of allocating, managing, and freeing the blob. So, any LSM that only uses blobs on those object types can be marked as non-exclusive at this point. 

That means a variety of LSMs can be used alongside SELinux, so "the IT people are really happy" since SELinux does not have to be turned off to get the protections afforded by some other module that only uses those blobs. There are also a number of smaller LSMs that are headed toward the mainline that could benefit from this. Those, or some custom module, can be run with one of the exclusive LSMs, mostly without interference; so "everybody's happy", he said. 

#### Next up

"But not everybody's happy", he continued, because there are still limitations, which leads to the plans for an upcoming kernel, possibly 5.5. The [code to remove the exclusive flag for AppArmor](<https://lwn.net/ml/netdev/20191119224714.13491-1-casey@schaufler-ca.com/>) is basically ready. AppArmor is different than Smack and SELinux, "in that it is path-name-based-ish", though it is less so now than it used to be. It has a different fundamental security model; Smack and SELinux are both based on subjects and objects, while AppArmor mostly focuses on path names. The use cases for AppArmor are also different than those of the others, so it makes sense to start with it. 

In order to make non-exclusive stacking work for AppArmor, kernel socket-object security blobs have to join the other infrastructure-managed blobs so that multiple LSMs can have them. That is fairly easily done, since it already has been done for the other objects. There are also more difficult pieces; when you get to those, that's where people start to bikeshed, he said. 

The first problem is sharing `/proc/PID/attr/current`, which is used by AppArmor and other major LSMs to report the security context for the process identified by PID. So SELinux and AppArmor would both want to put their contexts in that file, but that is impossible. Similarly, the `SO_PEERSEC` socket option to retrieve the security context of the other endpoint of a Unix socket also cannot be shared. The solution for both is to introduce a new interface, so that the existing interfaces stay backward compatible. 

A number of different ideas for the format of `/proc/PID/attr/context` and `SO_PEERCONTEXT` (the new interfaces) were proposed along the way, but the developers "finally did the intelligent thing" and asked the user community, the D-Bus developers in particular. They suggested a simple string with pairs of null-terminated strings of the form "LSM-name\0value\0"; the full length of the string will be known, so pulling out the individual LSM contexts will be straightforward. There is something of a lesson there, Schaufler said: instead of debating something like this, ask the people who will be using the information. 

But adding interfaces doesn't really solve the problem, since there are numerous system utilities that will use the existing interfaces—and for a long time to come. So there is a new `/proc/self/attr/display` setting that can be used to determine which LSM's context information is reported via the existing interfaces. An SELinux container could set its `display` to ensure that the container sees the SELinux information even if it is running on a kernel with AppArmor active as well; the rest of the system could set the `display` to AppArmor so that it would look like that was the only LSM active. 

The permissions required to change the `display` attribute also needed to be worked out. He thought there should be no checks on switching the value, but SELinux developer Stephen Smalley came up with some problems with that approach. So Schaufler suggested requiring the `CAP_MAC_ADMIN` capability, but it turns out that SELinux developers do not want to rely on capabilities, they want SELinux to be able to weigh in on the choice. So there is now a hook for `display` changes; SELinux and AppArmor have added ways to set a policy for changing `display`, while Smack just says "sure, go ahead". 

It turns out that Android's binder security mechanism also uses the contexts, so the code needed to ensure that the processes at both ends of the bind see the same context; it doesn't matter which it is, he said, but it needs to be the same. There was also a need to add new audit fields to support subject contexts on a per-LSM basis, while still maintaining the "`subj=`" entries for backward compatibility. The same thing can be done with object contexts (i.e. "`obj=`") if that is needed down the road. 

#### Before too long

The next major step is to remove the exclusive tag entirely, by getting rid of it for Smack and SELinux, so that you can use any set of arbitrary LSMs in the same running kernel That is targeted for the 5.8 kernel or so. It is more challenging, in part because the two LSMs do a lot of the same things; in particular, both interact with the networking subsystem extensively, he said. 

Two more kernel objects, for keys and superblocks, need to be added to the infrastructure-managed list. Part of the reason that superblocks need blobs for Smack and SELinux is that both process mount options, which is a bit messy to do. Instead of simply handing the options to a single LSM, they will need to be sent to a series of LSMs; each LSM needs to only deal with the options it knows about, ignoring those it doesn't, but then any options that are not handled by any LSM need special treatment. 

"The networking stuff has a wonderful set of challenges", Schaufler said. The [NetLabel](<https://www.kernel.org/doc/html/latest/netlabel/introduction.html>) interface is useful to allow an LSM to put [CIPSO](<https://tools.ietf.org/html/draft-ietf-cipso-ipsecurity-01>) or [CALIPSO](<https://tools.ietf.org/html/rfc5570>) labels on packets, but two LSMs cannot put different labels on the same packets. After much "gnashing of teeth", it was decided that unless all of the relevant LSMs could agree on the labels, packet sending would fail. It may be a bit harsh, but it makes sense: "If you can't get people to agree, you probably shouldn't send it". 

The label is set when the socket is created, so that is the operation that should fail, even though it doesn't really matter until a packet is actually sent. Making that work requires some changes in NetLabel and SELinux, but more in Smack. NetLabel is used differently by Smack, which "makes things more complicated", he said. 

The [secmark facility](<https://james-morris.livejournal.com/11010.html>) allows associating a 32-bit number with a packet; it is added to the socket buffer (sk_buff or SKB) object by nftables. However, 32 bits is not enough to be able to handle two, three, or even more LSMs that want to use secmarks. It is not clear what to do about that, yet. A hash-table mapping might work or only allowing a single LSM to use the facility is another option, though "it's kind of a cop-out". Another possibility is an SKB extension, but he is a bit leery of going that route because he anticipates some opposition from the networking developers. 

[Labeled NFSv4](<https://tools.ietf.org/html/rfc7204>) presents an "interesting conundrum", he said. It was defined with a format for the label data that is passed back and forth, which "Linux very carefully ignores". The Linux implementation doesn't add the labels or read them; it just assumes that any data that is there is reasonable for whatever is actually going to use it. The NFS developers are looking into that at this point. 

Schaufler wrapped up by reiterating that the first set of changes for AppArmor are targeting Linux 5.5. The second set needs more work and there are some solutions to be found, but it will hopefully make its way into the mainline in 5.8 or thereabouts. Interested readers can view his [slides [PDF]](<https://static.sched.com/hosted_files/lsseu2019/84/201910-LSS-EU-xxx-Stacking.pdf>) and the [YouTube video](<https://www.youtube.com/watch?v=XtKodXBxY6U>) of the talk. 

[I would like to thank LWN's travel sponsor, the Linux Foundation, for travel assistance to attend the Linux Security Summit Europe in Lyon, France.]
