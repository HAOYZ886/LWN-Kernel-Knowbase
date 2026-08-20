---
title: POHMELFS returns
url: https://lwn.net/Articles/480095/
date: "February 8, 2012"
category: "Filesystems-Network"
author: "By Jonathan Corbet February 8, 2012"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
February 8, 2012

LWN [wrote briefly](<https://lwn.net/Articles/267896/>) about the POHMELFS filesystem in early 2008; thereafter, POHMELFS has languished in the staging tree without much interest or activity. The POHMELFS developer, Evgeniy Polyakov, [expressed his unhappiness with the development process](<https://lwn.net/Articles/308566/>) and disappeared from the kernel community for some time. 

Now, though, [Evgeniy is back](<https://lwn.net/Articles/480093/>) with a new POHMELFS release. He said: 

It went a long way from parallel NFS design which lived in drivers/staging/pohmelfs for years effectively without usage case - that design was dead. 

New pohmelfs uses elliptics network as its storage backend, which was proved as effective distributed system. Elliptics is used in production in Yandex search company for several years now and clusters range from small (like 6 nodes in 3 datacenters to host 15 billions of small files or hundred of nodes to scale to 1 Pb used for streaming). 

This time around, he is asking that the filesystem be merged straight into the mainline without making a stop in the staging tree. But merging a filesystem is hard without reviews from the virtual filesystem maintainers, and no such reviews have yet been done. So Evgeniy may have to wait a little while longer yet.
