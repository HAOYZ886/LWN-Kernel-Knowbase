---
title: "Big block transfers: good or bad?"
url: https://lwn.net/Articles/77981/
date: "March 29, 2004"
category: Block layer; Serial ATA; libata
author: 
---

Users of serial ATA drives on Linux will be familiar with Jeff Garzik's "libata" driver, which provides solid support for those drives with several controllers. Jeff recently posted [a patch](<https://lwn.net/Articles/77826/>) which has the potential to make SATA users happier; with this patch, libata will use the "LBA48" mode, which can perform transfers of up to 32MB in length. Says Jeff: 

With this simple patch, the max request size goes from 128K to 32MB... so you can imagine this will definitely help performance. Throughput goes up. Interrupts go down. Fun for the whole family. 

Interestingly, the whole family was not entirely thrilled by the idea. The problem is latency: most SATA drives will take the better part of a second to perform a 32MB transfer, during which no other requests are being processed. Several people complained, saying that a 32MB limit is far too high, and that, in any case, the performance benefits of transfers above around 1MB are minimal at best. Jeff's [explanation](<https://lwn.net/Articles/77982/>) that, in reality, transfers would be limited to 8MB with the current libata driver did little to slow the debate. 

The issue being debated is not whether 32MB transfers could create latency problems; everybody agrees on that point. The difference of opinion is over where the decision on transfer sizes should be made. A device driver's job, according to Jeff, is to make the full capabilities of the device available to the kernel without imposing arbitrary limits. He would rather see the block layer deal with maximum transfer size issues. Jens Axboe, the maintainer of the block layer, [responds](<https://lwn.net/Articles/77985/>) that the block layer has no idea of the performance characteristics of any individual device, while the driver does. The driver, thus, is in the best position to make decisions about maximum transfer sizes. 

In truth, the driver doesn't know the right number, either; it can depend on individual drives, the controller being used, etc. As a result, the final outcome [looks](<https://lwn.net/Articles/78153/>) like it will involve some sort of adaptive, dynamic tuning. The block layer will track the execution time of requests and note when that time gets to be too long; at that point, it will have the information needed to put a lid on request size. The same timing information could also be used to tweak the maximum tagged command queueing depth (the number of requests which can be fed simultaneously to the drive), since a number of similar issues come up there.
