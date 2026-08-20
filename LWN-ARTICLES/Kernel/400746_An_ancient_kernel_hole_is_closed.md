---
title: An ancient kernel hole is closed
url: https://lwn.net/Articles/400746/
date: "August 18, 2010"
category: Security
author: "By Jake Edge August 18, 2010"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jake Edge**  
August 18, 2010

A longstanding bug in the Linux kernel--quite possibly since the first 2.6 release in 2003--has been fixed by a recent [patch](<http://git.kernel.org/?p=linux/kernel/git/torvalds/linux-2.6.git;a=commitdiff;h=320b2b8de12698082609ebbc1a17165727f4c893>), but the nearly two-month delay between the report and the fix is raising some eyebrows. It is a local privilege escalation flaw that can be triggered by malicious X clients forcing the server to overrun its stack. 

The problem was discovered by Rafal Wojtczuk of [Invisible Things Lab](<http://www.invisiblethingslab.com/itl/Welcome.html>) (ITL) while working on [Qubes OS](<http://lwn.net/Articles/386065/>), ITL's virtualization-based, security-focused operating system. ITL's CEO Joanna Rutkowska [describes the flaw](<http://theinvisiblethings.blogspot.com/2010/08/skeletons-hidden-in-linux-closet.html>) on the company's blog and Wojtczuk released a [paper [PDF]](<http://www.invisiblethingslab.com/resources/misc-2010/xorg-large-memory-attacks.pdf>) on August 17 with lots more details. In that paper, he notes that he reported the problem to the X.org security team on June 17, and by June 20 the team had determined that it should be fixed in the kernel. But it took until August 13 before that actually happened. 

In addition, the description in the patch isn't terribly forthcoming about the security implications of the bug. That is in keeping with Linus Torvalds's policy of disclosing security bugs via code, but not in the commit message, because he feels that may help "script kiddies" easily exploit the flaw. There have been endless arguments about that policy on linux-kernel, here at LWN, and elsewhere, but Torvalds is quite adamant about his stance. While [some](<http://news.softpedia.com/news/Critical-Vulnerability-Silently-Patched-in-Linux-Kernel-152678.shtml>) are calling it a "silent" security fix--and to some extent it is--it really should not come as much of a surprise. 

The bug is not in the X server, though the fact that it runs as root on most distributions makes the privilege escalation possible. Because Linux does not separate process stack and heap pages, overrunning a stack page into an adjacent heap page is possible. That means that a sufficiently deep stack (from a recursive call for example) could end up using memory in the heap. A program that can write to that heap page (e.g. an X client) could then manipulate the return address of one of the calls to jump to a place of its choosing. That means that the client can cause the server to run code of its choosing--arbitrary code execution--which can be leveraged to gain root privileges. 

Evidently, this kind of exploit has been known for five years or more as Wojtczuk's paper points to a [presentation [PDF]](<http://cansecwest.com/core05/memory_vulns_delalleau.pdf>) by Gael Delalleau at CanSecWest in 2005 describing the problem, and pointing out that Linux was vulnerable to it. Unfortunately it would seem that the information didn't reach the kernel security team until it was rediscovered recently. 

The X server has some other attributes that make it an ideal candidate to exploit the kernel vulnerability. Most servers run with the MIT shared memory extension (MIT-SHM) which allows clients to share memory with the server to exchange image data. An attacker can cause the X server to almost completely exhaust its address space by creating many shared memory segments to share with the server. 64-bit systems must allocate roughly 36,000 32Kx32K pixmaps in the server before creating the shared memory to further reduce the address space. One of the shared memory segments will get attached by the server in the "proper" position with respect to the server's stack. 

Once that is done, the client then causes the X server to make a recursive function call. By looking through the shared memory segments for non-zero data, the client can figure out which of the segments is located adjacent to the stack. At that point, it spawns another process that continuously overwrites that segment with the attack payload and triggers the recursion again. When the recursion unwinds, it will hit the exploit code and jump off to do the attacker's bidding--as _root_. 

It is possible that other root processes or setuid programs are vulnerable to the kernel flaw, and X servers with MIT-SHM disabled may be as well. All of those cases are, as yet, hypothetical, and are likely to be much harder to exploit. 

X.org hacker Keith Packard described how the fix progressed within the X team. He said that they tried several fixes in the X server, including using resource limits to reduce the address space allowed to the server and limiting recursion depth while ensuring adequate stack depth. None of those were deemed complete fixes for the problem, though. 

Andrea Arcangeli and Nick Piggin worked on a fix on the kernel side, but it was not accepted by Torvalds because it ""violated some internal VM rules"", Packard said. As the deadline for disclosure neared--after being extended from its original August 1 date--Torvalds implemented his own solution which fixed the problem. Overall, Packard was pleased with the response: 

The various security teams worked well together in coming up with proposed solutions, although the process was a bit slower than I would have liked. The kernel patch proposed by Linus was tested by Peter Hutterer within a few hours to verify that it prevented the specific attack written by Rafal. 

It should also be noted that Torvalds's original fix had a bug, which he has since fixed. The new [patch](<http://git.kernel.org/?p=linux/kernel/git/torvalds/linux-2.6.git;a=commitdiff;h=11ac552477e32835cb6970bf0a70c210807f5673>), along with a [fix](<http://git.kernel.org/?p=linux/kernel/git/torvalds/linux-2.6.git;a=commitdiff;h=d7824370e26325c881b665350ce64fb0a4fde24a>) for a user-space-visible change to the `/proc/<pid>/maps` file are out for stable kernel review at the time of this writing. So, a full correct fix for the problem is not yet available except for those running development kernels or patching the fix in on their own. 

All of the ""fancy security mechanisms"" in Linux were not able to stop this particular exploit, Rutkowska said. She also pointed out that the "[sandbox -X](<http://danwalsh.livejournal.com/31146.html>)" SELinux compartmentalization would not stop this exploit. While it isn't a direct remote exploit, it only takes one vulnerable X client (web browser, PDF viewer, etc.) to turn it into something that is remotely exploitable. Given the number of vulnerable kernels out there, it could certainly be a bigger problem in the future. 

The most unfortunate aspect of the bug is the length of time it took to fix. Not just the two months between its discovery and fix, but also the five years since Delalleau's presentation. We need to get better at paying attention to publicly accessible security reports and fixing the problems they describe. One has to wonder how many attackers took note of the CanSecWest presentation and have been using that knowledge for ill. There have been no reports of widespread exploitation--that would likely have been noticed--but smaller, targeted attacks may well have taken advantage of the flaw.
