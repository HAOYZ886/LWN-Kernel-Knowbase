---
title: InfiniBand arrives
url: https://lwn.net/Articles/112531/
date: "November 23, 2004"
category: InfiniBand
author: 
---

After a long period of development, the [OpenIB Alliance](<http://www.openib.org/>) has posted [an initial set of patches](<https://lwn.net/Articles/112535/>) for review. The current patch set is not proposed for inclusion, though the project has made it clear that merging into a not-too-distant 2.6 kernel is something they would like. The initial comments suggest that there may not be much opposition to that. 

The patch set is large, reflecting the complexity of the InfiniBand specification. At the bottom layer, a driver for Mellanox adapters is included with the patch set; it's some 9,000 lines of sparsely-commented code. The core "midlayer" manages InfiniBand ports and makes access to the fabric available for the upper layers. The midlayer also allows for user-space administration by facilitating the passing of "MADs" ("management datagrams") back and forth. 

The upper layers of the InfiniBand specification envision support for a number of features, including MPI (message passing interface, heavily used in clustered applications), SDP (socket direct protocol: a networking standard based on remote DMA), SRP (remote SCSI), and IP over InfiniBand using the classic socket interface. The current OpenIB patches concentrate on full IP (both IPv4 and IPv6) support; most of the other high-level protocols are not yet implemented. 

The comments on the InfiniBand code have been relatively minor, so far. The project's choices for device names (deeply nested names like `/dev/infiniband/mthca0/ports/1/mad`) will likely be changed. The project also went with dynamic device number assignment. This technique works well on systems running a tool like `udev` to create the device nodes, but it makes life difficult on systems where device nodes must be created manually by the administrator. For now, at least, plenty of such systems exist, so static device numbers are needed. The OpenIB drivers also rely on `ioctl()` calls for a number of administrative functions; questions were raised, but the current interface is not likely to be changed in any significant way. 

Perhaps the most surprising complaint, to many, was the objection to the dual GPL/BSD license carried by the OpenIB code. BSD-licensed code is not normally a problem in the kernel; it can be included in a larger, GPL-licensed program without any sort of infringement. The OpenIB code uses [read-copy-update (RCU)](<https://lwn.net/Articles/37889/>), however, and that usage brings an additional constraint. IBM holds a patent on RCU, and has licensed that patent for use with GPL-licensed code. As is the case with many of these patent licenses, BSD-licensed code is not covered. So the OpenIB developers may find themselves having to (1) drop the BSD license from their code, (2) stop using RCU, or (3) get some sort of special exemption from IBM. It [appears](<https://lwn.net/Articles/112618/>) that they will choose the second option. 

One issue which has _not_ come up is concern over the licensing of the InfiniBand specification or any patents which may apply to it. The InfiniBand developers seem to have [resolved](<http://lwn.net/Articles/108526/>) those concerns through a combination of easing access to the specification and pointing out that the InfiniBand patent agreement is closely aligned with the agreements which apply to other standards, such as PCI. There may well be patented technologies lurking within the InfiniBand specification, but InfiniBand should not present a higher risk of patent difficulties than any other part of the kernel.
