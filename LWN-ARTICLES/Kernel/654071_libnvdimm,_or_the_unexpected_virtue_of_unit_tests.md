---
title: "libnvdimm, or the unexpected virtue of unit tests"
url: https://lwn.net/Articles/654071/
date: "August 12, 2015"
category: Development tools
author: "August 12, 2015 This article was contributed by Dan J. Williams"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

August 12, 2015

This article was contributed by Dan J. Williams

In [the July 9th edition](<https://lwn.net/Articles/650299/>), Jonathan Corbet said: ""Over time, the kernel development community has indeed gotten better at merging code that does not require fixing later in the development cycle."" That’s good. Now how do we accelerate this trend? One need only look at the recent inclusion of [kselftests](<https://lwn.net/Articles/608959/>) in the kernel or the ongoing development of the venerable [xfstests](<http://oss.sgi.com/cgi-bin/gitweb.cgi?p=xfs/cmds/xfstests.git;a=summary>) suite to understand the current state of the art of Linux kernel test capabilities. However, what becomes readily apparent when looking at these test suites is that they are limited to tests that can be solely driven by user-space stimuli, such as generic sysfs attributes, system calls, and on-disk metadata formats. 

Granted this is a very large test surface, but it still leaves a coverage hole for a body of code that, to date, has only executed in the presence of specific hardware (be it virtual or physical), makes up the majority of the kernel source, and sees the highest rate of change: device drivers. The [libnvdimm](<https://lwn.net/Articles/649840/>) subsystem is one of the latest additions to the kernel's driver infrastructure; it layers services over persistent memory to make access to that memory safer and more reliable. To the best of this author’s knowledge, libnvdimm is the first upstream device-driver subsystem to integrate in-kernel interface mocking techniques in support of unit testing. This article walks through how and why this came to pass, as well as the challenge and promise of driver unit testing going forward. 

#### The trouble with TRBs

While libnvdimm is the first upstream inclusion of this technique, the first overall attempt to use mocking for driver development was part of [a patch kit](<http://marc.info/?l=linux-usb&m=140872781311287&w=2>) for rearchitecting transfer-request-block (TRB) handling in the USB 3.0 host controller (XHCI) driver. It is helpful to understand the problem the patch kit was trying to solve to see why mocking was proposed as a way to validate the implementation. 

Like many I/O devices, XHCI controllers have a ring buffer for describing transfers between memory and the device. An XHCI ring buffer entry is called a TRB, and it describes a transfer of a contiguous range of memory. The total set of TRBs in an XHCI ring need not be contiguous in memory as a ring can be divided into segmented groups of TRBs. These individual ring segments (TRB-segments) can be chained together with "link" TRBs. In the case when a single TRB is only able to describe a subset of the full transfer request (i.e., when performing scatter-gather to multiple discontiguous memory ranges), multiple TRBs can be ganged together to form the transfer-descriptor (TD). This architecture is fairly straightforward until one considers the pile of constraints imposed by the I/O request, the host controller, and the attached device: 

  * A TD may only cross a TRB-segment boundary (by way of a link-TRB) at a device-defined boundary called the MBP (max burst payload). This means that if the MBP is 1KB and we hit the end of a TRB segment after queuing a 512-byte TRB, things will fail; the driver should have instead enqueued an early link-TRB rather than the 512-byte "transfer"-TRB. 

  * A single TRB may not cross a 64KB boundary in memory. 

  * Each discontiguous range to be transferred must have its own TRB (implied since the driver is performing scatter-gather DMA in a TD). 

If you are still reading this article after that acronym soup then perhaps you are also starting to imagine the array of USB peripherals and contrived I/O scenarios needed to fully exercise all the corner-case conditions presented by the these overlapping constraints. Instead, these conditions can be explicitly tested by mocking all the interfaces and input conditions that surround the TRB queuing implementation. This is the approach taken in the patch titled "[xhci: unit test ring enqueue/dequeue routines](<http://marc.info/?l=linux-usb&m=140872785411304&w=2>)". 

#### Move fast and break things

The motivation for exercising the XHCI TRB handling code with a unit test was not driven solely by the complexities of the hardware implementation. After all, the kernel has been quite successful in the absence of widespread driver unit testing since its inception. However, that success has come in part from shipping buggy code to users and fixing it quickly when problems arise. Can this arrangement continue to scale into the future? 

While the number of new devices and platforms increases and new developers join the kernel development community, the number of active code reviewers remains relatively flat. A maintainer’s most important job is to say "no" and when that "no" can be automated by a robot like checkpatch, the 0day-kbuild infrastructure, or a unit test suite, those limited reviewer resources can scale out to higher-order review tasks. The power of unit tests to allow a project to scale was made clear by this author’s time working at Facebook. The risk incurred by onboarding new developers at a high rate was [mitigated in part](<http://www.infoq.com/presentations/Facebook-Release-Process>) by extensive unit tests. This discipline shifts the "break things" aspect of "move fast" to the developer’s workstation rather than the production environment. 

#### From read-only to rewrite

The potential of unit testing to move bug detection earlier in the development cycle is common knowledge. What was surprising during the development of libnvdimm were the occasions where the process of thinking through and developing a test resulted in early detection of warts in the architecture. This happened several times in small ways; early drafts of an approach that seemed suitable at first later felt kludgy after writing the test, and these were ultimately thrown away. A larger example was a reorganization of the block-translation-table (BTT) implementation from a driver to a library. 

First, here is a quick primer on the responsibilities of the BTT code. Non-volatile memory devices provide persistent storage and can be used anywhere a disk is used. Whereas disks perform I/O operations in units of sectors (512-bytes for example) that are atomic with respect to power loss, memory devices perform I/O in units of individual bytes. With this change comes an important question: what happens if a sector update is interrupted (by a power failure, say) partway through? 

Developers (and users) would like to see non-volatile devices be "atomic with respect to power loss," meaning that a write to a sector completes in full, or not at all, and software can depend on not seeing partially updated sectors upon recovery from a crash or power failure event. The BTT is a software mechanism to layer these atomic-sector-update semantics on top of byte-accessible memory. The initial architecture for BTT followed in the footsteps of other kernel drivers that layer various storage semantics on top of another storage device. Device Mapper (DM - volume management) and Multiple Devices (MD - RAID) use a stacked block device arrangement for this purpose. 

This stacked arrangement served the implementation well up until the point where code was added to handle occasions when the backing memory for a BTT is discovered to be read-only. The code gymnastics needed to keep the block-device-read-only flags synchronized between the "fronting" BTT block-device and the "backing" non-volatile memory block-device received special attention during the upstream review cycle. It became clear that the implementation would be cleaner with the stacking removed and instead have the BTT built as an intrinsic property of the storage device. 

In the typical development model, sans unit tests, the fact that the read-only flag for fronting and backing-device could get out of sync may not have been caught until well after the kernel had shipped to users. The process of developing a test highlighted the hidden requirement, exposed a deficiency in the architecture, and triggered significant reworking of the code, all in advance of the initial merge of the libnvdimm subsystem into the v4.2-rc1 kernel. 

#### Making a mockery

The unit-test infrastructure for libnvdimm lives in the `tools/testing/nvdimm` directory. It compiles modified versions of the nfit (the "firmware interface table" that enables discovery of persistent memory arrays), libnvdimm core, and nvdimm storage drivers that consume mocked resources for exercising the implementation. 

The [general method for inserting mock objects](<https://lwn.net/Articles/558106/>) into a C project is via the `--wrap` option to the linker. In the kernel this is really only suitable for exported symbols. See `tools/testing/nvdimm/Kbuild` for an example of providing fake resources to the ACPI NFIT driver and the resulting child devices. It is worth noting that `--wrap` should only ever appear in `Kbuild` configurations for external (out-of-tree) modules — and testing modules should always be considered out-of-tree, even when they are a part of the kernel source. Beyond giving a visual representation (a "taint" flag) in a kernel oops report that external modules were loaded into the failing kernel, it also separates unit test implementations that want to mock the same symbol in different ways. 

Lastly, a successful unit test implementation should not render the rest of the kernel inoperable. For example, the libnvdimm test modules provide a `__wrap_ioremap()` symbol that redirects requests to mock memory resources for test purposes. However, if that routine is handed a real libnvdimm resource, `__wrap_ioremap()` detects it and passes it through to the actual `ioremap()` in the base kernel. 

I struggled with the kernel build infrastructure to use the linker-provided method for calling "real" symbols from a wrapper. The linker defines a `__real_ioremap()` by default for this purpose, but that symbol cannot be handed to EXPORT_SYMBOL(). Instead, the Kbuild infrastructure can achieve the same effect as `__real_<symbol>()` by placing the definition of the wrappers in their own directory. See `tools/testing/nvdimm/test/iomap.c` for the implementation of the fallback mechanism. 

#### Conclusion

The addition of the unit tests was a contentious point during the review process — a fact that is reflected in [the changelog](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/tools/testing/nvdimm?id=6bc756193ff6>). Although there were numerous, pleasantly surprising occasions of early detection of bugs and architecture defects, this approach carries risk. Mock objects by definition take shortcuts and bend the normal rules of the objects they replace. This fact presents a unique maintenance burden in that different test implementations may choose to mock the same symbol in different ways — ways that would be surprising to a typical kernel developer. 

In general, code aspects in a project that surprise a typical developer are a tax on review resources. That said, the "tax" is overshadowed by the benefits. The volume and pace of iteration needed to take libnvdimm, without regressing, from its initial posting at v4.1-rc1 to its final merging state at v4.2-rc1 could not have been achieved without this capability. As the kernel continues to scale and attract new developers, the need for automated code review and testing will continue to grow.
