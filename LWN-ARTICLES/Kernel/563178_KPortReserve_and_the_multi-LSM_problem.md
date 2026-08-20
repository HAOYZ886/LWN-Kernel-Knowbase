---
title: "KPortReserve and the multi-LSM problem"
url: https://lwn.net/Articles/563178/
date: "August 14, 2013"
category: "Security-Security modules"
author: "By Jake Edge August 14, 2013"
---

By **Jake Edge**  
August 14, 2013

Network port numbers are a finite resource, and each port number can only be used by one application at a time. Ensuring that the "right" application gets a particular port number is important because that number is required by remote programs trying to connect to the program. Various methods exist to reserve specific ports, but there are still ways for an application to lose "its" port. Enter [KPortReserve](<https://lwn.net/Articles/563286/>), a Linux Security Module (LSM) that allows an administrator to ensure that a program gets its reservation. 

One could argue that KPortReserve does not really make sense as an LSM—in fact, Tetsuo Handa asked just that question in his RFC post proposing it. So far, no one has argued that way, and Casey Schaufler [took the opposite view](<https://lwn.net/Articles/563297/>), but the RFC has only been posted to the LSM and kernel hardening mailing lists. The level of opposition might rise if and when the patch set heads toward the mainline. 

But KPortReserve does solve a real problem. Administrators can ensure that automatic port assignments (i.e. those chosen when the `bind()` port number is zero) adhere to specific ranges by setting a range or ranges of ports in the `/proc/sys/net/ipv4/ip_local_reserved_ports` file. But that solution only works for applications that do not choose a specific port number. Programs that _do_ choose a particular port will be allowed to grab it—possibly at the expense of the administrator's choice. Furthermore, if the port number is not in the privileged range (<= 1024), even unprivileged programs can allocate it. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

There is at least one [existing user-space solution](<http://cyberelk.net/tim/2012/02/15/portreserve-systemd-solution/>) using [portreserve](<http://cyberelk.net/tim/software/portreserve/>), but it still suffers from race conditions. Systemd has a race-free way to reserve ports, but it requires changes to programs that will listen on those ports and is not available everywhere, which is why Handa turned to a kernel-based solution. 

The solution itself is fairly straightforward. It provides a `socket_bind()` method in its `struct security_operations` to intercept `bind()` calls, which checks the reserved list. An administrator can write some values to a control file (where, exactly, that control file would live and the syntax it would use were being discussed in the thread) to determine which ports are reserved and what program should be allowed to allocate them. For example: 

```
echo '10000 /path/to/server' >/path/to/control/file
```

That would restrict port 10,000 to only being used by the server program indicated by the path. A special "`<kernel>`" string could be used to specify that the port is reserved for kernel threads. 

Vasily Kulikov [objected](<https://lwn.net/Articles/563316/>) to specifying that certain _programs_ could bind the port, rather a user ID or some LSM security context, but Schaufler [disagreed](<https://lwn.net/Articles/563317/>), calling it ""very 21st century thinking"". His argument is that using unrelated attributes to govern port reservation could interfere with the normal uses of those attributes: 

[...] Android used (co-opted, hijacked) the UID to accomplish this. Some (but not all) aspects of SELinux policy in Fedora identify the program and its standing within the system. Both of these systems abuse security attributes that are not intended to identify programs to do just that. This limits the legitimate use of those attributes for their original purpose. 

What Tetsuo is proposing is using the information he really cares about (the program) rather than an attribute (UID, SELinux context, Smack label) that can be associated with the program. Further, he is using it in a way that does not interfere with the intended use of UIDs, labels or any other existing security attribute. 

Beyond that, Handa [noted](<https://lwn.net/Articles/563318/>) that all of the programs he is interested in for this feature are likely running as root. While it would seem that root-controlled processes could be coordinated so that they didn't step on each other's ports, there are, evidently, situations where that is not so easy to arrange. 

In his initial RFC, Handa wondered if the KPortReserve functionality should simply be added to the Yama LSM. At the 2011 Linux Security Summit, Yama was [targeted](<https://lwn.net/Articles/458652/>) as an LSM to hold discretionary access control (DAC) enhancements, which port reservations might be shoehorned into—maybe. But, then and since, there has been a concern that Yama _not_ become a "dumping ground" for unrelated security patches. Thus, Schaufler argued, Yama is not the right place for KPortReserve. 

However, there is the [well-known problem](<https://lwn.net/Articles/518345/>) for smaller, targeted LSMs: there is currently no way to have more than one LSM active on any given boot of the system. Handa's interest in Yama may partly be because it has, over time, changed from a "normal" LSM to one that can be [unconditionally stacked](<http://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=c6993e4ac002c92bc75379212e9179c36d4bf7ee>), which means that it will be called regardless of which LSM is currently active. Obviously, if KPortReserve were added to Yama, it would likewise evade the single-LSM restriction. 

But, of course, Schaufler has been working on another way around that restriction for some time now. There have been attempts to stack (or chain or compose) LSMs for nearly as long as they have existed, but none has ever reached the mainline. The latest entrant, Schaufler's "[multiple concurrent LSMs](<https://lwn.net/Articles/560802/>)" patch set, is now up to version 14. Unlike some earlier versions, any of the existing LSMs (SELinux, AppArmor, TOMOYO, or Smack) can now be arbitrarily combined using the technique. One would guess it wouldn't be difficult to incorporate a single-hook LSM like KPortReserve into the mix. 

While there was some discussion of Schaufler's patches when they were posted at the end of July—and no objections to the idea—it still is unclear when (or if) we will see this capability in a mainline kernel. One senses that we are getting closer to that point, and new single-purpose LSM ideas crop up fairly regularly, but we aren't there yet. Schaufler will be [presenting](<http://kernsec.org/wiki/index.php/Linux_Security_Summit_2013/Abstracts/Schaufler>) his ideas at the [Linux Security Summit](<http://kernsec.org/wiki/index.php/Linux_Security_Summit_2013>) in September. Perhaps the discussion there will help clarify the future of this feature.
