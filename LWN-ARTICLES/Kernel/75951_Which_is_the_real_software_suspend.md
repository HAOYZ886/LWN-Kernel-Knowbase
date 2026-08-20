---
title: "Which is the real software suspend?"
url: https://lwn.net/Articles/75951/
date: "March 16, 2004"
category: Software suspend; pmdisk
author: 
---

Laptop users may well have noticed that there are no less than three competing software suspend implementations for the 2.6 kernel. Two of them (pmdisk and swsusp) are in the kernel itself; the third ([swsusp2](<http://swsusp.sourceforge.net/>)) is not, but is also the implementation which has seen the most work over the last several months. Unfortunately, none of these implementations could be said to be production-level code. It _is_ possible to make a Linux system suspend to disk and resume into something that still runs, but making it work is not yet for the faint of heart. 

The software suspend discussion began anew when Pavel Machek, the maintainer of the in-kernel swsusp code, [asked](<https://lwn.net/Articles/75952/>) where things were going. Pavel's preference, not surprisingly, would be to remove the pmdisk code and stick with swsusp. Pavel is not alone in feeling this way. The pmdisk implementation is a fork of the swsusp code created by Patrick Mochel, who was not enjoying good relations with Pavel at the time. By some accounts, the pmdisk code is better, but it suffers from a major problem: Patrick has gotten a new job and has vanished from the kernel development world. As a result, pmdisk has seen no development work for several months, and it is a rare user who can make it work reliably. Unless Patrick surfaces and starts working on the code again, it is likely to go away fairly soon. 

The real question is what to do about swsusp2. This version of the suspend code has seen significant work by Nigel Cunningham and others. It has a number of features that others lack: the ability to abort a suspend operation, a "nice display," compression of the saved image (which can speed suspends and resumes on systems with slow disks), etc. The real difference, though, is that swsusp2 is, for many people, the only version that works at all reliably. So there is some real desire to see the swsusp2 work merged into 2.6, and further development efforts concentrated there. 

The hangup seems to be the fact that the swsusp2 patch is large, and it touches a great many core files. Many of those changes are aimed at making the "refrigerator" work better. Before a system can be suspended, all processes must be put into a quiet, known state. This works by setting a "freeze" flag and sending a signal to every process telling it to put itself into the refrigerator. Once all processes are nicely chilled, the system can save its state and suspend itself. 

Processes will not refrigerate themselves immediately; they must first get to a point where they hold no important resources. Sometimes, a process must get something from another process before it can be refrigerated; the example that is often raised is a process waiting for a response from an NFS server process. If the NFS server is refrigerated first, the other process will never get to where it can be frozen, and the suspend operation will fail. To avoid this sort of situation, the swsusp2 developers have gone to great lengths to identify places where a process should not, yet, be refrigerated. The result is a great many macros with names like `SWSUSP_ACTIVITY_STARTING` sprinkled widely though the code. If software suspend is not configured into the kernel, these macros simply vanish, so the actual changes to the core kernel are smaller than a look at a simple diffstat listing would indicate. Swsusp2 remains a large patch, however. 

Nigel has offered to provide a version of swsusp2 which lacks the intrusive refrigerator changes, though he warns that it will eventually become clear that those changes are needed. Andrew Morton has [indicated](<https://lwn.net/Articles/75971/>) that this would be a step in the right direction, but he is asking for more: 

Even happier would be a series of small, well explained patches which bring swsusp into a final shape upon which more than one developer actually agrees. 

These wholesale replacements and deletions are an indication that something has gone wrong with the development process here. 

What clearly needs to happen is that the swsusp2 work needs to be broken down into a long series of patches of the type that the kernel developers like to see: small and focused. That will be a significant effort, and the swsusp2 developers appear to lack the time to do that anytime soon. Now, perhaps, is the time for people who are concerned about a working software suspend solution (which Linux really does need) to get together to bring an end to the current, confused situation.
