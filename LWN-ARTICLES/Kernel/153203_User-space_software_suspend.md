---
title: "User-space software suspend"
url: https://lwn.net/Articles/153203/
date: "September 26, 2005"
category: Software suspend; devkmem
author: 
---

Suspend-to-disk is a feature desired by many Linux users; both laptop and desktop users can benefit from being able to save the state of the system to a local drive and, after a reboot, find everything as they left it. The current in-kernel suspend mechanism works for many, but not everybody is comfortable with the large amount of invasive code required. The out-of-tree [suspend2](<http://www.suspend2.net/>) implementation adds quite a few worthwhile features, but at the cost of expanding the software suspend implementation still further. Concern over putting some of the suspend2 features into the kernel has been one of the factors preventing its merging so far. 

Pavel Machek, the maintainer of the in-kernel suspend implementation, has now complicated the pictured with [the swsusp3 patch](<http://lwn.net/Articles/152278/>), which moves some of the work of suspending the system into user space. This code is said to work; if this approach continues to show promise, it could point the way toward adding suspend2's features without growing the kernel. 

The software suspend process, in _very_ rough terms, works like this: 

  1. All processes on the system (with a few exceptions) are put into a special "frozen" state. 

  2. Any memory which has on-disk backing store is forced out to disk; this step essentially clears the system of all user-space pages. Any kernel memory which can be done without - caches and such - is also dropped. 

  3. Any remaining memory which is not in reserved space (not part of the kernel text, for all practical purposes) is written to a suspend image on the disk. Also written is a map saying where the pages came from in the first place. 

  4. The system is shut down. 

When the system is resumed, these steps are reversed in the opposite order \- except that user-space memory remains on disk until faulted in by the newly-restarted system. 

The swsusp3 patch does not move all of the above work to user space - much of it must be done in the kernel. What does move is step 3 - the writing of kernel memory - to disk. This operation is handled by way of `/dev/kmem`. To that end, the swsusp3 patch adds a set of scary `ioctl()` calls to the `/dev/kmem` driver. 

The new user-space suspend program begins by locking itself into memory. This step is required - it would not do for it to change the memory state in the middle of the process via page faults. A call to the new `IOCTL_FREEZE` operation on `/dev/kmem` performs the first two steps listed above: freezing processes and clearing memory. The `IOCTL_ATOMIC_SNAPSHOT` call then puts devices on hold and creates an in-kernel list of pages which must be saved. 

The `ioctl(/dev/kmem, IOCTL_ATOMIC_SNAPSHOT)` call returns a pointer to that list of pages. The user-space program can then obtain the list (by reading it from `/dev/kmem`) and pass through it. Each page on the list is read from kernel memory and written to the suspend image file. Finally, the list itself is written to the suspend image. Once that is done, the system can be powered down. 

The resume process writes the saved image back into kernel memory. It has the additional problem, however, of having to deal with two kernels at once. This process will be running under a freshly-booted kernel (the "resume kernel") with its own idea of the state of the world; that state will eventually be overwritten by the state from the suspended kernel, but that step must be handled carefully. The resume process cannot simply overwrite arbitrary kernel memory, since it is counting on the resume kernel to continue to function until all of the suspended kernel's memory has been read in. So the user-space resume process must be able to allocate pages in kernel space. 

The answer is, of course, another `ioctl()` command, IOCTL_KMALLOC, which executes a `get_zeroed_page()` call and returns the address of the resulting page to user space. Once a full set of pages has been loaded with the suspended kernel's memory, an updated page map can be stored in the kernel, and an `IOCTL_ATOMIC_RESTORE` operation tells the resume kernel to finish the process. 

This code is very much in an early stage; even people who do not hesitate to use software suspend may want to be careful with swsusp3 on systems they actually care about resuming. Once things settle down, however, swsusp3 could open the door to a number of features, including graphical progress displays and the ability to interrupt the suspend process, which users have been asking for.
