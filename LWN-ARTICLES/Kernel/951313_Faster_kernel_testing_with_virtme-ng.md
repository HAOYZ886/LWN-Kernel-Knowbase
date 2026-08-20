---
title: "Faster kernel testing with virtme-ng"
url: https://lwn.net/Articles/951313/
date: "November 15, 2023"
category: "Development tools-Testing"
author: "By Jake Edge November 15, 2023 LPC"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
November 15, 2023

* * *

[LPC](<https://lwn.net/Archives/ConferenceByYear/#2023-Linux_Plumbers_Conference>)

Building new kernels and booting into them is an unavoidable—and time-consuming—part of kernel development. Andrea Righi works for Canonical on the Ubuntu kernel team, so he does a lot of that and wanted to find a way to speed up the task. To that end, he has been working on [virtme-ng](<https://github.com/arighi/virtme-ng>), which is a way to boot a new kernel in a virtual machine, and it does so quickly. He came to the [2023 Linux Plumbers Conference](<https://lpc.events>) (LPC) in Richmond, Virginia to introduce the project to a wider audience. 

His team builds lots of kernels for multiple architectures, often applying patches and fixes that need to be tested to ensure that they work and do not introduce regressions. There is a large testing infrastructure for that, but sometimes developers want to get more "hands on" with these kernels, for example to do a `git bisect`. There are often lots of build-reboot cycles for the work that he does, so his goal to reduce the time that they take. 

[ ![\[Andrea Righi\]](https://static.lwn.net/images/2023/lpc-righi-sm.png) ](<https://lwn.net/Articles/951508/>)

When he tests a kernel, he generally does so from a clean environment, so he will redeploy the system being used from a fresh image. That ensures the previous run has not left some kind of corruption behind that will affect the current run, but that takes time too. In general, kernel development is lacking the fast edit-compile-test loop that developers are used to from user-space development. 

His goal was to figure out how to, essentially, clone his development machine into another machine running the new kernel. He wanted the test system to have access to the root filesystem of the parent system; he also needed to be able to make changes in the test system without affecting the parent. That way there would be no redeployment needed and he would be able to, more or less instantly, access a system to run his tests. 

Once he came up with this idea, he started looking around and encountered [virtme](<https://github.com/amluto/virtme>), written by Andy Lutomirski, which did much of what he was looking for. It virtualizes the running system by creating a live snapshot of it, then starting a virtual machine with a new kernel. It exports the root filesystem to the guest using the [9p](<https://docs.kernel.org/filesystems/9p.html>) filesystem in read-only mode and allows writes to a tmpfs home directory. 

He was happy to find virtme and started using it, but it had some limitations and only really covered about 80% of the features that he needed. The read-only root filesystem meant that additional packages could not be installed on the test system, which limited the testing he could do. In addition, 9p had rather poor performance, so that, for example, a `git diff` in the kernel tree on the guest would take five minutes to run. That limited the testing he could do, but it also affected the boot time, which was longer than he wanted. It was taking around 15 seconds to boot to a console prompt, which he thought could be improved. 

Beyond that, virtme is not being maintained; Lutomirski no longer has the time to do so. Pull requests (PRs) from Righi and others were not being acted on, so he contacted Lutomirski, who encouraged him to create a fork. Righi created virtme-ng, went through the PRs on virtme and merged those that he liked. 

Since then, he has added a number of different features, starting with using [virtiofs](<https://virtio-fs.gitlab.io/>) and [overlayfs](<https://docs.kernel.org/filesystems/overlayfs.html>) to provide a copy-on-write export of the entire host filesystem, instead of using 9p. Virtiofs requires more infrastructure, such as a FUSE daemon running on the host, but it performs much better than 9p. Overlayfs then allows the guest to write to the filesystem, so that it can install packages, say, without actually affecting the host. 

Another fairly major change was the adoption of the QEMU [microvm](<https://www.qemu.org/docs/master/system/i386/microvm.html>) virtual-machine type. That was suggested to him by someone on Twitter when he had tweeted about his boot-time-reduction progress. Righi also switched virtme-ng to use a custom init script written in Rust to replace the Bash init script that virtme uses. The Bash script was not much of a maintenance problem, but he had an interest in learning Rust, so he rewrote it in that language, which turned out to have some large benefits in terms of boot time. 

Simply switching from 9p to virtiofs made a huge difference in terms of I/O speed; the `git diff` dropped from 284s to just 1.7s. That operation generates an enormous amount of I/O, which can be done efficiently in the FUSE-based virtiofs, because it puts the results into memory shared with the guest; with 9p, each I/O operation and result was a separate message using the 9p protocol. In addition, the boot time went from 6.2s to 5.2s, which was less dramatic because there is a lot less I/O for booting the system. 

Adding overlayfs on top of virtiofs allowed the guest to access and write to the host filesystem—without making any permanent changes. It uses a tmpfs as the upper directory, so when the VM exits, any changes made are gone. He did encounter a problem with overlayfs using the `O_NOATIME` flag, which caused permission errors for the virtiofs FUSE daemon, but that has now been fixed in the virtiofs upstream. 

The microvm machine type is just an option to QEMU that provides a machine optimized for boot time and memory size. It does not have PCI or ACPI, which saves time for probing and the like. Adding that into the mix dropped the boot time from 5.2s to 3.8s. That reduction is not huge in absolute terms, but it makes a large difference if you are using virtme-ng to boot lots of different kernels, he said. 

Using the new Rust virtme-ng-init further reduced the boot time to 1.2s. Those boot times are measured from the time he types `vng` to start the VM with a new kernel until he gets to a prompt where he can start typing commands into the guest. "That is quite amazing." For a bisect run, it can make a huge difference, for example. 

Righi considered doing a demo in the talk, but was concerned about it not going well. He has made a [YouTube video](<https://www.youtube.com/watch?v=3sDkVuXVw9A>) of a live virtme-ng demo and he described some of the things that he showed in it. You can do more than just type commands at the guest's shell once the VM boots; you can run scripts in the guest from the `vng` command line, for example. Virtme-ng has standard input and output set up so that you can run a command on the host, piped to a VM with a certain kernel, piped to a VM with a different kernel, and so on, which allows automating testing of a set of kernels. It can also be used to run graphical applications; his final demo in the video is running the Steam client on the guest and showing that the game is perfectly playable on the host display. 

He hopes that virtme-ng can bring the fast edit-compile-test cycle to kernel development. Even just over a second is nice, but that is measured on his laptop; on his more powerful home server, he can break the one-second barrier, with boot times of 0.8-0.9s. The tool is meant to be backward compatible with virtme, so he considers any problems in compatibility to be a bug that needs to be fixed; he encouraged anyone that encountered a problem of that sort to report it to the GitHub repository. 

Righi has the opportunity to work with students, who are generally quite excited to be able to change and build their own kernels, but have trouble with deploying those kernels onto VMs. When he shows them virtme-ng, it really helps to get them up and running quickly; he thinks the tool can be a way to smooth the path for any new kernel developers. He also [gave a talk](<https://lwn.net/Articles/935180/#eco>) at the OSPM power management and scheduling conference in April about being "more carbon efficient" by using virtme-ng to reduce the amount of time spent doing redeployment and the like for continuous-integration (CI) systems. 

The goal of his talk, Righi said, was to raise awareness of the tool in order to increase the user base. There are some people at SUSE using the tool, including for testing live patches, some Google developers use it as well; a Debian developer has started packaging virtme-ng, so it is available for Ubuntu and other Debian-based distributions. There is work going on to provide an RPM package for virtme-ng; with that and the Debian work, most distributions should be covered soon. He would like to collect more feedback from users as it basically does what he needs now, but there are probably other use cases that could be handled. 

It is not a priority, but he would like to add full systemd support to the tool; systemd does not work correctly currently because it has its own state in the host system that confuses a systemd that gets started for the guest. A systemd-based VM will not start as quickly as the custom init script, but it would bring capabilities that the current guests do not have. 

For example, he wanted to run the snap server in the guest, but it is a systemd-based daemon. He has a hack to trick the snap daemon into thinking that there is a systemd running, but he would like a better solution. The snap mode (which is enabled with a command-line option) works and is generally stable, but with better systemd support, both it and Flatpak would be cleanly supported by virtme-ng. 

An audience member asked about supporting alternate user IDs, so that a root filesystem from a tar file could be used instead of the host filesystems. Righi said that there is some support for that use case, but that the user ID problem is not solved cleanly; [ID-mapped mounts](<https://lwn.net/Articles/896255/>) was suggested as a potential path for handling that better. Another question was about adoption by other open-source projects; Righi said that [Mutter](<https://gitlab.gnome.org/GNOME/mutter>) is using virtme-ng as part of its testing. There is also an unnamed company using the tool to test webcams, which surprised him, but it turns out the company is using QEMU options to pass a USB device from the host to the guest for testing multiple kernels with the webcam. 

The VMs can be used with GDB, he said responding to another query; you can set breakpoints and such, as well as create a crash dump if desired. Another attendee asked about support for signed modules; Righi said that looking into that is on his "secret to-do list", so he was happy to get the question to spur him to look at it sooner. 

[I would like to thank LWN's travel sponsor, the Linux Foundation, for assistance with my travel costs to Richmond for LPC.]
