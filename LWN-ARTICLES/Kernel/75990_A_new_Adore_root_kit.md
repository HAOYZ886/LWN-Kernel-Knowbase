---
title: A new Adore root kit
url: https://lwn.net/Articles/75990/
date: "March 17, 2004"
category: Adore root kit; Security
author: 
---

For your cracking pleasure: a new version of the Adore root kit has been [announced](<https://lwn.net/Articles/75991/>). This code is, of course, "for educational purposes only." On the notion that it's best to look at code like this when one has downloaded it explicitly, rather than when one has found it on one's system, we grabbed a copy. 

Adore is a kernel module which is intended to give a cracker the full run of a compromised system without detection. To that end, it installs itself into several key parts of the kernel and lurks until somebody comes along who knows the right "key," where a key is a special process ID. If you do not know this key, finding signs of an Adore installation will be difficult, to say the least. 

The module starts by hooking itself into various filesystems. It digs up the inode for the root filesystem, and replaces that inode's `readdir()` function pointer with one of its own. The Adore version performs like the one it replaces, except that it hides any files owned by a specific user and group ID. If you are a Black Hat trying to keep installed files out of the eye of the system administrator, this is the way to do it. 

Similarly, Adore hooks itself into the lookup function for `/proc`. An attempt to read `/proc/KEY`, where `KEY` is a predefined key value, will give the current process the ability to access other Adore functions. A process which has been "authenticated" in this way can then, by accessing other special `/proc` filenames, give itself full root privileges or tell Adore to hide other processes from view. The module keeps a list of such processes; once a process appears in that list, it will never appear in `/proc`, and thus it will not by displayed by utilities like `ps` or `top`. The only way to find such processes, it would seem, would be to dig through the entire kernel task list and check to see if any of them are not represented in `/proc`. 

People who crack into systems may well want to run network services on those systems. To cater to their needs, Adore replaces the `show()` function for `/proc/net/tcp`; the new version edits out any connections involving ports that the person installing Adore would rather others didn't know about. A hidden server process, running from a hidden executable, and sitting behind a hidden port could be very hard for a system administrator to find. 

For good measure, Adore will also filter out entries made into files like `/var/log/utmp` or syslog on behalf of hidden processes. 

The one thing Adore does not do is hide itself; it will show up in the list of loaded kernel modules. To address that, a separate module called "cleanup" is provided. If cleanup is loaded immediately after Adore, it will patch Adore out of the list of loaded modules, thus hiding it altogether. 

All of this functionality has been implemented in a kernel module which is a mere 600 lines long. This module is scary; it is a living demonstration of what an attacker can do once he gets root access on a system. A careful attacker could, using this module, maintain undetected control of a compromised system indefinitely.
