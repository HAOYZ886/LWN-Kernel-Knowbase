---
title: "Control groups, part 2: On the different sorts of hierarchies"
url: https://lwn.net/Articles/604413/
date: "July 9, 2014"
category: "Control groups-LWNs guide to"
author: "July 9, 2014 This article was contributed by Neil Brown Control groups"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

July 9, 2014

This article was contributed by Neil Brown

* * *

[Control groups](<https://lwn.net/Articles/604609/>)

Hierarchies are everywhere. Whether this is a deep property of the universe or simply the result of the human thought process, we see hierarchies wherever we look, from the URL bar that your browser displays ([or maybe doesn't](<http://tech.slashdot.org/story/14/05/03/007209/could-googles-test-of-hiding-complete-urls-in-chrome-become-a-standard>)) to the pecking order in the farm yard. There is a [fun fact](<http://en.wikipedia.org/wiki/Wikipedia:Getting_to_Philosophy>) that if you click on the first link in the main text of a Wikipedia article, and then repeat that on each following article, you eventually get to [Philosophy](<http://en.wikipedia.org/wiki/Philosophy>), though this is apparently only true 94.52% of the time. Nonetheless it suggests that all knowledge can be arranged hierarchically underneath the general heading of "Philosophy".

Control groups (cgroups) allow processes to be grouped hierarchically and the specific details of this hierarchy are one area where cgroups have both undergone change and received criticism. In our [ongoing effort](<https://lwn.net/Articles/604609/>) to understand cgroups enough to enjoy the debates that regularly spring up, it is essential to have an appreciation of the different ways a hierarchy can be used, so we can have some background against which to measure the hierarchy in cgroups.

I find that an example from my past raises some relevant issues that we can then see play out in some more generally familiar filesystem hierarchies and that we can be prepared to look for in cgroup hierarchies.

#### Hierarchies in computer account privileges

In a previous role as a system administrator for a modest-sized computing department at a major Australian university, we had a need for a scheme to impose various access controls on, and provide resource allocations to, a wide variety of users: students, both undergraduate and post-graduate, and staff, both academic and professional. Already it is clear that a hierarchy is presenting itself, with room for further subdivisions between research and course-work students, and between technical and clerical professional staff.

Largely orthogonal to this hierarchy were divisions of the school into research groups and support groups (I worked in the Computing Support Group) together with a multitude of courses that were delivered, each loosely associated with a particular program (Computer Engineering, Software Engineering, etc.) at a particular year level. Within each of the different divisions and courses there could be staff in different roles as well as students. Some privileges best aligned with the role performed by the owner of the account, so staff received a higher printing allowance than students. Others aligned with the affiliation of the account owner — a particular printer might be reserved for employees in the School Office who had physical access and used it for printing confidential transcripts. Similarly, students in some particular course had a genuine need for a much higher budget of color printouts.

To manage all of this we ended up with two separate hierarchies that were named "Reason" (which included the various roles, since they were the reason a computer account was given) and "Organisation" (identifying that part of the school in which the role was active). From these two we formed a cross product such that for each role and for each affiliation there was, at least potentially, a group of user accounts. Each account could exist in several of these groups, as both staff and students could be involved in multiple courses, and some senior students might be tutors for junior courses. Various privileges and resources could be allocated to individual roles and affiliations or intersections thereof, and they would be inherited by any account in the combined hierarchy.

#### Manageable complexity

Having a pair of interconnected hierarchies was certainly more complex than the single hierarchy that I was hoping for, but it had one particular advantage: it worked. It was an arrangement that proved to be very flexible and we never had any trouble deciding where to attach any particular computer account. The complexity was a small price to play for the utility.

Further, the price was really quite small. While creating the cross product of two hierarchies by hand would have been error prone, we didn't have to do that. A fairly straightforward tool managed all the complexity behind the scenes, creating and linking all the intermediate tree nodes as required. While working with the tree, whether assigning permissions or resources or attaching people to various roles or affiliations, we rarely needed to think about the internal details and never risked getting them wrong.

This exercise left me with a deep suspicion of simple hierarchies. They are often tempting, but just as often they are an over-simplification. So the first lesson from this tale is that a little complexity can be well worth the cost, particularly if it is well-chosen and can be supported by simple tools.

#### Two types of hierarchy

The second lesson from this exercise is that the two hierarchies weren't just different in detail; they had quite different characters.

The "Reason" hierarchy is what might be called a "classification" hierarchy. Every individual had their own unique role but it is useful to group similar roles into classes and related classes into super-classes. A widely known hierarchy that has this same property is the [Linnaean taxonomy](<http://en.wikipedia.org/wiki/Linnaean_taxonomy>) of [Biological classification](<http://en.wikipedia.org/wiki/Biological_classification>), which is a hierarchy of life forms with seven main ranks of Kingdom, Phylum, Class, Order, Family, Genus, and Species.

With this sort of hierarchy all the members belong in the leaves. In the biological example, all life forms are members of some species. We may not know (or be able to agree) which species a particular individual belongs to, but to suggest that some individual is a member of some Family, but not of any Genus or Species doesn't make sense. It would be at best an interim step leading to a final classification.

The "Organisation" hierarchy has quite a different character. The different research groups did not really represent a classification of research interests, but were a way to organize people into conveniently sized groups to distribute management. Certainly the groups aligned with people's interests where possible, but it was not unheard of for someone to be assigned to a group not because they naturally belonged, but because it was most convenient. To some extent the grouping exists for some separate purpose and members are placed in groups to meet that purpose. This contrasts with a "classification" where each "class" exists only to contain its members.

An organizational hierarchy has another important property: it is perfectly valid for internal nodes to contain individuals. The Head of School was the head of the whole school, and so belonged at the top of the hierarchy. Similarly, a program director could reasonably be associated with the program as a whole without being specifically associated with each of the courses in the program. In many organizations, the leader or head of each group is a member of the group one step up in the organizational hierarchy, which affirms this pattern.

These two different types of hierarchy are quite common and often get mingled together. Two places that we can find them that will be familiar to many readers are the "sysfs" filesystem in Linux, and the source code tree for the Linux kernel.

#### Devices in `/sys`

The "sysfs" filesystem (normally mounted at `/sys`) is certainly a hierarchy — as that is how filesystems work. While sysfs currently contains a range of different objects including modules, firmware information, and filesystem details, it was originally created for devices and it is only the devices that will concern us here.

There are, in fact, three separate hierarchical arrangements of devices that all fit inside sysfs, suggesting that each device should have three parents. As devices are represented as directories, this is clearly not possible, since Unix directories may have only one parent. This conundrum is resolved thorough the use of symbolic links (or "symlinks") with implicit, rather than explicit, links to parents. We will start exploring with the hierarchies that are held together with symlinks.

The hierarchy rooted at `/sys/dev` could be referred to as the "legacy hierarchy". From the early days of Unix there have been two sorts of devices: block devices and character devices. These are represented by the various device-special-files that can normally be found in `/dev`. Each such file identifies as either a block device or a character device and has a major device number indicating the general class of device (e.g. serial port, parallel port, disk or tape drive) and a minor number that indicates which particular device of that class is the target.

This three-level hierarchy is exactly what we find under `/sys/dev`, though a colon is used, rather than a slash, to separate the last two levels. So `/sys/dev/block/8:0` (block device with major number 8 and minor number 0) is a symbolic link to a directory representing the device also known as "`sda`". If we start in that directory and want to find the path from `/sys/dev`, we can find the last two components ("`8:0`") by reading the "`dev`" file. Determining that it is a block device is less straightforward, though the presence of a "`bdi`" (block device info) directory is a strong hint. 

This hierarchy is particularly useful if all you have is the name of a device file in `/dev`, or an open file descriptor on such a device. The `stat()` or `fstat()` system calls will report the device type and the major and minor numbers, and these can trivially be converted to a path name in `/sys/dev`, which can lead to other useful information about the device. 

The second symlink-based hierarchy is probably the most generally useful. It is rooted at `/sys/class` and `/sys/bus`, suggesting that there really should be another level in there to hold both of these. There are [plans](<http://lxr.free-electrons.com/source/Documentation/sysfs-rules.txt?v=3.15#L100>) to combine both of these into a new `/sys/subsystem` tree, though as those plans are at least seven years old, I'm not holding my breath. One valuable aspect of these plans that is already in place is that each device directory has a `subsystem` symlink that points back to either the `class` or `bus` tree, so you can easily find the parent of any device within this hierarchy.

The `/sys/class` hierarchy is quite simple, containing a number of device classes each of which contains a number of specific devices with links to the real device directory. As such, it is conceptually quite similar to the legacy hierarchy, just with names instead of numbers. The `/sys/bus` hierarchy is similar, though the devices are collected into a separate `devices` subdirectory allowing each `bus` directory to also contain `drivers` and other details.

The third hierarchy for organizing devices is a true directory-based hierarchy that doesn't depend on symlinks. It is found in `/sys/devices` and has a structure that, in all honesty, is rather hard to describe.

The overriding theme to the organization is that it follows the physical connectedness of devices, so if a hard drive is accessed via a USB port with the USB controller attached to a PCI bus, then the path though the hierarchy to that hard drive will first find the PCI bus, and then the USB port. After the hard drive will be the "block" device that provides access to the data on the drive, and then possibly subordinate devices for partitions.

This is an arrangement that seems like a good idea until you realize that some devices get control signals from one place (or maybe two if there is a separate reset line) and power supply from another place, so a simple hierarchy cannot really describe all the interconnectedness. This is an issue that was [widely discussed](<http://lists.linuxfoundation.org/pipermail/ksummit-discuss/2014-May/000430.html>) in preparation for this year's Kernel Summit.

When examining these hierarchies from the perspective of "classification" versus "organization", some fairly clear patterns emerge. The `/sys/dev` hierarchy is a simple classification hierarchy, though possibly overly simple as many devices (e.g. network interfaces) don't appear there. The `/sys/class` part of the subsystem hierarchy is similarly a simple classification, though it is more complete.

The `/sys/bus` part of the subsystem hierarchy is also a simple two-level classification, though the presence of extra information for each bus type, such as the `drivers` directory, confuse this a little. Devices in the `class` hierarchy are classified by what functionality they provide (net, sound, watchdog, etc). Devices in the `bus` hierarchy are classified by how they are accessed and represent different addressable units rather than different functional units. The extra entries in the `/sys/bus` subtree allow some control over what functionality (represented by a driver and realized as a class device) is requested of each addressable unit.

With this understood, it is hierarchically a simple two-level classification.

The `/sys/devices` hierarchy is indisputably an organizational hierarchy. It contains all the class devices and all the bus devices in a rough analog of the physical organization of devices. When there is no physical device, or it is not currently represented on any sort of bus, devices are organized into `/sys/devices/virtual`.

Here again we see that both a classification hierarchy and an organization hierarchy for the same objects can be quite useful, each in its own way. There can be some complexity to working with both, but if you follow the [rules](<http://lxr.free-electrons.com/source/Documentation/sysfs-rules.txt>), it isn't too bad.

#### The Linux kernel source tree

For a significantly different perspective on hierarchies, we can look at the Linux kernel source code tree, though many evolving source code trees could provide similar examples. This hierarchy is more about organization than classification, though, as with the research groups discussed earlier, there is generally an attempt to keep related things together when convenient.

There are two aspects of the hierarchy that are worth highlighting, as they illustrate choices that must be made — consciously or unconsciously.

At the top level, there are directories for various major subsystems, such as `fs` for filesystems (and also file servers like `nfsd`), `mm` for memory management, `sound`, `block`, `crypto`, etc. These all seem like reasonable classifications. And then there is `kernel`. Given that all of Linux is an operating system kernel, maybe this bit is the kernel of the kernel?

In reality, it is various distinct bits and pieces that don't really belong to any particular subsystem, or they are subsystems that are small enough to only need one or two files. In some cases, like the `time` and `sched` directories, they are subsystems which were once small enough to belong in `kernel` and have grown large enough to need their own directory, but not bold enough to escape from the `kernel` umbrella.

The `fs` subtree has a similar set of files. Most of `fs` is the different filesystems and there are a few support modules that get their own subdirectory, such as `exportfs`, which helps various file servers, and `dlm`, which supports locking for cluster filesystems. However, in `fs` is also an _ad hoc_ collection of C files providing services to filesystems, or implementing the higher-level system call interfaces. These are exactly like the code that appears in `kernel` (and possibly `lib`) at the top level. However, in `fs` there is no subdirectory for miscellaneous things, it all just stays in the top level of `fs`.

There is not necessarily a right answer as to whether everything should be classified into its own leaf directory (following the `kernel` model), or whether it is acceptable to store source code in internal directories (as is done in `fs`). However, it is a choice that must be made, and is certainly something to hold an opinion on when debating hierarchies in cgroups.

The kernel source tree also contains a different sort of classification: scripts live in the `scripts` directory, firmware lives in the `firmware` directory, and header files live in the `include` directory — except when they don't. There has been a tendency in recent years to move some header files out of the `include` directory tree and closer to the C source code files that they are related to. To make this more concrete, let's consider the example of the NFS and the ext3 filesystems.

Each of these filesystems consist of some C language files, some C header files, and assorted other files. The question is: should the header files for NFS live with the header files for ext3 (header files together), or should the header files for NFS live with the C language files for NFS (NFS files together)? To put this another way, do we need to use the hierarchy to classify the header files as different from the other files, or are the different names sufficient?

There was a time when most, if not all, header files were in the `include` tree. Today, it is very common to find include files mixed with the C files. For ext3, a [big change](<http://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=4613ad180d19082f99551477dcb13cb23d23661b>) happened in Linux 3.4, when all four header files were moved from `include/linux/` into a single file with the rest of the ext3 code: `fs/ext3/ext3.h`.

The point here is that classification is quite possible without using a hierarchy. Sometimes hierarchical classification is perfect for the task. Sometimes it is just a cumbersome inconvenience. Being willing to use hierarchy when, but only when, it is needed, makes a lot of sense.

#### Hierarchies for processes

Understanding cgroups, which is the real goal of this series of articles, will require some understanding of how to manage groups of processes and what role hierarchy can play in that management. None of the above is specifically about processes, but it does raise some useful questions or issues that we can consider when we start looking at the details of cgroups:

  * Does the simplicity of a single hierarchy outweigh the expressiveness of multiple hierarchies, whether they are separate (as in sysfs) or interconnected (as in the account management example)?

  * Is the overriding goal to classify processes, or simply to organize them? Or are both needs relevant, and, if so, how can we combine them?

  * Could we allow non-hierarchical mechanisms, such as symbolic links or file name suffixes, to provide some elements of classification or organization?

  * Does it ever make sense for processes to be attached to internal nodes in the hierarchy, or should they be forced into leaves, even if that leaf is simply a `miscellaneous` leaf.

In the hierarchy of process groups we looked at [last time](<https://lwn.net/Articles/603762/>), we saw a single simple hierarchy that classified processes, first by login session, and then by job group. All processes that were in the hierarchy at all were in the leaves, but many processes, typically system daemons that never opened a tty at all, were completely absent from the hierarchy.

To begin to find answers to these questions in a more modern setting, we need to understand what cgroups actually does with processes and what the groups are used for. In the next article we will start answering that question by taking a close look at some of the cgroups "subsystems", which include resource controllers and various other operations that need to treat a set of processes as group.
