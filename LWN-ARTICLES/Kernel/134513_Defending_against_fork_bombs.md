---
title: Defending against fork bombs
url: https://lwn.net/Articles/134513/
date: "May 3, 2005"
category: "Fork bombs; Memory management-Out-of-memory handling; OOM killer; Security-Security technologies"
author: 
---

Standard wisdom says that the proper defense against fork bomb attacks (where a simple script forks children until the system chokes under the load) is to use resource limits. Put a cap on the number of processes which can be created, and the problem goes away. In reality it's not quite so simple; the limit can be softened by logging in multiple times. And, in any case, some people feel that the system should not collapse when faced with such an attack. A Linux system, it is said, should not be so easy to bring down in its default configuration. 

The last defense against fork bombs is typically the out-of-memory (OOM) killer. As the system fills up with processes, it will eventually run out of memory and, in its desperation, start looking for processes to kill. The OOM killer has a set of heuristics which attempt to choose the "best" process to kill. These rules help the system to avoid (sometimes) killing processes which are vital to the continued operation of the system. They are not particularly helpful in dealing with fork bombs, however. 

Coywolf Qi Hunt has posted [a patch](<https://lwn.net/Articles/134388/>) which tries to do a better job of defending against fork bombs in the OOM killer. It works by extending the task structure to keep better track of a process's "biological" parent and children. These lists are maintained separately from the regular process hierarchy pointers, and are not actually used during normal system operation. They are, in other words, pure overhead most of the time. 

Things change, however, when an out-of-memory situation hits. When the OOM killer starts up, it will select its first victim in the usual way. When a second process is chosen for an untimely death, however, the new lists come into play. For both the current and previous victim, the OOM killer will traverse the "biological parent" pointers to create a path through the process hierarchy. Using those paths, the code can select the "least common ancestor," the lowest process which is an ancestor to both victims. Then, rather than killing the second chosen victim directly, the OOM killer goes after the ancestor - and all of its children. If the OOM situation persists, the killer should be able to quickly work its way up the process hierarchy until it finds (and eliminates) the process responsible for the whole mess. 

Coywolf has a set of test cases and a system he is willing to run them on; for all but the nastiest of the three, the patched system was able to put an end to the fork bomb attack without any ill effects beyond a temporary slowdown. In the worst case, the system still recovered, but with some collateral damage. The patch adds some significant overhead (one pointer and two `list_head` structures) to each process in the system, so it may encounter some resistance - most systems will pay that overhead, but never actually need to run the OOM killer. But, for systems which are exposed to that sort of attack, this patch could be a useful last line of defense.
