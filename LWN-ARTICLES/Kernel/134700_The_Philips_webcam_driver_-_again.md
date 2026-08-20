---
title: "The Philips webcam driver - again"
url: https://lwn.net/Articles/134700/
date: "May 4, 2005"
category: Copyright issues; Philips driver
author: 
---

The 2.6.12-rc kernels include, among many other things, the long-awaited return of the Philips web camera driver. This driver, remember, was removed at the original author's request; that author (known as "Nemosoft Unv") objected to the removal of a special-purpose hook which allowed a non-free decompression module to be loaded into the kernel. After the removal, Luc Saillard took over the driver, with the goal of getting it back into the mainline. As part of that process, he reverse engineered the image decompression code and included it in the GPL-licensed module. It would appear that this episode has led to a good result: the Philips driver is back, and more free than before. 

Nemosoft has recently [resurfaced](<https://lwn.net/Articles/134703/>), however, to make the claim that things may not be quite as good as they seem. According to Nemosoft, no real reverse engineering job was done. Instead: 

In case you hadn't noticed, that code has been reverse compiled (I would not even call it "reverse engineered"), and is simply illegal. Maybe not in every country, but certainly in some. There are still some intellectual property rights being violated here, you know, and I'm surprised at the contempt you and Linux kernel maintainers show in this regard for a few lines of the law. 

Mr. Saillard has been silent on how he performed the reverse engineering task. A look at the code ([example - pwc-kiara.c](<https://lwn.net/Articles/134704/>)) is somewhat unenlightening - the decompression code consists mostly of a set of tables filled with mysterious numbers. It is hard to imagine how those tables could be created in any way other than extracting them from the binary decompressor module. 

If the code was truly decompiled and relicensed, there could be a copyright issue here. On the other hand, the tables used for decompression will be hard to protect if they are truly the only way to interpret images produced by the camera. Alan Cox (who forwarded the PWC patches for merging) [acknowledges](<https://lwn.net/Articles/134705/>) that there could be an issue with the decompression code, but he is not overly worried about it: 

The legal position on reverse engineering is in general fairly clear. What you describe might not be. If so then we need to find someone who hasn't read the code to rewrite it from the algorithm description of the current code. Shouldn't take more than a week. 

Alan also points out an issue others have raised: by Nemosoft's admission, the non-disclosure agreement which forced the decompression code to be proprietary ran out some time ago. Nemosoft could thus resolve the licensing issues by simply releasing the decompression code under a free license.
