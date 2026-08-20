---
title: Xen is coming
url: https://lwn.net/Articles/112391/
date: "November 22, 2004"
category: "Virtualization-Xen; Xen"
author: 
---

[Xen](<http://www.cl.cam.ac.uk/Research/SRG/netos/xen/index.html>) is a free virtualization system designed to allow multiple virtual machines to be run on a single host system with high performance. The Xen system (version 2.0 was [released](<http://lwn.net/Articles/109789/>) recently) offers a number of interesting features, including flexible networking between virtual machines and the ability to transparently move virtual machines between physical hosts while they are running. Xen's authors claim that the performance hit from running under Xen is only "a few percent." 

Now that the 2.0 release is out, the Xen developers would like to merge their code into the mainline kernel. The bulk of this code adds the new Xen "architecture," which enables the kernel to run on the virtual machine provided by Xen itself. The architecture code is [available from the Xen site](<http://www.cl.cam.ac.uk/netos/xen/downloads/arch-xen.patch>) for those who are interested. Another significant chunk is [a set of drivers](<http://www.cl.cam.ac.uk/netos/xen/downloads/drivers-xen.patch>) which provide Xen-hosted systems with network interfaces, file-backed block devices, and console devices. 

Inclusion of both of those patch sets should be relatively uncontroversial; they do not affect any code which is not actually built for the Xen architecture, and thus should not risk breaking anything. The final set, however, will have to be looked at more closely; these are the patches to the core kernel itself. Most of these patches make the kernel work with Xen's very different way of managing and allocating memory; they include a new `sk_buff` structure allocation function, a change to how `/dev/mem` works on the Xen architecture, and a new `ptep_establish_new()` function which optimizes the instantiation of new pages. Perhaps the most controversial change is a change in how the architecture-specific `arch_free_page()` function works: under Xen, this function might actually short out the rest of the page allocator functions and dispose of the page itself. This technique allows Xen to manage a single page pool for multiple virtual machines, but not everybody liked changing the interface to `arch_free_page()` in that way. 

That said, there appears to be no strong opposition to the inclusion of these patches. It would not be surprising to see them go into -mm sometime after 2.6.10 comes out.
