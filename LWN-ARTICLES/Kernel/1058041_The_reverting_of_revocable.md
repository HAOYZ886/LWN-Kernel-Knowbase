---
title: The reverting of revocable
url: https://lwn.net/Articles/1058041/
date: "February 12, 2026"
category: "Device drivers-Support APIs; Race conditions"
author: "By Jonathan Corbet February 12, 2026"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
February 12, 2026

Transient devices pose a special challenge for an operating-system kernel. They can disappear at any time, leaving behind kernel data structures that no longer refer to an existing device, but which may still be in use by unknown kernel code. Managing the resulting lifecycle issues has frustrated kernel developers for years. In September 2025, the [revocable resource-management patch series](<https://lwn.net/Articles/1038523/>) from Tzung-Bi Shih appeared to offer a partial solution to this problem. Since then, though, other problems have arisen, and the planned merging of this series into the 7.0 release has been called off. 

The core idea behind this series is the careful management of references to data structures associated with transient devices. Kernel code needing access to one of those structures would attempt to obtain a short-lived reference; the attempt will succeed if the device is still present and functioning normally. That reference is protected by [sleepable read-copy-update (SRCU)](<https://lwn.net/Articles/202847/>), ensuring that the data structure in question will not disappear until after the next SRCU grace period. 

If a device disappears from the system, the relevant driver will mark it as "gone" and deny any subsequent requests for references to its data structures. After an SRCU grace period has passed, the owner of the data structure, secure in the knowledge that no references to it can still exist, can safely free that structure. The uncertainty around the data's lifecycle has been replaced with a clear indication of when it is no longer in use. 

Greg Kroah-Hartman [welcomed](<https://lwn.net/ml/all/2025091224-blaming-untapped-6883@gregkh/>) this series when it was posted; he took it into the driver-core repository with the intent of pushing it upstream during the 7.0 merge window. On January 24, though, Johan Hovold [requested a revert](<https://lwn.net/ml/all/20260124170535.11756-1-johan@kernel.org>), complaining that the series should never have been applied. Normally this sort of infrastructure is not accepted without code that actually uses it, but that practice was not followed in this case; there was no in-tree user of the revocable-access functionality. Hovold criticized that move, saying that the proposed use cases for this feature do not actually need it, and that the code itself had some serious race-condition bugs. The revocable code, he said, should be taken back out ""until a redesign has been proposed and evaluated properly"". 

For his part, Kroah-Hartman [resisted](<https://lwn.net/ml/all/2026020316-regain-posing-df8e@gregkh>) the idea of reverting this change: 

> Ah, but I do think this is the way forward, given that the pattern/idea works in the rust side of the kernel, and it's exactly what I've been asking for for years now :) 
> 
> But yes, without a real user, it's hard for me to justify it. But, I want it present in the tree now so that lots of others can play with it easily. If it turns out it is not correct, and does not work properly, then great, we will delete the files entirely. But I'm not so sure that we are there yet. 

He was referring to the [`Revocable` trait](<https://rust.docs.kernel.org/kernel/revocable/struct.Revocable.html>) used by Rust code in the kernel. It provides an abstraction to provide access (at no run-time cost) to a data structure that is guaranteed by its owner to not abruptly disappear. For cases where that guarantee cannot be made, there is a `try_access()` function that works in a manner similar to the proposed C functionality. For the curious, Danilo Krummrich [described the Rust implementation](<https://lwn.net/ml/all/DFXV1QMQMTRK.2W8SWGVHS2K69@kernel.org>) in some detail. He pointed out that a C implementation cannot work in the same way ""due to language limitations"", but thought that the revocable series was a worthwhile exercise in figuring out how best to adapt the Rust pattern to the C side. 

Jason Gunthorpe, though, [described](<https://lwn.net/ml/all/20260126000730.GI1134360@nvidia.com>) that mechanism — and any interface that allows access to a device after it has been unregistered — as ""*dangerous*"", and said that use of the `try_access()` functions should be treated as ""a code smell that says something is questionable in the driver or subsystem"". The real value in the Rust abstraction, he said, is how it forces documentation of which contexts can safely access a device structure, and which are uncertain. The C version, instead, forces all accesses to be treated as uncertain, losing the documentation value, hurting performance, and possibly encouraging other types of bugs. 

Hovold [described](<https://lwn.net/ml/all/aXdxDBXdyqLFfKKI@hovoldconsulting.com>) `Revocable` as ""a design pattern that's perhaps needed for rust, but not necessarily elsewhere"". Gunthorpe [said more strongly](<https://lwn.net/ml/all/20260127235232.GS1134360@nvidia.com>) that adding something like the Rust abstraction is ""not something we want to do"". Instead, he said, changes should be made so that driver operations (often called "fops" since they are gathered together in the [`file_operations` structure](<https://elixir.bootlin.com/linux/v6.18.6/source/include/linux/fs.h#L2268>)) should simply be run in a safe context where resources cannot disappear from underneath them. Laurent Pinchart [agreed](<https://lwn.net/ml/all/20260129010822.GA3310904@killaraus>), and outlined a possible solution around safer `file_operations` invocations. 

Meanwhile, Shih, who was unsurprisingly against reverting the series, [said](<https://lwn.net/ml/all/aXjgeNY-jf9rIw09@google.com>) that keeping it in linux-next, at least, would be helpful. He posted [a separate series](<https://lwn.net/ml/all/20260129143733.45618-1-tzungbi@kernel.org>) fixing the race conditions reported by Hovold. Kroah-Hartman quickly picked up the fixes, leading to [another complaint](<https://lwn.net/ml/all/aYHm9pr0e7myeqS3@hovoldconsulting.com>) from Hovold, who asked again for the series to be reverted. In response, Kroah-Hartman [defended](<https://lwn.net/ml/all/2026020315-twins-probe-d988@gregkh>) his acceptance of the fixes, but agreed to disable the revocable feature from the build for the 7.0 release cycle. That did not stop the disagreement, though; Hovold [responded](<https://lwn.net/ml/all/aYNZYqpV62wVtxDI@hovoldconsulting.com>) that ""API design should not be done incrementally in-tree"". 

It took a few more days but, on February 6, Kroah-Hartman [threw in the towel](<https://lwn.net/ml/all/2026020624-buddhism-clavicle-7a90@gregkh>) and applied Hovold's revert patches. ""Kernel developers / maintainers are only 'allowed' one major argument / fight a year, and I really don't want to burn my 2026 usage so early in the year :)"" He asked Shih to go through the feedback and prepare a new series to be reviewed and, with luck, merged for a future kernel release. 

This, of course, is not the sort of outcome anybody is hoping for when they put together an improvement for the kernel (or any other free-software project). But it certainly happens at times. If all goes well from here, this setback will lead, in the long term, to a better and more maintainable solution that will, finally, address a problem that kernel developers have struggled with for years.
