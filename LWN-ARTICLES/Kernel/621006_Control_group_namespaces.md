---
title: Control group namespaces
url: https://lwn.net/Articles/621006/
date: "November 19, 2014"
category: Control groups; Namespaces
author: "By Jake Edge November 19, 2014"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jake Edge**  
November 19, 2014

Containers in Linux use both [control groups](<https://lwn.net/Articles/604609/>) (cgroups) and [namespaces](<https://lwn.net/Articles/531114/>) to isolate a set of processes into a virtual system at the operating system level (as opposed to at the hardware level as with KVM). But, currently, cgroups themselves are not virtualized. That leads to a number of problems for container managers (e.g. LXC, Docker), since processes inside the containers can see the global cgroup landscape. A recent patch set seeks to fix those problems by creating a new namespace for cgroups. 

Aditya Kali [posted](<https://lwn.net/Articles/618873/>) v2 of the cgroup namespace patch set at the end of October. It is based on Tejun Heo's [unified cgroup hierarchy](<https://lwn.net/Articles/601840/>) work and is meant to solve several problems for containers. For example, when a task consults the `/proc/self/cgroup` file, it currently sees the full cgroup path from the global cgroup hierarchy, which leaks information about the host system. That information makes it difficult to do container migration across systems (using checkpoint/restore in user space, aka CRIU) since all of the names would need to be unique across all systems so that there were no collisions with names on the new system. In addition, running container-management tools inside of containers (to nest them) is difficult because the information available is not relative to the existing container. 

The basic idea in the patch set is that a process can call `unshare()` using the `CLONE_NEWCGROUP` flag to enter a new cgroup namespace. Once it does that, it will no longer see the global cgroup hierarchy, but will instead see itself in the root cgroup. In the first patch, Kali described how that would look: 

```
$ cat /proc/self/cgroup
        0:cpuset,cpu,cpuacct,memory,devices,freezer,hugetlb:/batchjobs/c_job_id1
    
        $ ~/unshare -c  # calls unshare(CLONE_NEWCGROUP) and exec’s /bin/bash
    
        # From within new cgroupns, process sees that its in the root cgroup
        [ns]$ cat /proc/self/cgroup
        0:cpuset,cpu,cpuacct,memory,devices,freezer,hugetlb:/
```

Similarly, `/proc/<PID>/cgroup` will return a path that is relative to the cgroup namespace root (known as cgroupns-root). In addition, mounting control group filesystem (cgroupfs) within the namespace would make cgroupns-root be the root of the mounted cgroupfs. In effect, it would be like bind-mounting the cgroup namespace's subtree in cgroupfs (i.e. starting at cgroupns-root) at the mount point. Currently, mounting cgroupfs exposes the full hierarchy of existing cgroups, which leaks unnecessary (and confusing) information. 

The main area of discussion on the patch set (and its [v1 predecessor](<https://lwn.net/Articles/616099/>)) has been about which processes can be moved into cgroup namespaces at various levels in the hierarchy (e.g. below, above, or into sibling hierarchies). The original patches only allowed processes to be moved into namespaces below the root of the cgroupns they are in, but that was deemed too restrictive (it could lead to a situation where the root user could not move a process to a particular namespace, for example). The current patches allow suitably privileged processes to move processes to any cgroup namespace in the hierarchy, though it does not do any implicit movement of the process into a different cgroup—that must be handled by the process doing the moving. That can lead to relative paths in `/proc/<PID>/cgroup` depending on the namespace of the process looking and that of the PID in question: 

```
# ns is at '/groups/a', PID 4567 is in a cgroupns at '/groups/b'
        [ns]$ cat /proc/4567/cgroup
        0:cpuset,cpu,cpuacct,memory,devices,freezer,hugetlb:/../b
```

With those changes in place, container managers can treat nested containers the same as they do the top level. Tools for container migration can also do their job without having to be concerned about name collisions on the new system. 

So far, the reception has been fairly positive. There has been discussion about various aspects of the patch set, but no one seems to be putting the brakes on the idea. In fact, namespaces developer Eric W. Biederman [noted](<https://lwn.net/Articles/621622/>) that the patch set ""definitely looks like the right direction to go, and something that in some form or another I had been asking for since cgroups were merged"". There is certainly more work to do, but it would seem likely that a new namespace for cgroups is in the kernel's future.
