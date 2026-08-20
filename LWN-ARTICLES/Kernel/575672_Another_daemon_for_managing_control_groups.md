---
title: Another daemon for managing control groups
url: https://lwn.net/Articles/575672/
date: "December 5, 2013"
category: Control groups
author: "By Jake Edge December 5, 2013"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
December 5, 2013

Control groups (cgroups) in the kernel are [changing](<https://lwn.net/Articles/574317/>), in both their implementation and their interface. One of those changes is that systems that use the new cgroup interface will require a single management process to coordinate access. For many, that management daemon may well be the systemd init replacement, but there are distributions (notably Ubuntu) and users who will want a different choice. To that end, Serge E. Hallyn is working on an alternative cgroup management daemon that he calls "cgmanager". 

Cgroups are a way to group processes and to apply various resource limits to the group. The current cgroup API in the kernel is a filesystem-based interface called cgroupfs. Groups are represented as directories and processes are placed into them by writing their process ID to a special file. Additional special files are used to set various limits and to control other aspects of the group, which depend on the specific controllers that are associated with the group (i.e. CPU, memory, block I/O, etc.). 

Depending on permissions, any process may be able use that interface to set up its own hierarchy of groups, which is one of the problems with the existing implementation, according to cgroup co-maintainer Tejun Heo and others. So, in the future, there will be a single process that is responsible for managing the single hierarchy of groups that will be allowed under the new cgroup interface. While systemd already has code to perform that job, systemd has never been a requirement to use cgroups—but if it isn't used, something has to take its place. 

So Hallyn [put out](<https://lwn.net/Articles/575683/>) a preliminary design for cgmanager on the lxc-devel mailing list at the end of November. It envisions a single daemon process with a D-Bus interface that will manage the hierarchy, which will mount cgroupfs inside a private namespace that is accessible only by cgmanager. Processes will make requests of the daemon to create cgroups, configure them, and to move processes into them. Obviously, some of those operations are privileged, and Hallyn has worked out a set of privileges required for each. Users own the cgroups they create and can place their own processes into those cgroups. More complicated arrangements are possible, of course, but generally either require real root privileges or at least root inside a user namespace owned by the user. 

There is also the concept of handing the ownership of a cgroup (or hierarchy) to another user. Doing that using the standard filesystem permissions on the cgroupfs, as is done today, is a big security hole, which can lead to various kinds of denial of service. Mediating that access through cgmanager should alleviate that problem, however. 

So far, Hallyn's design has been well-received; there are questions and suggestions, of course, but overall it would seem that Hallyn has taken several constituencies into account. 

The first of those is the [LXC containers project](<http://linuxcontainers.org/>), but there are several other interested parties as well. Upstart-using distributions (mostly limited to Ubuntu and derivatives—though possibly Debian too, [depending](<https://lwn.net/Articles/572805/>)) will also need some kind of cgroup manager if they intend to use cgroups. Hallyn has clearly been talking to the Upstart folks, so it would seem that cgmanager should eventually fit the bill there. 

Another constituent is Google, which uses cgroups extensively in its fairly sizable datacenter. Tim Hockin from the search giant had been rather critical of the single hierarchy cgroup plan. He has since largely made his peace with the plan, but an [article from July](<https://lwn.net/Articles/557082/>) outlines Google's use of cgroups as well as some of the problems with the "new cgroups" that Hockin had foreseen. Clearly Hallyn was also paying attention to Google's needs as Hockin seems relatively pleased with the direction of cgmanager's design. In particular, Google uses the "change ownership" feature today, but it trusts all of the users on its cluster. For others, who may not be able to have that trust level, Hallyn has provided a means to get that functionality, while reducing its security impact. 

Where does systemd fit into this picture? The answer to that seems to essentially be: not at all. Systemd has its own [control group interface](<http://www.freedesktop.org/wiki/Software/systemd/ControlGroupInterface/>) that it intends to proceed with. Two messages from June make it pretty clear that Lennart Poettering, at least, is not particularly interested in having systemd [work with another cgroup manager](<https://lwn.net/Articles/557111/>) or to provide [cgroup management in a library](<https://lwn.net/Articles/557140/>). Poettering's contention that init should not be dependent on some other daemon seems sensible, but forcing anyone who wants to use cgroups to use systemd clearly isn't. 

It would be nice if some kind of common API could be worked out so that applications and users don't have to support two different ways of accessing cgroups. It would seem that Hockin [approached](<https://lwn.net/Articles/575705/>) the systemd folks about working together but was evidently rebuffed. That may make sense for the systemd project, but will unfortunately leave applications and users who need cgroup functionality in something of a bind. Supporting both systemd and cgmanager will likely be required. 

While it will undoubtedly be annoying to support two (or more if another competitor shows up) interfaces to cgroups, having two separate implementations may actually be a good thing—at least in the short term. That wouldn't preclude a common interface, of course, but that doesn't seem to be in the cards. Cgroups have been a problematic feature in the kernel since they were merged for 2.6.24 back in 2007. One hopes we are on our way toward a better implementation and user-space interface for the feature—so the more code that exercises all of that, the better.
