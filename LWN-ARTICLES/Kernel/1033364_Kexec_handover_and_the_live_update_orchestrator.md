---
title: Kexec handover and the live update orchestrator
url: https://lwn.net/Articles/1033364/
date: "August 18, 2025"
category: Kexec handover; Virtualization
author: "By Jonathan Corbet August 18, 2025"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
August 18, 2025

Rebooting a computer ordinarily brings an abrupt end to any state built up by the old system; the new kernel starts from scratch. There are, however, people who would like to be able to reboot their systems without disrupting the workloads running therein. Various developers are currently partway through the project of adding this capability, in the form of "kexec handover" and the "live update orchestrator", to the kernel. 

Normally, rebooting a computer is done out of the desire to start fresh, but sometimes the real objective is to refresh only some layers of the system. Consider a large machine running deep within some cloud provider's data center. A serious security or performance issue may bring about a need to update the kernel on that machine, but the kernel is not the only thing running there. The user-space layers are busily generating LLM hallucinations and deep-fake videos, and the owner of the machine would much rather avoid interrupting that flow of valuable content. If the kernel could be rebooted without disturbing the workload, there would be great rejoicing. 

Preserving a workload across a reboot requires somehow saving all of its state, from user-space memory to device-level information within the kernel. Simply identifying all of that state can be a challenge, preserving it even more so, as a look at the long effort behind the [Checkpoint/Restore in Userspace project](<https://criu.org/Main_Page>) will make clear. All of that state must then be properly restored after the kernel is swapped out from underneath the workload. All told, it is a daunting challenge. 

The problem becomes a little easier, though, in the case of a system running virtualized guests. The state of the guests themselves is well encapsulated within the virtual machines, and there is relatively little hardware state to preserve. So it is not surprising that this is the type of workload that is being targeted for the planned kernel-switcheroo functionality. 

#### Preserving state across a reboot

The first piece of the solution, kexec handover (KHO), was [posted](<https://lwn.net/Articles/1008321/>) by Mike Rapoport earlier this year and merged for the 6.16 kernel release. Rapoport [discussed this work](<https://lwn.net/Articles/1015997/>) at the 2025 Linux Storage, Filesystem, Memory Management, and BPF Summit. KHO offers [a deceptively simple API](<https://docs.kernel.org/core-api/kho/concepts.html>) to any subsystem that needs to save data across a reboot; for example, a call to [`kho_preserve_folio()`](<https://elixir.bootlin.com/linux/v6.16/source/kernel/kexec_handover.c#L648>) will save the contents of a folio. After the new kernel boots, that folio can be restored with [`kho_restore_folio()`](<https://elixir.bootlin.com/linux/v6.16/source/kernel/kexec_handover.c#L184>). A subsystem can use these primitives to ensure that the data it needs will survive a reboot and be available to the new kernel. 

Underneath the hood, KHO prepares the memory for preservation by coalescing it into specific regions. A data structure describing all of the preserved memory is created as a type of [flattened devicetree](<https://devicetree-specification.readthedocs.io/en/stable/flattened-format.html>) that is passed through to the new kernel. Also described in that devicetree are the "scratch areas" of memory — the portions of memory that do _not_ contain preserved data and which, consequently, are available for the new kernel to use during the initialization process. Once the bootstrap is complete and kernel subsystems have reclaimed the memory that was preserved, the system operates as usual, with the workload not even noticing that the foundation of the system was changed out from underneath it. 

Every subsystem that will participate in KHO must necessarily be supplemented with the code that identifies the state to preserve and manages the transition. For the virtualization use case, much of that work can be done inside KVM, which contains most of the information about the virtual machines that are running. With support added to a few device drivers, it should be possible to save (and restore) everything that is needed. What is missing in current kernels, though, is the overall logic that tells each subsystem _when_ it should prepare for the change and when to recover. 

#### The live update orchestrator

The [live update orchestrator (LUO) patches](<https://lwn.net/ml/all/20250723144649.1696299-1-pasha.tatashin@soleen.com>) are the work of Pasha Tatashin; the series is currently in its second version. LUO is the control layer that makes the whole live-update process work as expected. To that end, it handles transitions between four defined system states: 

  * **Normal** : the ordinary operating state of the system. 
  * **Prepared** : once the decision has been made to perform a reboot, all LHO-aware subsystems are informed of a `LIVEUPDATE_PREPARE` event by way of a callback (described below), instructing them to serialize and preserve their state for a reboot. If this preparation is successful across the system, it will enter the prepared state, ready for the final acts of the outgoing kernel. The workload is still running at this time, so subsystems have to be prepared for their preserved state to change. 
  * **Frozen** : brought about by a `LIVEUPDATE_FREEZE` event just prior to the reboot. At this point, the workload is suspended, and subsystems should finalize the data to be preserved. 
  * **Updated** : the new kernel is booted and running; a `LIVEUPDATE_FINISH` event will be sent, instructing each subsystem to restore its preserved state and return to normal operation. 

To handle these events, every subsystem that will participate in the live-update process must create a set of callbacks to implement the transition between system states: 

```
struct liveupdate_subsystem_ops {
    	int (*prepare)(void *arg, u64 *data);  /* normal → prepared */
    	int (*freeze)(void *arg, u64 *data);   /* prepared → frozen */
    	void (*cancel)(void *arg, u64 data);   /* back to normal w/o reboot */
    	void (*finish)(void *arg, u64 data);   /* updated → normal */
        };
```

Those callbacks are then registered with the LUO core: 

```
struct liveupdate_subsystem {
    	const struct liveupdate_subsystem_ops *ops;
    	const char *name;
    	void *arg;
    	struct list_head list;
    	u64 private_data;
        };
    
        int liveupdate_register_subsystem(struct liveupdate_subsystem *subsys);
```

The `arg` value in this structure reappears as the `arg` parameter to each of the registered callbacks (though this behavior [seems likely to change](<https://lwn.net/ml/all/CA+CK2bBEX6C6v63DrK-Fx2sE7fvLTZM=HX0y_j4aVDYcfrCXOg@mail.gmail.com/>) in future versions of the series). The `prepare()` callback can store a data handle in the space pointed to by `data`; that handle will then be passed to the other callbacks. Each callback returns the usual "zero or negative error code" value indicating whether it was successful. 

There is a separate in-kernel infrastructure for the preservation of file descriptors across a reboot; the set of callbacks (defined in [this patch](<https://lwn.net/ml/all/20250723144649.1696299-15-pasha.tatashin@soleen.com>)) looks similar to those above with a couple of additions. For example, the `can_preserve()` callback returns an indication of whether a given file can be preserved at all. Support will need to be added to every filesystem that will host files that may be preserved across a reboot. 

LUO provides an interface to user space, both to control the update process and to enable the preservation of data across an update. For the control side, there is a new device file (`/dev/liveupdate`) supporting a set of [`ioctl()`](<https://man7.org/linux/man-pages/man2/ioctl.2.html>) operations to initiate state transitions; the `LIVEUPDATE_IOCTL_PREPARE` command, for example, will attempt to move the system into the "prepared" state. The current state can be queried at any time, and the whole process aborted before the reboot if need be. The patch series includes a program called [`luoctl`](<https://lwn.net/ml/all/20250723144649.1696299-32-pasha.tatashin@soleen.com>) that can be used to initiate transitions from the command line. 

The preservation of specific files across a reboot can be requested with the `LIVEUPDATE_IOCTL_FD_PRESERVE` `ioctl()` command. The most common anticipated use of this functionality would appear to preserve the contents of [memfd](<https://man7.org/linux/man-pages/man2/memfd_create.2.html>) files, which are often used to provide the backing memory for virtual machines. There is a [separate document](<https://lwn.net/ml/all/20250723144649.1696299-30-pasha.tatashin@soleen.com>) describing how memfd preservation works that gives some insights into the limitations of file preservation. For example, the close-on-exec and sealed status of a memfd will not be preserved, but its contents will. In the prepared phase, reading from and writing to the memfd are still supported, but it is not possible to grow or shrink the memfd. So reboot-aware code probably needs to be prepared for certain operations to be unavailable during the (presumably short) prepared phase. 

This series has received a number of review comments and seems likely to go through a number of changes before it is deemed ready for inclusion. There does not, however, seem to be any opposition to the objective or core design of this work. Once the details are taken care of, LUO seems likely to join KHO in the kernel and make kernel updates easier for certain classes of Linux users.
