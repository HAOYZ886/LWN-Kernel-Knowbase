---
title: Btrfs at Facebook
url: https://lwn.net/Articles/824855/
date: "July 2, 2020"
category: "Btrfs; Filesystems-Btrfs"
author: "By Jonathan Corbet July 2, 2020 OSSNA"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
July 2, 2020

* * *

[OSSNA](<https://lwn.net/Archives/ConferenceByYear/#2020-Open_Source_Summit_North_America>)

The Btrfs filesystem has had a long and sometimes turbulent history; LWN first [wrote about it](<https://lwn.net/Articles/238923/>) in 2007. It offers features not found in any other mainline Linux filesystem, but reliability and performance problems have prevented its widespread adoption. There is at least one company that is using Btrfs on a massive scale, though: Facebook. At the [2020 Open Source Summit North America](<https://events.linuxfoundation.org/open-source-summit-north-america/>) virtual event, Btrfs developer Josef Bacik described why and how Facebook has invested deeply in Btrfs and where the remaining challenges are. 

Every Facebook service, Bacik began, runs within a container; among other things, that makes it easy to migrate services between machines (or even between data centers). Facebook has a huge number of machines, so it is impossible to manage them in any sort of unique way; the company wants all of these machines to be as consistent as possible. It should be possible to move any service to any machine at any time. The company will, on occasion, bring down entire data centers to test how well its disaster-recovery mechanisms work. 

#### Faster testing and more

All of these containerized services are using Btrfs for their root filesystem. The initial use case within Facebook, though, was for the build servers. The company has a lot of code, implementing the web pages, mobile apps, test suites, and the infrastructure to support all of that. 

The Facebook workflow dictates that nobody commits code directly to a repository. Instead, there is a whole testing routine that is run on each change first. The build system will clone the company repository, apply the patch, build the system, and run the tests; once that is done, the whole thing is cleaned up in preparation for the next patch to test. That cleanup phase, as it turns out, is relatively slow, averaging two or three minutes to delete large directory trees full of code. Some tests can take ten minutes to clean up, during which that machine is unavailable to run the next test. 

The end result was that developers were finding that it would take hours to get individual changes through the process. So the infrastructure team decided to try using Btrfs. Rather than creating a clone of the repository, the test system just makes a snapshot, which is a nearly instantaneous operation. After the tests are run, the snapshot is deleted, which also appears to be instantaneous from a user-space point of view. There is, of course, a worker thread actually cleaning up the snapshot in the background, but cleaning up a snapshot is a lot faster than removing directories from an ordinary filesystem. This change saves a lot of time in the build system and reduced the capacity requirement — the number of machines needed to do builds and testing — by one-third. 

After that experiment worked out so well, the infrastructure team switched fully to Btrfs; the entire build system now uses it. It turns out that there was another strong reason to switch to Btrfs: its support for on-disk compression. The point here is not just saving storage space, but also extending its lifetime. Facebook spends a lot of money on flash storage — evidently inexpensive, low-quality flash storage at that. The company [![\[Josef Bacik\]](https://static.lwn.net/images/conf/2020/ossna/JosefBacik-sm.png)](<https://lwn.net/Articles/824864/>) would like this storage to last as long as possible, which implies minimizing the number of write cycles performed. Source code tends to compress well, so compression reduces the number of blocks written considerably, slowing the process of wearing out the storage devices. 

This work, Bacik said, was done by the infrastructure team without any sort of encouragement by Facebook's Btrfs developers; indeed, they didn't even know that it was happening. He was surprised by how well it worked in the end. 

Then, there is the case of virtual machines for developers. Facebook has an "unreasonable amount" of engineering staff, Bacik said; each developer has a virtual machine for their work. These machines contain the entire source code for the web site; it is a struggle to get the whole thing to fit into the 800GB allotted for each and still leave room for some actual work to be done. Once again, compression helps to save the day, and works well, though he did admit that there have been some "`ENOSPC` issues" (problems that result when the filesystem runs out of available space). 

Another big part of the Facebook code base is the container system itself, known internally as "tupperware". Containers in this system use Btrfs for the root filesystem, a choice that enables a number of interesting things. The [send and receive mechanism](<https://lwn.net/Articles/581558/>) can be used both to build the base image and to enable fast, reliable upgrades. When a task is deployed to a container, a snapshot is made of the base image (running a version of CentOS) and the specific container is loaded on top. When that service is done, cleanup is just a matter of deleting the working subvolume and returning to the base snapshot. 

Additionally, Btrfs compression once again reduces write I/O, helping Facebook to make the best use of cheap flash drives. Btrfs is also the only Linux filesystem that works with the [io.latency](<https://lwn.net/Articles/782876/>) and [io.cost](<https://lwn.net/Articles/792256/>) (formerly io.weight) block I/O controllers. These controllers don't work with ext4 at all, he said, and there are some problems still with XFS; he has not been able to invest the effort to make things work better on those filesystems. 

An in-progress project concerns the WhatsApp service. WhatsApp messages are normally stored on the users' devices, but they must be kept centrally when the user is offline. Given the number of users, that's a fair amount of storage. Facebook is using XFS for this task, but has run into unspecified "weird scalability issues". Btrfs compression can help here as well, and snapshots will be useful for cleaning things up. 

But Btrfs, too, has run into scalability problems with this workload. Messages are tiny, compressed files; they are small enough that the message text is usually stored with the file's metadata rather than in a separate data extent. That leads to filesystems with hundreds of gigabytes of metadata and high levels of fragmentation. These problems have been addressed, Bacik said, and it's "relatively smooth sailing" now. That said, there are still some issues left to be dealt with, and WhatsApp may not make the switch to Btrfs in the end. 

#### The good, the bad, and the unresolved

Bacik concluded with a summary of what has worked well and what has not. He told the story of tracking down a bug where Btrfs kept reporting checksum errors when working with a specific RAID controller. Experience has led him to assume that such things are Btrfs bugs, but this time it turned out that the RAID controller was writing some random data to the middle of the disk on every reboot. This problem had been happening for years, silently corrupting filesystems; Btrfs flagged it almost immediately. That is when he started to think that, perhaps, it's time to start trusting Btrfs a bit more. 

Another unexpected benefit was the help Btrfs has provided in tracking down microarchitectural processor bugs. Btrfs tends to stress the system's CPU more than other filesystems; features like checksumming, compression, and work offloaded to threads tend to keep things busy. Facebook, which builds its own hardware, has run into a few CPU problems that have been exposed by Btrfs; that made it easy to create reproducers to send to CPU vendors in order to get things fixed. 

In general, he said, he has spent a lot of time trying to track down systemic problems in the filesystem. Being a filesystem developer, he is naturally conservative; he worries that "the world will burn down" and it will all be his fault. In almost every case, these problems have turned out to have their origin in the hardware or other parts of the system. Hardware, he said, is worse than Btrfs when it comes to quality. 

What he was most happy with, though, was perhaps the fact that most Btrfs use cases in the company have been developed naturally by other groups. He has never gone out of his way to tell other teams that they need to use Btrfs, but they have chosen it for its merits anyway. 

[PULL QUOTE:  "It's not Btrfs if there isn't an `ENOSPC` issue", he said.  END QUOTE] All is not perfect, however. At the top of his list was bugs that manifest when Btrfs runs out of space, a problem that has plagued Btrfs since the beginning; "it's not Btrfs if there isn't an `ENOSPC` issue", he said, adding that he has spent most of his career chasing these problems. There are still a few bad cases in need of fixing, but these are rare occurrences at this point. He is relatively happy, finally, with the state of `ENOSPC` handling. 

There have been some scalability problems that have come up, primarily with the WhatsApp workload as described above. These bugs have highlighted some interesting corner cases, he said, but haven't been really difficult to work out. There were also a few "weird, one-off issues" that mostly affected block subsystem maintainer Jens Axboe; "we like Jens, so we fixed it". 

At the top of the list of problems still needing resolution is quota groups; their overhead is too high, he said, and things just break at times. He plans to solve these problems within the next year. There are users who would like to see encryption at the subvolume level; that would allow the safe stacking of services with differing privacy requirements. Omar Sandoval is working on that feature now. 

Then there is the issue of RAID support, another longstanding problem area for Btrfs. Basic striping and mirroring work well, and have done so for years, he said. RAID 5 and 6 have "lots of edge cases", though, and have been famously unreliable. These problems, too, are on his list, but solving them will require "lots of work" over the next year or two.
