---
title: A journal for MD/RAID5
url: https://lwn.net/Articles/665299/
date: "November 24, 2015"
category: "Block layer-RAID; RAID"
author: "November 24, 2015 This article was contributed by Neil Brown"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

November 24, 2015

This article was contributed by Neil Brown

RAID5 support in the MD driver has been part of mainline Linux since 2.4.0 was released in early 2001. During this time it has been used widely by hobbyists and small installations, but there has been little evidence of any impact on the larger or "enterprise" sites. Anecdotal evidence suggests that such sites are usually happier with so-called "hardware RAID" configurations where a purpose-built computer, whether attached by PCI or fibre channel or similar, is dedicated to managing the array. This situation could begin to change with the 4.4 kernel, which brings some enhancements to the MD driver that should make it more competitive with hardware-RAID controllers. 

While hardware-RAID solutions suffer from the lack of transparency and flexibility that so often come with closed devices, they have two particular advantages. First, a separate computer brings dedicated processing power and I/O-bus capacity which takes some load off the main system, freeing it for other work. At the very least, the system CPU will never have to perform the XOR calculations required to generate the parity block, and the system I/O bus will never have to carry that block from memory to a storage device. As commodity hardware has increased in capability and speed over the years, though, this advantage has been significantly eroded.

The second advantage is non-volatile memory (NVRAM). While traditional commodity hardware has not offered much NVRAM because it would hardly ever be used, dedicated RAID controllers nearly always have NVRAM as it brings real benefits in both performance and reliability. Utilizing NVRAM provides more than just the incremental benefits brought by extra processing components. It allows changes in data management that can yield better performance from existing devices.

With recent developments, non-volatile memory is becoming a reality on commodity hardware, at least on server-class machines, and it is becoming increasing easy to attach a small solid-state storage device (SSD) to any system that manages a RAID array. So the time is ripe for MD/RAID5 to benefit from the ability to manage data in the ways that NVRAM allows. Some engineers from Facebook, particularly Shaohua Li and Song Liu, have been working toward this end; Linux 4.4 will be the first mainline release to see the fruits of that labor. 

#### Linux 4.4 — closing the RAID5 write hole

RAID5 (and related levels such as RAID4 and RAID6) suffer from a potential problem known as the "write hole". Each "stripe" on such an array — meaning a set of related blocks, one stored on each active device — will contain data blocks and parity blocks; these must always be kept consistent. The parity must always be exactly what would be computed from the data. If this is not the case then reconstructing the data that was on a device that has failed will produce incorrect results.

In reality, stripes are often inconsistent, though only for very short intervals of time. As the drives in an array are independent (that is the "I" of RAID) they cannot all be updated atomically. When any change is made to a stripe, this independence will almost certainly result in a moment when data and parity are inconsistent. Naturally the MD driver understands this and would never try to access data during that moment of inconsistency ... unless....

Problems occur if a machine crash or power failure causes an unplanned shutdown. It is fairly easy to argue that the likelihood that an unclean shutdown would interrupt some writes but not others is extremely small. It's not easy to argue that such a circumstance could never happen, though. So when restarting from an unclean shutdown, the MD driver must assume that the failure may have happened during a moment of inconsistency and, thus, the parity blocks cannot be trusted. If the array is still optimal (no failed devices) it will recalculate the parity on any stripe that could have been in the middle of an update. If, however, the array is degraded, the parity cannot be recalculated. If some blocks in a stripe were updated and others weren't, then the block that was on the failed device will be reconstructed based on inconsistent information, leading to data corruption. To handle this case, MD will refuse to assemble the array without the "`--force`" flag, which effectively acknowledges that data might be corrupted.

An obvious way to address this issue is to use the same approach that has worked so well with filesystems: write all updates to a journal before writing them to the main array. When the array is restarted, any data and parity blocks still in the journal are simply written to the array again. This ensures the array will be consistent whether it is degraded or not. This could be done with a journal on a rotating-media drive but the performance would be very poor indeed. The advent of large NVRAM and SSDs makes this a much more credible proposition.

#### The new journal feature

The functionality developed at Facebook does exactly this. It allows a journal device (sometimes referred to as a "cache" or "log" device) to be configured with an MD/RAID5 (or RAID4 or RAID6) array. This can be any block device and could even be a mirrored pair of SSDs (because you wouldn't want the journal device to become a single point of failure).

To try this out you would need Linux 4.4-rc1 or later, and the current mdadm from `git://neil.brown.name/mdadm`. Then you can create a new array with a journal using a command like 

```
mdadm --create /dev/md/test --level=5 --raid-disks=4 --write-journal=/dev/loop9 \
              /dev/loop[0-3]
```

It is not currently possible to add a journal to an existing array, but that functionality is easy enough to add later.

With the journal in place, RAID5 handling will progress much as it normally does, gathering write requests into stripes and calculating the parity blocks. Then, instead of being written to the array, the stripe is intercepted by the journaling subsystem and queued for the journal instead. When write traffic is sufficiently heavy, multiple stripes will be grouped together into a single transaction and written to the journal with a single metadata block listing the addresses of the data and parity. Once this transaction has been written and, if necessary, flushed to stable storage, the core RAID5 engine is told to process the stripe again, and this time the write-out is **not** intercepted.

When the write to the main array completes, the journaling subsystem will be told; it will occasionally update its record of where the journal starts so that data that is safe on the array effectively disappears from the journal. When the array is shut down cleanly, this start-of-journal pointer is set to an empty transaction with nothing following. When the array is started, the journal is inspected and if any transactions are found (with both data and parity) they are written to the array.

The journal metadata block uses 16 bytes per data block and so can describe well over 200 blocks. Along with each block's location and size (currently always 4KB), the journal metadata records a checksum for each data block. This, together with a checksum on the metadata block itself, allows very reliable determination of which blocks were successfully written to the journal and so should be copied to the array on restart.

In general, the journal consists of an arbitrarily large sequence of metadata blocks and associated data and parity blocks. Each metadata block records how much space in the journal is used by the data and parity and so indicates where the next metadata block will be, if it has been written. The address of the first metadata block to be considered on restart is stored in the standard MD/RAID superblock.

The net result of this is that, while writes to the array might be slightly slower (depending on how fast the journal device is), a system crash never results in a full resync — only a short journal recovery — and there is no chance of data corruption due to the write hole.

Given that the write-intent bitmap already allows resynchronization after crash to be fairly quick, and that write-hole corruption is, in practice, very rare; you may wonder if this is all worth the cost. Undoubtedly different people will assess this tradeoff differently; now at least the option is available once that assessment is made. But this is not the full story. The journal can provide benefits beyond closing the write-hole. That was a natural place to start as it is conceptually relatively simple and provides a context for creating the infrastructure for managing a journal. The more interesting step comes next. 

#### The future: writeback caching and more full-stripe writes

While RAID5 or RAID6 provide a reasonably economical way to combine multiple devices to provide large storage capacity with reduced chance of data loss, they do come at a cost. When the host system writes a full stripe worth of data to the array, the parity can be calculated from that data and all writes can be scheduled almost immediately, leading to very good throughput. When writing to less than a full stripe, though, throughput drops dramatically.

In that case, some data or parity blocks need to be read from the array before the new parity can be calculated. This read-before-write introduces significant latency to each request, so throughput suffers. The MD driver tries to delay partial-stripe writes a little bit in the hope that the rest of the stripe might be written soon. When this works, it helps a lot. When it doesn't, it just increases latency further.

It is possible for a filesystem to help to some extent, and to align data with stripes to increase the chance of a full-stripe write, but that is far from a complete solution. A journal can make a real difference here by being managed as a writeback cache. Data can be written to the journal and the application can be told that the data is safe before the RAID5 engine even starts considering whether some pre-reading might be needed to be able to update parity blocks.

This allows the application to see very short latencies no matter what data-block pattern is being written. It also allows the RAID5 core to delay writes even longer, hoping to gather full stripes, without inconveniencing the application. This is something that dedicated RAID controllers have (presumably) been doing for years, and hopefully something that MD will provide in the not-too-distant future.

There are plenty of interesting questions here, such as whether to keep all in-flight data in main memory, or to discard it after writing to the journal and to read it back when it is time to write to the RAID. There is also the question of when to give up waiting for a full stripe and to perform the necessary pre-reading. Together with all this, a great deal of care will be needed to ensure we actually get the performance improvements that theory suggests are possible.

This is just engineering though. There is interest in this from both potential users of the technology and vendors of the NVRAM and there is little doubt that we will see the journal enhanced to provide very visible performance improvements to complement the nearly invisible reliability improvements already achieved.
