---
title: Debugging ARM kernels using fast interrupts
url: https://lwn.net/Articles/600359/
date: "May 29, 2014"
category: "Debugging; Development tools-Kernel debugging"
author: "May 29, 2014 This article was contributed by Daniel Thompson"
---

May 29, 2014

This article was contributed by Daniel Thompson

Recently things have pretty quiet for the interactive kernel debugging tools, with kgdb and kdb combined receiving only four patches in the last year. However, activity has started to pick up as new work inspired by Android's out-of-tree fiq_debugger has been posted for consideration. One of the key features proposed increases the robustness of kdb and kgdb by making it much harder for bugs in the system under test to prevent the user from invoking the debugger.

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Both kgdb and kdb have been included in the kernel for a long time. [Kgdb](<http://landley.net/kdocs/Documentation/DocBook/xhtml-nochunks/kgdb.html#EnableKGDB>), which is a debug stub that allows another machine to connect a source-level debugger over a serial link, was merged in 2.6.26, while [kdb](<http://landley.net/kdocs/Documentation/DocBook/xhtml-nochunks/kgdb.html#usingKDB>), after a significant rewrite, was merged into 2.6.37. The rewrite allowed kdb to reuse kgdb's breakpoint and polled I/O infrastructure in order to implement a machine-level kernel debugger that runs entirely on the machine being debugged.

On a PC, the main distinction between kdb and kgdb is that kdb can be operated from the PC's own keyboard and display. This difference is less obvious on embedded systems that seldom have their own keyboard. However the property is retained; kdb is self hosting, requiring only a terminal emulator, while kgdb requires a machine loaded up with the developer tools and the corresponding `vmlinux` file.

Both of these debug tools share common infrastructure and they also share a limitation: there are circumstances where other parts of the kernel can mask interrupts, including the one from the serial port, making it impossible for the user to manually stop the machine to debug it. When that happens, the request to stop the system never makes it from the serial port to the processor. A good example of this occurs if `spin_lock_irq()` is used incorrectly by a faulty driver causing a deadlock that cannot be studied with the debugger.

#### ARM's fast interrupt (FIQ) support

The ARM architecture includes two ways to interrupt the processor, the normal interrupt (IRQ) and the fast interrupt (FIQ). The two forms of interrupt have separate mask bits within the ARM processor status register and Linux code seldom, if ever, sets the FIQ mask bit. The processor also implements special features to reduce the overhead of FIQ handling. For example, it has a separate bank of seven registers, five of which can be used by the FIQ handler without interfering with any normal CPU registers. In addition, the FIQ vector is carefully placed within the exception vector table so that its handler can be directly executed (all other ARM exceptions must jump due to lack of space in the vector table). This means an FIQ handler, if specially crafted to use only a few registers, need not save or restore any state. The combination of seldomly being masked, reduced demultiplexing overhead (because few drivers employ FIQ), and additional hardware features give fast interrupts their name.

At the CPU level, the ARM FIQ signal is technically very similar to the x86 non-maskable interrupt (NMI), but its role within the system architecture has different historical roots. ARM FIQs were, as the name suggests, designed to rapidly service demanding peripherals or even to allow software to replace hardware (for example in synchronous serial communication). This contrasts strongly with the PC world, where the NMI has long been associated with diagnostics and other troubleshooting techniques. NMI was originally used in the IBM PC to report hardware faults such as memory parity errors. Today, watchdogs built into PC chipsets signal failure using NMI; server systems may even include a physical NMI button that can be used to invoke diagnostic features.

Most ARM systems have interrupt controllers that allow any interrupt source to be routed as either an IRQ or an FIQ. Occasionally, in embedded ARM/Linux systems, this facility it used for its original purpose of supporting a single peripheral with aggressive latency requirements. For example, the Raspberry Pi kernel uses fast interrupts to improve USB performance. However, it is much more common for the FIQ signal to never be used at all. This makes it possible to route the UART (serial port) interrupt to the FIQ signal, improving the robustness of communication between the UART and the debugger. Since the FIQ is never masked, a faulty driver would no longer be able to prevent the debugger operating normally simply by disabling interrupts.

#### Android's fiq_debugger

Google's Android team have already implemented an interactive debugger that can, optionally, take advantage of FIQ interrupts. [Fiq_debugger](<https://android.googlesource.com/kernel/common.git/+/android-3.4/arch/arm/common/fiq_debugger.c>) has a long history that dates back several years before kdb was merged into the kernel. Recently it was used in the development of many of Google's Nexus phones and tablets. On these devices, the UART is connected either to the [USB](<http://forum.xda-developers.com/showthread.php?t=1461986>) or [headphone](<http://blog.accuvant.com/jduckandryan/building-a-nexus-4-uart-debug-cable/>) sockets. These UARTs are disabled during normal use but become active when presence-detect resistors indicate that something is listening to the UART.

On devices whose application processors can support it, fiq_debugger receives and processes all user input and executes the majority of commands from the FIQ handler. This makes it extremely robust against driver bugs that leave the system unresponsive, although there are some drawbacks. In particular, an FIQ can interrupt the kernel at more or less any point during kernel execution, including during critical sections. That means that certain debug commands cannot execute safely from the FIQ handler because they might conflict with the interrupted activity. When running in the FIQ handler, even taking a spin lock can lead to a lock up if the spin lock is held by an interrupted critical section.

To solve this, fiq_debugger can drop into normal interrupt handling using ARM's software interrupt feature. This allows robust basic commands (such as single CPU stack trace) to use the FIQ signal but also to be implemented alongside richer, but slightly less robust, status-reporting features in the same debugger.

Some ARM systems do not permit routing of the UART interrupt from IRQ to FIQ. On these systems, the Android debugger remains useful to study a variety of system failures, but it does not retain the robustness of FIQ-based systems.

In addition to FIQ support, fiq_debugger contains some other unusual features that distinguish it from the existing in-kernel debug technologies. These features are motivated by the relatively hostile environment the debugger might be deployed in.

For example:

  * The UART (and the associated presence-detect circuit) might be presented with [significant noise](<http://thread.gmane.org/gmane.linux.kernel/1323496/focus=1323516>) due to the serial port being multiplexed with other activity. Noise must not cause the device to spuriously stop the world.

  * The debugger may be deployed on devices with one or more external hardware watchdogs standing by ready to reset the system should it become stalled for any reason.

  * The debugger may be deployed on production devices and cannot be used as a means to compromise user privacy. For example a hostile charging station or airline headphone service must not be able to access private user data.

Fiq_debugger has two features to counter these issues.

First, fiq_debugger's command interpreter is asynchronous. All CPUs continue to run while commands are received from the user and, on SMP systems, all of the other CPUs continue to run as usual even during command execution. This contrasts with kdb, which is a stop-the-world debugger. As soon as kdb is invoked, all CPUs in the system are brought to a halt and the system will not resume normal processing until the user issues a "go" command. 

Stop-the-world has many advantages, in particular the system cannot change state while the user is reasoning about it, however if the world were stopped accidentally due to noise (for example when inserting headphones) then this looks to the user as though the phone has crashed. In this situation, the watchdog will come to the rescue of the normal user but at a terrible cost. If a developer actually wants to stop the world, they will find that the device resets ten seconds after they started debugging it because the watchdog fired. An asynchronous implementation keeps both users happy.

Second, fiq_debugger supports only a fairly limited set of built-in commands. There are no general memory inspection commands and, apart from magic-SysRq and reboot, there is no means to divert the device from normal processing. The idea is that the passive inspection commands that do exist (stack trace, process list, irq status, dmesg, and register dump) give a reasonable chance of performing successful post-mortem analysis without much risk of leaking the user's private data.

Fiq_debugger, like kdb, offers a command that switches to kgdb mode and enables both arbitrary memory access and traditional stop-the-world step/breakpoint debugging. This command is disabled by default and can only be enabled by the root user. Despite its interesting and unique features, it seems unlikely that fiq_debugger will be merged into the kernel because its functionality overlaps so significantly with that of kdb.

#### Improving kgdb and kdb

Inspired by the Android team's work on fiq_debugger, Anton Vorontsov of Linaro developed a series of patches to implement some of the best ideas from that debugger in kdb. This includes the NMI/FIQ patchset and the reduced capability series.

The NMI/FIQ patchset introduced a generic framework to support NMI-based debuggers together with a concrete implementation for ARM that is based on FIQ. The framework allows both kgdb and kdb to be triggered from non-maskable interrupts, which brings the robustness of non-maskable debuggers to all in-kernel debug technologies.

The [generic framework](<http://thread.gmane.org/gmane.linux.kernel/1363754>) provides a means for the NMI handler to deliver characters to a special TTY driver that interacts with a real serial port driver using its polled I/O interfaces. The TTY driver allows the user to invoke kdb (or kgdb), but takes steps to avoid spuriously stopping the world due to noise by requiring a special "knock" to stop the system.

The framework was [merged into 3.7](<https://lwn.net/Articles/520001/>) but, unfortunately, the ARM-specific patches to take advantage of it never were able to get reviewer or maintainer attention despite multiple submissions. Anton moved on to other things and it falls to me to [update them with support for multi-platform kernels](<http://www.spinics.net/lists/arm-kernel/msg334349.html>) and to [fix bitrot since 3.7](<http://thread.gmane.org/gmane.linux.kernel/1703588>). 

My work brings NMI/FIQ support to STMicroelectronics STiH415 and STiH416 devices, as well as support for the ARM Versatile platform. It works well with multi-platform kernels, although Russell King has [identified some potential issues](<http://www.spinics.net/lists/arm-kernel/msg334390.html>) that require spin locks to be avoided within one of the interrupt controller callbacks. In the device tree portion of the patch, Srinivas Kandagatla [asked for better documentation](<http://www.spinics.net/lists/arm-kernel/msg334383.html>) of the new device tree bindings and King has [serious concerns](<http://www.spinics.net/lists/arm-kernel/msg334391.html>) about how FIQ-capable interrupt signals are described within device tree interrupt maps. Finally Colin Cross, one of the developers of Android's fiq_debugger, has [previously noted](<http://thread.gmane.org/gmane.linux.kernel/1676608/focus=1692875>) the need for additional changes to kgdb to fully benefit from fast interrupts on SMP platforms. In particular, the current code to stop the world uses an inter-processor IRQ to stop the other processors. That should be made to use fast interrupts to fully benefit from the robustness improvements.

Supplementing the NMI/FIQ patchset is the [reduced capability patchset](<http://thread.gmane.org/gmane.linux.kernel/1697818>), which is a means to restrict which classes of kdb commands can be used during a debug session. The permitted commands are set at boot and can be modified by the root user while the system is running. This allows kdb to be set up with a similar range of commands as fiq_debugger has, although other combinations are also possible and can be used to target different use-cases.

#### The impact of TrustZone on FIQ

So far, we have assumed that the interrupt controller provides a means to route the UART interrupt to the processor's FIQ signal and that this signal is observable by the kernel. Unfortunately, for modern ARM systems that implement TrustZone, this is not always the case.

[TrustZone](<http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.prd29-genc-009492c/index.html>) is a security technology for ARM. It works by dividing the processor into two virtual processors, each of which is considered to occupy a different "world". Peripherals, including processor-intimate peripherals such as the interrupt controller, can determine which world a memory access originates from. This is used to implement hardware-based controls that prevent the "normal world" virtual processor from interfering with the operation of the "secure world".

ARM systems with TrustZone do not introduce new interrupt signals between the interrupt controller and the processor. Instead, the processor will switch automatically from normal world to trusted world in response to the FIQ signal, meaning only the IRQ signal can be used by an operating system running in the normal world. The interrupt controller supports this division by ignoring writes to the FIQ routing registers from the normal world and returning zero for all reads.

In typical systems that employ TrustZone, the secure monitor is booted from a tamper-resistant bootloader and the kernel is later booted in normal world. The kernel can interact with the secure monitor by using a special Secure Monitor Call (SMC) instruction that operates in a similar manner to a system call.

This means that a developer working on a kernel running in normal world must rely on cooperation from the secure monitor to help pass FIQ signals to the kernel. Unfortunately, features to support this are not yet standardized and implemented in currently available secure monitors. Thankfully. there are projects like [ARM trusted firmware](<https://github.com/ARM-software/arm-trusted-firmware>) from ARM itself and the [work on trusted execution environments](<http://www.linaro.org/news/linaro-forms-security-working-group/>) by Linaro, STMicroelectronics, and NVIDIA to provide us with open-source infrastructure to prototype and develop interfaces between the kernel and the secure monitor. This should eventually open up the opportunity for developers to employ NMI-like debug techniques on almost all modern ARM systems.

#### What's next?

The NMI/FIQ patchset is relatively small, but conceals within it some fairly significant behavioral changes. Not content with proposing big changes to the default configuration of one of ARM's most common interrupt controllers, it also causes all of the debugger code to run from a non-maskable interrupt handler, thus imposing new restrictions on the use of spin locks within the debugger implementation. This means a good bit of testing will be required in order to gain sufficient confidence for the patches to be merged.

Many types of testing are needed, from simple does-it-still-boot regression testing right through to deliberately breaking the kernel and checking that the debugger can still be invoked. For example, running these tests on SMP systems will reveal the limitations of the kgdb code to stop the world, which will allow it to get fixed.

To encourage wider testing, a port to the BeagleBone Black is planned, although progress here has been frustrated slightly by the [worldwide shortage of boards](<http://beagleboard.org/blog/2014-04-13-dude-wheres-my-beaglebone-black/>). In the meantime, be aware that porting to a new board can be as little as 31 lines of code if the interrupt controller is already supported. That makes porting to other development boards a terrific way for an interested developer to get involved with this work.

It is too late for anything to happen in the 3.16 merge window, but we can hope to see at least some of these patches making their way into 3.17.
