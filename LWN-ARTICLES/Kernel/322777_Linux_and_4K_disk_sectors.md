---
title: Linux and 4K disk sectors
url: https://lwn.net/Articles/322777/
date: "March 11, 2009"
category: "Block layer-Large physical sectors; Device drivers-Block drivers"
author: "March 11, 2009 This article was contributed by Goldwyn Rodrigues"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

March 11, 2009

This article was contributed by Goldwyn Rodrigues

As storage devices become bigger and bigger in capacity, the areal density (number of bits packed per physical square inch) increases; hard drives are now hitting the limits. Hard drive manufacturers are now pushing to increase the basic unit of data transfer in hard drives - physical sector size - from 512 bytes to 4096 bytes (or 4KB) to improve storage efficiency and performance. However, there are a lot of subsystems affected by this change that are currently not ready to accept a 4K sector size. 

The first hard drive, the RAMAC, was shipped on September 13, 1956. It weighed 2,140 pounds and held a total of 5 megabytes (MB) of data on fifty 24-inch platters. It was available for lease for $35,000 USD, the equivalent of approximately $300,000 in today's dollars. 

We have come a long way since then. Hard drive capacities are now measured in terabytes, but some legacy parameters, such as the sector size, have remained unchanged. The sector size is wired into a lot of data structures in the kernel, for example, the i_blocks field of `struct inode` stores the number of 512-byte physical blocks it occupies on the media. Even though the core kernel deals with 512-byte sectors, the block layer is capable of handling hardware with different length sector sizes. 

### Why the Change?

Any sort of data communication must contend with noise. This noise is also present during the data transfer from the magnetic surface of the physical hard drive platter to the head of the hard drive. Noise can be introduced by physical defects on the hard drive platter. Noise such as this is measured with respect to the signal strength, more commonly known as Signal to Noise Ratio (SNR). As disk drive areal density increases, the signal to noise ratio decreases, thereby creating increased sensitivity to defects. 

Hard Disk Drives have special reserved bits in addition to the packed data, called the Error-Correcting Code (ECC) bits. Each physical data byte sector block is followed by, besides other bytes, the ECC bytes on the physical medium. ECC is responsible for the reliability of the data transferred. Usually the [Reed-Solomon Algorithm](<http://en.wikipedia.org/wiki/Reed%E2%80%93Solomon_error_correction>) is used to compute the ECC bits; to detect and to a certain extent, correct the errors read; it is an efficient algorithm to correct errors which come in bursts. The ECC bits are placed immediately after the data bytes (as shown in the diagram below), so the error, if any, can be corrected as the disk spins. Besides the ECC, the disk also has bits reserved before the data bits, for the preamble, data sync mark; and the Inter Sector Gap (ISG) after the ECC bits. 

> ![\[On-disk sector structure\]](https://static.lwn.net/images/ns/kernel/4ksec-sector.png)

With the increase in areal density, more bits are packed in a square inch of physical surface. A physical defect of, say 100 nanometers, would require more ECC bits to correct than is needed at lower densities. The physical defect induces more noise than signal hence the SNR decreases. This requires more bytes packed in ECC fields of the sector to compensate for the decrease in SNR and ensure the reliability of the data stored on the disk. For example: on disks with a density of 215 kbpi (kilo bytes per square inch), a 512-byte data sector requires 24 bytes of ECC; a format efficiency (number of user data bytes vs total number of bytes on disk) of 92%. With an increase of areal density to 750 kbpi, each 512-byte sector requires 40 bytes per sector to achieve the same level of disk reliability. The format efficiency of such a drive is 87%. 

A sector size of 4096 bytes requires 100 bytes for ECC to maintain the same level of reliability at an areal density of 750kbpi; that yields a format efficiency of 96%. As areal densities in disk drives continue to increase, the physical size of each sector on the surface of the disk become smaller. If the mean size and number of disk defects and scratches does not scale at the same rate, then we expect more sectors to be corrupted, and we expect the resulting burst errors to more easily exceed the error correction capability of each sector. Having larger sectors, would enable such burst errors to be detected for larger sectors, hence decreasing the total ECC overhead. Besides the ECC, the disk also has bits reserved before the data bits, for the preamble, data sync mark, and the Inter Sector Gap (ISG). Increasing the sector size to 4K from 512 bytes, would decrease the occurrences of these fields, thus improving the format efficiency further. 

For all of these reasons, the storage industry wants to move to larger sector sizes. The IDEMA [International Disk Drive Equipment and Materials Association (IDEMA)](<http://idema.org>) was formed to increase co-operation among competing hard drive brands. IDEMA is responsible for the smooth transition of sector size from 512 bytes to 4Kbytes. Also, [bigsector.org](<http://bigsector.org>) was set up to maintain documentation of the transition. [The documentation section of bigsector.org](<http://bigsector.org/documents.php>) contains more information about the transition. 

### Transition

This change affects a lot of areas in the storage system chain: from the drive interface, the host interface, BIOS, OS to applications such as partition managers. A change affecting so many subsystems might not be readily acceptable to the market. To make a smooth transition, the following stages are planned: 

  1. 512 byte logical with 512 byte physical. This is the current state of hard drives 

  2. 512-byte logical with 4096-byte physical sector size. This would facilitate a smooth transition from 512-byte to 4096-byte sector sizes. 

  3. 4096-byte logical with 4096-byte physical sectors. This would be done once all hardware and software would be aware of the underlying change and geometry with respect to sector size. This change would first be seen in SCSI devices and later in ATA devices. 

During the transition phase (step 2), drives are planned to use 512 byte emulation, known as read-modify write (RMW). Read-modify-write is a technique used to emulate 512-byte sector size over a 4K physical sector size. Written data which does not correspond to full 4K sectors would result in the drive first reading the existing 4K sector, modifying the part of data which changed, and writing the 4K sector data back to the drive. More information on RMW and its implementation can be found [in this set of slides](<http://bigsector.org/WD-Seagate-Maxtor.pdf>). Needless to say, RMW decreases the throughput of the device, though the shorter ECC will compensate by giving an overall better performance (hopefully). Such drives are expected to be commercially available in the first quarter of 2011. 

Matthew Wilcox recently posted a [patch](<http://lwn.net/Articles/320732/>) to support 4K sectors according to the [ATA-8 standard (PDF)](<http://hddguru.com/download/documentation/ATA-ATAPI-standard-8/d1699r2b-ATA8-Command-Set.pdf>). The patch adds an interface function by the name `sector_size_supported()`. Individual drivers are required to implement this function and return the sector size used by the hardware. The size returned is stored in the `sect_size` field of the `ata_device` structure. This function returns 512 if the device does not recognize the ATA-8 command, or the driver does not implement the interface. The `sect_size` is used instead of `ATA_SECT_SIZE` when the data transfer is a multiple of 512-byte sectors. 

The partitioning system and the bootloader will also require changes because they rely on the fact that partitions start from the 63rd sector of the drive, which is misaligned with the 4K sector boundary. This problem will be solved, in the short term, by using the 4K physical - 512 byte logical drives. The 512-byte sectors are aligned in a way that the 1st logical sector starts from the 1st octant of the physical 1st 4K sector, as shown below. 

> ![\[Odd-aligned sector layout\]](https://static.lwn.net/images/ns/kernel/4ksec-odd.png)

This scheme to coincide the logical and physical sectors to optimize data storage and transfer is known as odd-aligned physical/logical sectors. It can lead to other problems though: odd-aligned sectors might misalign the data with respect to filesystem blocks. Assuming a 4K page size, a random read would require two 4K sector reads. This is the reason, applications such as bootloaders and partitioning systems should be ready for 4K sector size hard drives (step 3), for overall throughput efficiency. 

An increased sector size is required by hard drives to break the current limits of hard drive capacity while minimizing the overhead of error checking data. However, a smooth transition will decide the acceptability of these drives in the market. The previous transition, which broke the 8.4GB limit using Large Block Access (LBA), was easily accepted. However, with so many drives in use currently, the transition would be determined by the co-operation of various subsystems of the data supply chain, such as filesystems and applications dealing with hard drives.
