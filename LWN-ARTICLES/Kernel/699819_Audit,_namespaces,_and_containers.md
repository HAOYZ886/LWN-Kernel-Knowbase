---
title: "Audit, namespaces, and containers"
url: https://lwn.net/Articles/699819/
date: "September 8, 2016"
category: Auditing; Namespaces
author: "By Jake Edge September 8, 2016 Linux Security Summit"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
September 8, 2016

* * *

[Linux Security Summit](<https://lwn.net/Archives/ConferenceByYear/#2016-Linux_Security_Summit>)

Richard Guy Briggs works on the kernel's [audit subsystem](<https://github.com/linux-audit>) for Red Hat and has run into some problems with the interaction between the audit daemon (`auditd`) and namespaces. He gave a report on those difficulties to the [Linux Security Summit](<http://events.linuxfoundation.org/events/linux-security-summit>) in Toronto. In the talk, he also looked at containers and what they might mean in the context of audit. 

[ ![\[Richard Guy Briggs\]](https://static.lwn.net/images/2016/lss-briggs-sm.jpg) ](<https://lwn.net/Articles/699852/>)

Audit was started in 2004, around the same time that the kernel started using Git. It is a "syslog on steroids", he said. Syslog is used a lot for debugging, but audit is meant as a secure audit trail to log kernel and other events in a way that could be used in court. There are configurable filters in the kernel for what events should be logged and it has the `auditd` user-space daemon that can log to disk or the network. 

Audit only reports on behavior; it does not actively interfere with what is going on in the system. There is one exception, though: audit can panic the system if it cannot log its data. 

Briggs then went through a bit of an introduction to [namespaces](<https://lwn.net/Articles/531114/>) in Linux, noting that they are a kernel-enforced user-space view of the resource being namespaced. There are seven different namespaces in Linux; three are hierarchical in nature (PID, user, and control groups), which means their permissions and configuration are inherited from the parent namespace, while the other four are made up of peer namespaces (mount, UTS, IPC, and net). 

He is not sure that anyone actually uses IPC namespaces, he said, but the net namespace is one of the easier ones to understand. Network devices can be assigned to a net namespace from the initial net namespace, so each namespace can have its own firewall rules, routing, and so on. If two namespaces need to talk, a virtual ethernet pair can be set up between them. 

The user namespace has been "the most contentious one so far", as there are a number of "security traps" in allowing unprivileged users to be root within the namespace. Many distributions don't enable the feature by default at this point. The [control groups (cgroups) namespace](<https://lwn.net/Articles/621006/>), which is the most recent namespace (added in 4.6), is meant to hide system-resource limits so that processes only see what resources have been allocated to their cgroup. 

Namespaces are one component of the concept of containers, but there really is no hard definition of containers, Briggs said. In fact, there are almost as many definitions as there are users of containers. There is some general agreement that containers use a combination of namespaces, cgroups, and seccomp to partition some portion of the system into its own world. 

But the kernel has no concept of a container at all. Managing containers is left up to a user-space container-orchestration system of some kind. From an audit perspective, though, there is interest in having some knowledge of containers in the kernel. That might be through some form of "container ID" or simply by collecting up the namespace IDs that correspond to the container. 

#### Problems

With that introductory material out of the way, he turned to the problems, which boil down to a [Highlander](<https://en.wikipedia.org/wiki/Highlander_\(franchise\)>) quote: "There can be only one." The "one" in this case is `auditd`, which runs in the initial namespace but must be reachable from the other namespaces. For the mount, UTS, and IPC namespaces, there have been no problems, but others do have a variety of issues. 

For example, the net namespace partitions netlink sockets (which processes use to talk to the audit subsystem) so that only processes in the initial network namespace can send their audit messages. That broke various container implementations because things like pluggable authentication modules (PAM) would try to write an audit message and get an unexpected error return. Instead of the `ECONNREFUSED` error that it expected when `auditd` cannot be reached, PAM-using programs (e.g. the login process) would get `EPERM` and fail such that users could not log in. The short-term solution for that was to simply "lie" in non-initial namespaces and return the expected error message so that user-space programs do not break. 

For PID namespaces, the problem cropped up with [vsftpd](<https://security.appspot.com/vsftpd.html>) authentication that wanted to write a log message to `auditd`. Until 3.15, that could only be done from the initial PID namespace, where processes could see the PID of `auditd`. Some distributions put vsftpd in its own PID namespace, however, which meant that vsftpd could not talk to auditd. By adding the `CAP_AUDIT_WRITE` capability to the program and adding some code in 3.15, though, that could be worked around. 

PID namespaces also present another problem for audit: the PIDs that get reported are not the "real" PIDs in the system. Processes within a PID namespace get their own PID range that is separate from the PIDs in the parent namespace (which might be the initial namespace where the real system PIDs are used). So audit needed to do a translation of the PID reported from non-initial PID namespaces. Someday, when `CAP_AUDIT_CONTROL` is allowed in PID namespaces (so that processes with that capability can configure the audit filters), there will need to be more cleanup done on the PID handling in the kernel, he said. 

Allowing multiple `auditd` processes in the system would be reasonable if they are tied to user namespaces. There was an idea "thrown around" about creating a new audit namespace, but it became clear that yet another namespace was not a particularly popular idea. Having one `auditd` per user namespace still requires some process having `CAP_AUDIT_CONTROL` within the namespace. He wondered if the process creating the user namespace also needed that capability. 

Beyond that, the configuration of audit running in the initial namespace cannot be changed from inside user namespaces even with the capability. In particular, only the initial namespace audit can panic the system; instead of that, the audit in a user namespace might instead kill off the user namespace and all its children if it cannot log (thus wants to panic). So each user namespace will get its own set of audit rules (a "rulespace") and its own event queue. Originally it was thought that the event queue might be shared by all of the `auditd` processes, but a single one that overflowed the queue would affect the rest of the system, which is unacceptable, Briggs said. 

There is interest in being able to track containers by some kind of ID. There was a proposal in 2013 to use the `/proc` inode number that uniquely identifies each namespace in the audit log messages. He felt that was harder to use, so he prototyped a simple incrementing serial number for each namespace. The checkpoint/restore in user space (CRIU) developers were not happy with that, since those numbers would not easily translate during a migration. 

Eventually, Briggs reworked the inode-number-based scheme to work with the namespace filesystem (nsfs). Each event then has a set of namespace IDs along with a device ID for the nsfs. That allows container orchestration systems to track the information, even across migrations, which allows them to aggregate logs from multiple hosts. 

An alternative would be to add a "container ID" that would be set by the orchestration system and tracked in the task structure. The container ID would be inherited by children and audit events would contain the ID. There is precedent for this kind of ID, he said; session IDs are not something the kernel itself knows anything about, but it helps user space manage those values. 

In conclusion, he said that namespace support for audit is largely working at this point, though changes for net and PID namespaces will be needed down the road. There is work to be done to allow multiple `auditd` processes anchored to the user namespace, as well. As far as IDs go, there is a decision to be made between the list of namespace IDs versus a single kernel-managed container ID. He favors the former, even though dealing with eight separate numbers is harder to use. Either solution will require higher-level tools to map, track, and aggregate information about the containers across multiple hosts. 

[I would like to thank the Linux Foundation for travel support to attend the Linux Security Summit in Toronto.]
