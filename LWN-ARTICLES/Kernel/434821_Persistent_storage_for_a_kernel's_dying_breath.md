---
title: "Persistent storage for a kernel's \"dying breath\""
url: https://lwn.net/Articles/434821/
date: "March 23, 2011"
category: Crash dumps; Debugging; Pstore
author: "By Jake Edge March 23, 2011"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jake Edge**  
March 23, 2011

When Linux systems crash, there are various ways to find out what went wrong, but generally those rely on writing to log files on disk. For some systems, disk may not be available, or trusted in the case of a crash, so a way to poke some data into a platform-specific place for use by a subsequent kernel boot would be useful. That's exactly what the [pstore filesystem](<http://permalink.gmane.org/gmane.linux.kernel.commits.head/289849>), which was just added during the current 2.6.39 merge window, will provide. 

The [idea for pstore](<http://article.gmane.org/gmane.linux.kernel.cross-arch/8273/>) came out of a conversation between Tony Luck and Thomas Gleixner at last year's Linux Plumbers Conference. Luck wanted to use the ACPI error record serialization table (ERST) to store crash information across a reboot. The ERST is a mechanism specified by the [ACPI specification [PDF]](<http://www.acpi.info/DOWNLOADS/ACPIspec40a.pdf>) (in section 17.4, page 519) that allows saving and retrieving hardware error information to and from a non-volatile location (like flash). 

Rather than just doing something specific for the x86 architecture, he decided to create a more general framework so that other platforms could use whatever persistent storage they had available. It would be, as Luck put, ""a generic layer for persistent storage usable to pass tens or hundreds of kilobytes of data from the dying breath of a crashing kernel to its successor"". 

There have been a number of iterations of the code since Luck first posted it for comments back in November. After Alan Cox's [suggestion](<http://article.gmane.org/gmane.linux.kernel.cross-arch/8280>), pstore moved from its original firmware driver with a sysfs interface to a more straightforward filesystem-based implementation. 

The basic idea is that a platform can register the availability of a persistent storage location with a call to `pstore_register()` and pass a pointer to a `struct pstore_info`, which looks like: 

```
struct pstore_info {
    	    struct module   *owner;
    	    char            *name;
    	    struct mutex    buf_mutex;      /* serialize access to 'buf' */
    	    char            *buf;
    	    size_t          bufsize;
    	    size_t          (*read)(u64 *id, enum pstore_type_id *type,
    			    struct timespec *time);
    	    u64             (*write)(enum pstore_type_id type, size_t size);
    	    int             (*erase)(u64 id);
        };
```

The platform driver needs to provide three I/O routines and a buffer. There is also a mutex present to protect against simultaneous access to the buffer. With that, pstore will implement a filesystem that can be accessed from the kernel--or from user space once it has been mounted. The underlying ERST storage is record oriented, and Luck posits that other platform storage areas will be also, so the I/O interface is record oriented as well. 

In addition to the pstore framework, the ERST driver was [modified to take advantage of pstore](<http://article.gmane.org/gmane.linux.kernel.commits.head/289854/match=pstore>); that change was also merged, so there is an in-kernel user of pstore. The `pstore_info` buffer is allocated and managed by `drivers/acpi/apei/erst.c`, and is larger than the `bufsize` advertised to account for the record and section headers required by ERST. Users of the IO interface either fill the buffer before calling `pstore_info.write()` or read the data from the buffer after a call to `pstore_info.read()`. 

Each item is stored with a type, either `PSTORE_TYPE_DMESG` for log messages (likely oops output), `PSTORE_TYPE_MCE` for hardware errors, or `PSTORE_TYPE_UNKNOWN` for other undefined types. When stored, each item gets a record ID associated with it, which gets returned from the `pstore_info.write()` call. That ID can then be used in `read()` and `erase()` operations, but it also appears in the filenames in the pstore filesystem. 

The filesystem can be mounted using: 

```
# mount -t pstore - /dev/pstore
```

Files will appear there with names based on the type, name of the storage driver, and the id, so the first dmesg record for ERST would be `/dev/pstore/dmesg-erst-1`. The typical scenario would be for the filesystem to be mounted at boot time, then some user-space process would check for any files there, copy them to some more permanent place, and delete the files with `rm`. That will allow the storage facility driver to reclaim the space in order to be ready for other crashes or hardware errors. 

By default, pstore will register a dump handler with kmsg_dump to write the last 10K bytes of data from the kernel log to the pstore device when there is a kernel oops or panic. The amount of data to store can be configured at mount time using the `kmsg_bytes` parameter. 

Luck has also put out an [RFC patch](<https://lwn.net/Articles/434918/>) to disable dumping information into pstore for some kinds of kmsg_dump reasons (e.g. `KMSG_DUMP_HALT` or `KMSG_DUMP_RESTART`), but various other developers weren't so sure. Seiji Aguchi pointed to two use cases ([1](<https://lkml.org/lkml/2010/11/17/249>), [2](<https://lkml.org/lkml/2011/2/23/325>)) he has found for needing to store the tail of the kernel log messages in most of those cases. In addition, Artem Bityutskiy [pointed out](<https://lwn.net/Articles/434920/>) that having pstore decide which kmsg_dump reasons to handle ""smells like policy in the kernel"". Adding more options to control that behavior is certainly possible, but Luck seems to be of a mind to [wait a bit](<https://lwn.net/Articles/434921/>) before making any change. 

There are other persistent storage methods for kernel log messages, notably `devices/mtd/mtdoops.c` and `devices/char/ramoops.c`. But those are targeted at the embedded space where NVRAM devices are prevalent or for platforms where RAM can be reserved that will not be cleared on a restart. Pstore is more flexible, as it can store more than just kernel logs, while the two `*oops` devices are wired into storing the output of kmsg_dump. 

Now that pstore has been merged, others will likely start using it. David Miller has already [indicated](<http://article.gmane.org/gmane.linux.kernel.cross-arch/8275/>) that he will use it for sparc64, where a region of memory can be set aside to persist across reboots. One would guess that other architectures that have hardware support for similar mechanisms will as well.
