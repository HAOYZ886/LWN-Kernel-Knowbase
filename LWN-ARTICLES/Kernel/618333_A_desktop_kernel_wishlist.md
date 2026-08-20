---
title: A desktop kernel wishlist
url: https://lwn.net/Articles/618333/
date: "October 29, 2014"
category: Desktop support
author: "By Nathan Willis October 29, 2014"
---

By **Nathan Willis**  
October 29, 2014

GNOME developer Bastien Nocera recently [shared](<https://lwn.net/Articles/618360/>) a "[wishlist](<https://wiki.gnome.org/BastienNocera/KernelWishlist>)" with the kernel mailing list, outlining a number of features that GNOME and other desktop-environment projects would like to see added to or enhanced in the kernel. In the resulting discussion, some of the wishlist items were subsequently crossed off, but many of the others sparked real discussion that could, in time, develop into mainline kernel features. 

Nocera prefaced his list by saying that GNOME has had productive discussions with kernel developers in the past, and that the current list consisted of ""items that kernel developers might not realise we'd like to rely on, or don't know that we'd make use of if merged."" Most of these items fall into either one of two categories: power management or filesystem features, although there are a handful of others in the mix as well. 

#### Less power

Under power management, Nocera listed native support for hybrid suspend (known as Intel Rapid Start on certain hardware), connected standby support, ""a hibernation implementation that doesn't use the swap space"", and several smaller items (such as establishing uniform semantics for screen-backlight settings and better documentation for managing USB power). 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Hybrid suspend is a firmware-level feature intended to minimize the time require to restore a system from the suspended state. It works by initially suspending the system to RAM, but then writing the memory contents to disk (directly from the firmware) after a given time period and hibernating. Resume will be fast before the timeout expires, but the system will preserve its state regardless of how long it is suspended. Matthew Garrett [wrote a patch](<http://mjg59.dreamwidth.org/26022.html>) implementing support for the feature in mid-2013, which Nocera subsequently [made a push](<http://thread.gmane.org/gmane.comp.sysutils.systemd.devel/13511/>) to merge upstream--a push that ultimately stalled. One of the critiques was that Intel's hybrid-suspend feature would eventually be made obsolete by connected standby, a newer idle-state power mode that allows certain background processes to continue. Garrett has also [worked on](<https://lwn.net/Articles/580451/>) an approach to connected standby. 

John Stultz [asked](<https://lwn.net/Articles/618362/>) for clarification about one other power-management request: exporting the cause of a wake event. Nocera [elaborated](<https://lwn.net/Articles/618363/>), saying that the goal was to be able to determine whether the machine was awakened by a user event or by a hardware event (such as the realtime clock alarm), in order to respond accordingly to different scenarios. The use case he cited was for user-space code to try and determine if it was a good time to run a previously scheduled backup: if the user woke the machine by opening the laptop lid, presumably it would be a bad time to start a lengthy backup process. If the wake was automatic, the backup should proceed. 

Stultz, however, argued that reporting the cause of the wakeup would not truly satisfy the use case--in part because any number of wake events could take place (and, being asynchronous, could arrive in an unhelpful order) in between the kernel waking up and user-space code being able to run. But more importantly, as Zygo Blaxnell [put it](<https://lwn.net/Articles/618367/>), which event was most recent is far less important than (for example) whether or not the user is actually using the machine--a fact that could be determined through other means, such as keyboard activity. Alan Cox, on the other hand, [commented](<https://lwn.net/Articles/618502/>) that, in the long term, most of the assumptions that go with current thinking about suspend, resume, and hibernation states may go away anyhow: 

\- There may be no such thing as suspend or resume, just make your code very well behaved on wakeup events, and closing unneeded devices/resources whenever it can. 

\- On/off is an extreme action rarely taken (feature parity with 1970s VAXen ;-) ) 

\- The "blob with a lid" model of construction is no longer useful. Even a keyboarded device is quite likely have a removable keyboard. 

#### Filesystem issues

The second large category of feature requests concerned filesystem features and the VFS layer. First, Nocera reported that inotify is not meeting the needs of desktop utilities in a [number of ways](<https://lwn.net/Articles/605128/>). Performance on large directory structures consumes too many resources, file-creation notification requires watching an entire directory, and monitoring directories' file-renaming and removal events is expensive on large directories. These limitations impact the performance of filesystem indexers, backup tools, and programs that manage file "libraries" (e.g., music and video managers). 

Sergey Davidoff of elementary OS [elaborated](<https://lwn.net/Articles/618463/>) on the subject, saying that desktop application developers are keen to move to the file-library concept (as used in music and video managers) in other application types as well. Presenting the user with the filesystem hierarchy, he said, is far less useful than intelligently tracking the relevant files and allowing the user to search and interact with them based on their metadata. fanotify, as both noted, lacks the proper level of detail, such as reporting rename and move events. 

Nocera also asked for a way to propagate timestamp changes up a directory chain. That is, if a file located in `/foo/bar/` changes, there would be a way to detect the change not only on the file itself, but also on `/foo/bar` and on `/foo` itself. He added, though, that simply updating the change time on the containing directories would clearly be the wrong solution, since it would break many programs. 

In short, he said, user-space programs would benefit from a better file-change notification system--ideally one that would consolidate events and monitor a directory structure without re-crawling it periodically. The combination of improving fanotify and adding user-space glue might work, he said, as would adding the changelog features currently available in Btrfs and XFS to other filesystems. 

Pavel Machek [asked](<https://lwn.net/Articles/618472/>) whether or not a adding a (hypothetical) recursive version of the mtime timestamp would be a possible solution. Davidoff [replied](<https://lwn.net/Articles/618473/>) with skepticism that it would work for monitoring online changes, but that monitoring Btrfs's changelog ""more or less as it happens"" would probably suffice. In particular, monitoring Btrfs changelogs on the fly could at least ensure that a fixed-size buffer would get the job done, as opposed to the unbounded memory that fanotify would require for the same task. 

#### Miscellany

Nocera's final cluster of wishlist items is a bit of a grab bag. It includes a better user-space API for the industrial input/output (IIO) subsystem used for various sensors, a user-space [helper](<https://lwn.net/Articles/590960/>) for the out-of-memory (OOM) killer, a system call to poll whether any processes out of a set of processes have exited, and a variant of `epoll_wait()` that accepts an absolute time rather than a timeout. 

As was the case with the other categories, some of these items sparked quick responses. Patrik Lundquist [suggested](<https://lwn.net/Articles/618523/>) that the desired `epoll_wait()` functionality could be achieved with `timerfd()`. To that, Nocera [quoted](<https://lwn.net/Articles/618508/>) the original source of the request, Ryan Lortie, who said that making a separate call to set up `timerfd()` on every instance of entering the kernel to sleep is cumbersome, and that ""epoll in general suffers from being _way_ too chatty about the syscalls that you have to do."" Andy Lutomirski [added](<https://lwn.net/Articles/618483/>) that he had implemented [procfs polling](<http://lkml.kernel.org/g/1840e47fc4113af16989a4250d98bed62a9bce53.1354559528.git.luto@amacapital.net>) several years ago, and would be willing to resume work on it if it were useful. procfs polling would allow a user to open a set of `/proc` directories corresponding to a chosen set of processes, then poll the directories to see if any exit. 

As to the other suggestions, there has thus far been little reaction either way. Certainly a number of the wishlist items boil down to implementing friendlier user-space APIs. Nocera commented on both the IIO and wake-event reporting issues that the present-day interface of examining raw sysfs files is far from sufficient. 

On the whole, though, the kernel community has clearly been receptive to these needs of desktop-environment projects. On the wishlist wiki page, Nocera likened the exercise to the "plumbers' wishlists" [submitted](<https://lwn.net/Articles/462076/>) to kernel developers by Kay Sievers, Lennart Poettering, and Harald Hoyer. The plumbers' wishlist approach, of course, was successful enough that the plumbers in question have since repeated it. That bodes well for Nocera's desktop wishlist. 

It is fairly common to hear about new kernel work that is driven by the needs of either high-end data center users or embedded-system builders--perhaps simply because companies in those lines of business tend to hire kernel developers. Thus, is it always good to see that the kernel community is equally responsive to the needs of user-space developers working in other areas, when those developers take time to reach out with their concerns.
