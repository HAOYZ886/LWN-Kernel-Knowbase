---
title: "LPC: Booting Linux in five seconds"
url: https://lwn.net/Articles/299483/
date: "September 22, 2008"
category: Bootstrap process; Fast booting
author: "September 22, 2008 This article was contributed by Don Marti"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

September 22, 2008

This article was contributed by [Don Marti](<http://zgp.org/~dmarti/>)

At the Linux Plumbers Conference Thursday, [Arjan van de Ven](<http://www.fenrus.org/>), Linux developer at Intel and author of PowerTOP, and Auke Kok, another Linux developer at Intel's [Open Source Technology Center](<http://oss.intel.com/en-us/>), demonstrated a Linux system booting in five seconds. The hardware was an Asus EEE PC, which has solid-state storage, and the two developers beat the five second mark with two software loads: one modified [Fedora](<http://fedoraproject.org/>) and one modified [Moblin](<http://www.moblin.org/>). They had to hold up the EEE PC for the audience, since the time required to finish booting was less than the time needed for the projector to sync.

How did they do it? Arjan said it starts with the right attitude. ""It's not about booting faster, it's about booting in 5 seconds"". Instead of saving a second here and there, set a time budget for the whole system, and make each step of the boot finish in its allotted time. And no cheating. ""Done booting means CPU and disk idle"", Arjan said. No fair putting up the desktop while still starting services behind the scenes. (An audience member pointed out that Microsoft does this.) The "done booting" time did not include bringing up the network, but did include starting NetworkManager. A system with a conventional hard disk will have to take longer to start up: Arjan said he has run the same load on a ThinkPad and achieved a 10-second boot time.

Out of the box, Fedora takes 45 seconds from power on to [GDM](<http://www.gnome.org/projects/gdm/>) login screen. A tool called [Bootchart](<http://www.bootchart.org/>), by Ziga Mahkovec, offers some details. In a Bootchart graph of the Fedora boot (fig. 1), the system does some apparently time-wasting things. It spends a full second starting the loopback device--checking to see if all the network interfaces on the system are loopback. Then there's two seconds to start "sendmail." ""Everybody pays because someone else wants to run a mail server"", Arjan said, and suggested that for the common laptop use case--an SMTP server used only for outgoing mail--the user can simply run [ssmtp](<http://packages.debian.org/stable/mail/ssmtp>).

[ ![\[Fedora bootchart\]](https://static.lwn.net/images/fastboot/fastboot-f1_sm.png) ](<http://lwn.net/Articles/299538/>)  
---  
**Figure 1**  
  
Another time-consuming process on Fedora was "setroubleshootd," a [useful tool](<http://danwalsh.livejournal.com/7995.html>) for finding problems with Security Enhanced Linux (SELinux) configuration. It took five seconds. Fedora was not to blame for everything. Some upstream projects had puzzling delays as well. The X Window System runs the C preprocessor and compiler on startup, in order to build its keyboard mappings.

Ubuntu's boot time is about the same: two seconds shorter (fig. 2). It spends 12 seconds running modprobe running a shell running modprobe, which ends up loading a single module. The tool for adding license-restricted drivers takes 2.5 seconds--on a system with no restricted drivers needed. ""Everybody else pays for the binary driver"", Arjan said. And Ubuntu's GDM takes another 2.5 seconds of pure CPU time, to display the background image.

[ ![\[Ubuntu bootchart\]](https://static.lwn.net/images/fastboot/fastboot-f2_sm.png) ](<http://lwn.net/Articles/299540/>)  
---  
**Figure 2**  
  
Both distributions use splash screens. Arjan and Auke agreed, ""We hate splash screens. By the time you see it, we want to be done"". The development time that distributions spend on splash screens is much more than the Intel team spent on booting fast enough not to need one.

### How they did it: the kernel

Step one was to make the budget. The kernel gets one second to start, including all modules. "Early boot" including init scripts and background tasks, gets another second. X gets another second, and the desktop environment gets two.

The kernel has to be built without initrd, which takes half a second with nothing in it. So all modules required for boot must be built into the kernel. ""With a handful of modules you cover 95% of laptops out there"", Arjan said. He suggested building an initrd-based image to cover the remaining 5%.

Some kernel work made it possible to do asynchronous initialization of some subsystems. For example, the modified kernel starts the Advanced Host Controller Interface (AHCI) initialization, to handle storage, at the same time as the Universal Host Controller Interface (UHCI), in order to handle USB (fig.3). ""We can boot the kernel probably in half a second but we got it down to a second and we stopped"", Arjan said. The kernel should be down to half a second by 2.6.28, thanks to a brand-new fix in the AHCI support, he added.

[ ![\[Asynchronous hardware
init\]](https://static.lwn.net/images/fastboot/fastboot-f3_sm.png) ](<http://lwn.net/Articles/299542/>)  
---  
**Figure 3**  
  
One more kernel change was a small patch to support readahead. The kernel now keeps track of which blocks it has to read at boot, then makes that information available to userspace when booting is complete. That enables readahead, which is part of the early boot process.

### How they did it: readahead and init

Fedora uses [Upstart](<http://upstart.ubuntu.com/>) as a replacement for the historic "init" that traditionally is the first userspace program to run. But the Intel team went back to the original init. The order of tasks that init handles is modified to do three things at the same time: first, an "sReadahead" process, to read blocks from disk so that they're cached in memory, second, the critical path: filesystem check, then the [D-Bus](<http://www.freedesktop.org/wiki/Software/dbus>) inter-process communication system, then X, then the desktop. And the third set of programs to start is the [Hardware Abstraction Layer (HAL)](<http://www.freedesktop.org/wiki/Software/hal>), then the [udev](<http://www.kernel.org/pub/linux/utils/kernel/hotplug/udev.html>) manager for hot-plugged devices, then networking. udev is used only to support devices that might be added later--the system has a persistent, old-school /dev directory so that boot doesn't depend on udev.

The arrangement of tasks helps get efficient use out of the CPU. For example, X delays for about half a second probing for video modes, and that's when HAL does its CPU-intensive startup (fig. 4).

[ ![\[Parallel tasks\]](https://static.lwn.net/images/fastboot/fastboot-f4_sm.png) ](<http://lwn.net/Articles/299545/>)  
---  
**Figure 4**  
  
In a graph of disk and CPU use, both are at maximum for most of the boot time, thanks to sReadahead. When X starts, it never has to wait to read from disk, since everything it needs is already in cache. sReadahead is based on Fedora [Readahead](<http://dailypackage.fedorabook.com/index.php?/archives/59-Wednesday-Why-Readahead.html>), but is modified to take advantage of the kernel's new list of blocks read. sReadahead is to be released next week on [moblin.org](<http://www.moblin.org/>), and the kernel patch is intended for mainline as soon as Arjan can go over it with ext3 filesystem maintainer Ted Ts'o. (Ted, in the audience, offered some suggestions for reordering blocks on disk to speed boot even further.)

There's a hard limit of 75MB of reads in order to boot, set by the maximum transfer speed of the Flash storage: 3 seconds of I/O at 25MB/s. So, ""We don't read the whole file. We read only the pieces of the file we actually use"", Arjan said. sReadahead uses the "idle" I/O scheduler, so that if anything else needs the disk it gets it. With readahead turned off, the system boots in seven seconds, but with readahead, it meets the target of five.

X is still problematic. ""We had to do a lot of damage to X"", Arjan said. Some of the work involved eliminating the C compiler run by re-using keyboard mappings, but other work was more temporary. The current line of X development, though, puts more of the hardware detection and configuration into the kernel, which should cut the total startup time. Since part of the kernel's time budget is already spent waiting for hardware to initialize, and it can initialize more than one thing at a time, it's a more efficient use of time to have the kernel initialize the video hardware at the same time it does USB and ATA. X developer Keith Packard, in the audience and also an Intel employee, offered help. Setting the video mode in the kernel would not let the kernel initialize it at the same time as the rest of the hardware, as shown in figure 3. The fast-booting system does not use GDM but boots straight to a user session, running the XFCE desktop environment. Instead of GDM, Arjan said later, a distribution could boot to the desktop session of the last user, but start the screensaver right away. If a different user wanted to log in, he or she could use the screensaver's "switch user" button.

[ ![\[5 second boot\]](https://static.lwn.net/images/fastboot/fastboot-f5_sm.png) ](<http://lwn.net/Articles/299546/>)  
---  
**Figure 5**  
  
In conclusion, Arjan said, ""Don't settle for 'make boot faster.' It's the wrong question. The question is 'make boot fast'"". And don't make all users wait because a few people run a filesystem that requires a module or sendmail on their laptops. ""Make it so you only pay the price if you use the feature"". Distributions shouldn't have to maintain separate initrd-based and initrd-free kernel packages, he said later. The kernel could try to boot initrd-free, then fall back if for whatever reason it couldn't see /sbin/init, as might happen if it's missing the module needed to mount the root filesystem.

PowerTOP spawned a flurry of power-saving hacks from all areas of the Linux software scene. The combination of Bootchart, readahead, and a five-second target looks likely to set off a friendly boot time contest among Linux people as well. At the conference roundup Friday, speaker Kyle McMartin announced that both Fedora and Ubuntu have fixed some delays in their boot process, and there was much applause.

FIGURE CREDIT: Arjan van de Ven and Auke Kok, Intel
