---
title: "Modular, switchable I/O schedulers"
url: https://lwn.net/Articles/102976/
date: "September 21, 2004"
category: "Block layer-IO scheduling; Elevator; IO scheduler"
author: 
---

The I/O scheduler ("elevator") has a challenging job: it must arrange for disk I/O operations to be executed in the optimal order. "Optimal" means maximizing the I/O bandwidth to the disk while, simultaneously, ensuring that all requests are satisfied in a timely manner, no process suffers excessive latency, and, for desktop systems, that the interactive "feel" of the system is responsive. Some schedulers take on additional tasks, such as dividing the available bandwidth equally between processes (or users) contending for each disk. 

Given that set of demands, it is not surprising that there are multiple I/O schedulers in the Linux kernel. The deadline scheduler works by enforcing a maximum latency for all requests. The anticipatory scheduler briefly stalls I/O after a read request completes with the idea that another, nearby read is likely to come in quickly. The completely fair queueing scheduler (recently [updated](<https://lwn.net/Articles/101029/>) by Jens Axboe) applies a bandwidth allocation policy. And there is a simple "noop" scheduler for devices, such as RAM disks, which do not benefit from fancy scheduling schemes (though such devices usually short out the request queue entirely). 

The kernel has a nice, modular scheme for defining and using I/O schedulers. What it lacks, however, is any flexible way of letting a system administrator _choose_ a scheduler. I/O schedulers are built into the kernel code, and exactly one of them can be selected - for all disks in the system - at boot time with the `elevator=` parameter. There is no way to use different schedulers for different drives, or to change schedulers once the system boots. The chosen scheduler is used, and any others configured into the system simply sit there and consume memory. 

Jens Axboe has recently posted [a patch](<https://lwn.net/Articles/102593/>) which improves on this situation. With this patch in place, I/O schedulers can be built as loadable modules (though, as Jens cautions, at least one scheduler must be linked directly into the kernel or the system will have a hard time booting). A new `scheduler` attribute in each drive's sysfs tree lists the available schedulers, noting which one is active at any given time. Changing schedulers is simply a matter of writing the name of the new scheduler into that attribute. 

The patch is long, but the amount of work required to support switchable I/O schedulers wasn't all that great. The internal structures describing elevators have been split apart to reflect the more dynamic nature of things; `struct elevator_ops` contains the scheduler methods, while `struct elevator_type` holds the metadata which describes an I/O scheduler to the kernel. The new `elevator_queue` structure glues an instance of an I/O scheduler to a specific request queue. Updating the mainline schedulers to work with the new structures required a fair number of relatively straightforward code changes. Each scheduler now also has module initialization and cleanup functions which have been separated from the code needed to set up or destroy an elevator for a specific queue. 

One interesting question is: what should be done with the currently queued block requests when an I/O scheduler change is requested? One could imagine requeueing all of those requests with the new scheduler in order to let it have its say immediately. The simpler approach, which was chosen for this patch, is to block the creation of new requests and wait for the queue to empty out. Once all outstanding I/O has been finished up, the old scheduler can be shut down and moved out of the way. 

There have been no (public) objections to the patch; chances are it will find its way into the mainline sometime after 2.6.9 comes out.
