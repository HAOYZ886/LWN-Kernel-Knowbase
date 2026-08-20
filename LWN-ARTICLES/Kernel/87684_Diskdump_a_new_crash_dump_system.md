---
title: "Diskdump: a new crash dump system"
url: https://lwn.net/Articles/87684/
date: "June 2, 2004"
category: Crash dumps; Debugging
author: 
---

A standard feature of most commercial operating systems is a "crash dump" facility. If something goes wrong in the operating system kernel, the system saves its entire state to a file and reboots; the contents of that file can then be examined at leisure to try to figure out what went wrong. The Linux kernel, however, lacks this capability. There are a few possible reasons for this omission: the kernel never crashes (not quite true, unfortunately), kernel developers rarely want crash dumps for their own work, and there is a certain degree of unhappiness with all of the crash dump patches currently in circulation. The fact of the matter, however, is that a number of Linux vendors would like to have a good crash dump system in place so they can better support their customers. 

A [recent patch](<https://lwn.net/Articles/87039/>) posted by Takao Indoh may provide that capability. The new "diskdump" system has taken a simpler approach to crash dumps that, with some fixes, may just get enough core hacker support to be considered for merging into the (presumably 2.7) mainline. 

Diskdump works by taking absolute control of the system when a panic occurs. It shuts down all interrupts to keep the processor from getting distracted; it also freezes all other processors on SMP systems. It then checksums its own code, comparing against a value computed at initialization time; if the checksums fail to match, diskdump assumes that it has been corrupted as a result of whatever went wrong and refuses to run. 

The next step involves finding a place to store the crash dump. Diskdump can be set up with multiple dump partitions. For each possibility, it queries the state of the driver, then reads and verifies the entire crash dump space. The diskdump authors are (rightly) fearful of overwriting important data while the system is in an unstable state, so diskdump requires that every block of the crash dump partition be initialized with a special pattern. If any blocks fail the test, that destination will not be used. 

When a suitable location has been found, diskdump writes a header with the system state and panic information, followed by a memory image. At that point the system can be rebooted; once things are stable again, the "savecore" utility turns the memory image into a proper core dump and reinitializes the crash dump partition. All is then in readiness for debugging and, if need be, the next crash. 

Diskdump needs some significant block driver modifications to be able to do its job. The driver must export a new set of operations: 

```
struct disk_dump_device_ops {
            int (*sanity_check)(struct disk_dump_device *);
            int (*quiesce)(struct disk_dump_device *);
            int (*shutdown)(struct disk_dump_device *);
            int (*rw_block)(struct disk_dump_partition *, int rw, unsigned long
                block_nr, void *buf);
        };
```

The `sanity_check()` call checks to ensure that the device in question is ready to accept a crash dump. If that function finds that, for example, the device is offline or somebody, somewhere is holding a spinlock for the device, the sanity check will fail and the dump will have to go somewhere else. A call to `quiesce()` follows, in case any preparation is needed. The current implementation (which only works with some SCSI devices) performs a full SCSI bus reset at this point. The actual I/O is done via `rw_block`, which is expected to transfer one page per call. This I/O should be done without interrupts (which are, remember, disabled when the panic happens), so the typical implementation will work by polling the device. At the end, `shutdown()` is called to ensure that all blocks have been flushed to the media. 

Perhaps the ugliest part of the patch - and the part which some developers have complained about - is the rerouting of timer and tasklet calls. Since all interrupts are disabled, the normal timer and software interrupt mechanisms will not function. Diskdump does not need those capabilities itself, but a number of disk drivers do. As a result, diskdump must, somehow, run tasklets and timers expected by the driver, but without running arbitrary code unrelated to the dump process. To this end, diskdump sets up its own private timer and tasklet lists which come into action once the system is locked down and the dump process begins. 

Currently, all this works by modifying the drivers to call diskdump's functions rather than the core kernel variants. So, for example, instead of setting up a timer with `add_timer()`, a driver implementing dumps would call this little wrapper: 

```
static inline void diskdump_add_timer(struct timer_list *timer)
        {
            if (crashdump_mode())
                _diskdump_add_timer(timer);
            else
                add_timer(timer);
        }
```

But that function is only available if crash dumps are configured into the system, so some preprocessor macros are used to redefine `add_timer()` if need be. This solution is not going to make it into the mainline kernel, however. The preferred approach would appear to be integrating this functionality directly into the core timer and tasklet routines; that change will make the driver changes smaller, but at the cost of intruding into some of the core kernel code.
