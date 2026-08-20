---
title: Network acceleration with DPDK
url: https://lwn.net/Articles/725254/
date: "July 5, 2017"
category: Data Plane Development Kit DPDK
author: "July 5, 2017 This article was contributed by Rami Rosen"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

July 5, 2017

This article was contributed by Rami Rosen

Network acceleration has always been a subject that naturally attracts the interest of network device vendors and developers. Kernel network acceleration techniques that require, for example, the caching of kernel networking data structures inside the network driver (or maintaining a private modified kernel for a specific device) are naturally frowned upon and bound to be rejected by the kernel networking community. There are also user-space kernel-bypass solutions, including the [Data Plane Development Kit (DPDK)](<http://dpdk.org/>). 

Among the most popular open-source projects providing user-space network acceleration are [Snabb](<https://lwn.net/Articles/713918/>), [netmap](<http://info.iet.unipi.it/~luigi/netmap/>), and DPDK. With the recent [announcement](<https://www.linuxfoundation.org/announcements/networking-industry-leaders-join-forces-to-expand-new-open-source-community-to-drive>) by Jim Zemlin this April that DPDK project has moved to the Linux Foundation, it seems that this is a good time to get an overview of the current status of this project and its roadmap. 

#### The DPDK project

DPDK was created by Intel in 2010 as a suite of tools that enable the efficient transfer of packets through a server. In 2013, the project web site, www.dpdk.org, was created by 6Wind and, recently, it moved to the Linux Foundation. DPDK is a set of libraries and drivers written in C providing I/O acceleration for network and cryptographic devices. It is a fully open-source (BSD-licensed) project, and it runs on Linux and FreeBSD. The project maintainer is Thomas Monjalon. 

DPDK is used by more than 20 open-source projects, including [OPNFV](<https://www.opnfv.org/>), [OvS-DPDK](<https://software.intel.com/en-us/articles/open-vswitch-with-dpdk-overview>), the Fast Data project ([FD.io](<https://fd.io/>)), [Rump](<https://github.com/rumpkernel/drv-netif-dpdk>), [dpdk-nginx](<https://github.com/ansyun/dpdk-nginx>), [OpenDaylight](<https://www.opendaylight.org/>), [Contrail Virtual Router](<https://github.com/Juniper/contrail-vrouter>), and more. It supports a wide variety of platforms and [over 20 types of interface cards](<http://dpdk.org/doc/nics>); it runs on a variety of CPU architectures. It includes contributions from over 400 individuals from 70 different organizations. Starting April 2016, it adopted the Ubuntu numbering scheme, where each release is tagged as YY.MM; so the last DPDK release is DPDK 17.05, from May 2017, and the next release will be DPDK 17.08, which will be released in August 2017, reflecting the project's quarterly release cadence. 

Among the interesting new features added in DPDK 17.05 is the new event-driven programming model library (`rte_eventdev`). In this model, as opposed to the polling model, the cores call the DPDK scheduler, which selects packets for them. This model adds support for dynamic load balancing, automatic multi-core scaling, and more. Until 17.05, the DPDK cryptodev API had supported only Intel hardware accelerators; a new poll mode driver was added by NXP for its [Data Path Acceleration Architecture Gen2](<http://dpdk.org/doc/guides/nics/dpaa2.html>) cryptographic accelerators. 

One of the more interesting features introduced in the previous release, DPDK 17.02, is the [generic flow API (rte_flow)](<http://dpdk.org/doc/guides/prog_guide/rte_flow.html>), which provides a generic means to configure hardware to match specific ingress or egress traffic. In [the upcoming 17.08 release](<http://dpdk.org/dev/roadmap>), one can expect to see features like support for a generic quality-of-service API, [generic receive offload](<https://lwn.net/Articles/358910/>) support, and more. 

#### A simple DPDK application

Before delving into the details, let's take a look at a simple [layer 2 (L2)](<https://en.wikipedia.org/wiki/Data_link_layer>) forwarding DPDK application; becoming familiar with it will help to understand and develop more advanced DPDK applications. With this program, packets arriving at one port will be forwarded back via a second port after switching the source and destination MAC addresses. 

After initializations of ports, queues, and other settings via generic calls like `[rte_eth_dev_configure()](<http://dpdk.org/doc/api/rte__ethdev_8h.html#ac30d075b4b206c7122e200164ce69893>)`, the program enters the following loop: 

```
struct rte_mbuf *m;
        /* ... */
        while (!force_quit) {
    	/* ... */
    	nb_rx = rte_eth_rx_burst((uint8_t) portid, 0, pkts_burst, MAX_PKT_BURST);
    	port_statistics[portid].rx += nb_rx;
    	for (j = 0; j < nb_rx; j++) {
      	    m = pkts_burst[j];
    	    /* ... */
    	    l2fwd_simple_forward(m, portid);
    	}
```

In this loop, we read received packets (represented by the `rte_mbuf` structure) from the incoming port in a burst of size `MAX_PKT_BURST`, update the stats, and then each packet is processed by `l2fwd_simple_forward()`, which switches the source and the destination MAC addresses of this packet and transmits it via the outgoing port by invoking `[rte_eth_tx_buffer()](<http://dpdk.org/doc/api/rte__ethdev_8h.html#a1b95c114276a21717b20be4cbf112223>)`. 

This example, (like other DPDK applications) uses a high-level DPDK API, which does not depend on the implementation details of any specific DPDK network driver. Those who want to delve into the full source code for this example can find it [here](<http://dpdk.org/browse/dpdk/tree/examples/l2fwd/main.c>). More information can also be found in the [Sample Applications User Guides](<http://dpdk.org/doc/guides/sample_app_ug/>). 

#### DPDK components

Those who want to start learning and exploring DPDK could start with the many sample applications on the [examples page](<http://dpdk.org/browse/dpdk/tree/examples/>). There are over 40 of them, starting from a simple "hello world" and proceeding to more complex applications like IP pipelining and an IPSec gateway. All these examples are well documented. It is also recommended learning to use the [`testpmd`](<http://dpdk.org/doc/guides/testpmd_app_ug/index.html>) tool, which enables you to start and stop packet forwarding, display statistics, configure various settings, and more. 

For those who want to become familiar with the DPDK API, it is recommended to explore the [Programmer's Guide](<http://dpdk.org/doc/guides/prog_guide>) and the fundamental data structures. Those structures include the [`rte_mbuf` structure](<http://dpdk.org/doc/api/structrte__mbuf.html>) (representing a packet) and the [`rte_ethdev` structure](<http://dpdk.org/doc/api/rte__ethdev_8h.html#details>) (representing a network device). One should also learn the [Environment Abstraction Layer](<http://dpdk.org/doc/guides/prog_guide/env_abstraction_layer.html>) API. 

For more advanced DPDK knowledge, it is worth learning the memory pools implementation (the [`rte_mempool`](<http://dpdk.org/doc/api/structrte__mempool.html>) object and the [`librte_mempool` library](<http://dpdk.org/doc/guides/prog_guide/mempool_lib.html>)). Those who are seeking familiarity with the cryptographic layer can explore the [`rte_cryptodev`](<http://dpdk.org/doc/api/structrte__cryptodev.html>) structure, representing a cryptographic device. See also the [cryptodev API](<http://dpdk.org/doc/guides-16.04/prog_guide/cryptodev_lib.html>), which provides cryptographic poll-mode drivers as well as a standard API that supports all these drivers and can be used to perform cipher, authentication, and symmetric cryptographic operations. The library also enables migration between hardware and software cryptographic accelerators. One should become familiar with the [`dpdk-devbind` script](<http://dpdk.org/doc/guides/tools/devbind.html>) in order to bind and unbind devices and in order to view the status of the NICs. 

The DPDK web site contains a set of open-source tools such as the [dpdk-ci](<http://dpdk.org/browse/tools/dpdk-ci/>) continuous-integration suite and the [DPDK test suite (DTS)](<http://dpdk.org/doc/dts/gsg/>), which is a Python-based testing framework. DTS works with software traffic generators like [Scapy](<http://www.secdev.org/projects/scapy/>) and [pktgen-dpdk](<http://dpdk.org/browse/apps/pktgen-dpdk/>); it can also be used with the IXIA hardware traffic generator. DTS is easy to set up and run; it contains over 90 test modules for various networking scenarios. Here, again, one can start with a simple "hello world" test, and end up with complex tests including [SR-IOV](<https://en.wikipedia.org/wiki/Single-root_input/output_virtualization>) and live migration. Currently DTS supports Intel and Mellanox NICs, and patches for Cavium Networks NICs are circulating on the DTS mailing list. DTS provides both functional tests as well as benchmarking tests. 

The DPDK site also hosts pktgen-dpdk, which is a DPDK-based traffic generator. There are more DPDK-based, open-source traffic generators, including [TRex](<https://wiki.fd.io/view/TRex>), which has both a stateful mode (which can be helpful when testing load balancers and NATs for example) and a stateless mode, and the LuaJIT-based [MoonGen project](<https://github.com/emmericp/MoonGen>). 

Work has been done to add DPDK plugins to [collectd](<https://collectd.org/>), which is a popular system statistics collection daemon. Two DPDK plugins have been merged into collectd: [dpdkevents](<https://github.com/collectd/collectd/blob/master/src/dpdkevents.c>) and [dpdkstat](<https://github.com/collectd/collectd/blob/master/src/dpdkstat.c>). The dpdkevents plugin retrieves the DPDK link status and the DPDK forwarding core's status. The dpdkstat plugin polls statistics from DPDK drivers. 

#### DPDK at higher layers

While DPDK applications are focused mostly on layer 2, there are several interesting projects under FD.io that use DPDK as their primary I/O layer, including [VPP](<https://wiki.fd.io/view/VPP>). Also worth a mention is the [Transport Layer Development Kit (TLDK)](<https://wiki.fd.io/view/TLDK>) project, implementing a set of libraries for Layer-4 protocol processing. For those who are interested to learn more about TLDK, we suggest watching Ray Kinsella's talk at FOSDEM 2017: [Accelerating TCP with TLDK](<https://fosdem.org/2017/schedule/event/tldk/>). 

#### DPDK and the community

All DPDK development is done over the public [dev@dpdk.org](<http://dpdk.org/ml/archives/dev/>) mailing list. The guidelines for contributing code to DPDK are described [here](<http://dpdk.org/doc/guides/contributing/patches.html>). Long-term support releases are available, with support for two years. Governance for DPDK is provided by [two boards](<http://dpdk.org/about/charter>): a Governing Board (budget, marketing, etc.) and a Technical Board (technical issues including approval of new sub-projects, deprecating old sub-projects, etc). 

The DPDK project is a community-driven project and, as such, there are several DPDK events across the globe. The last DPDK Summits were held in Bangalore in April 2017 (the first DPDK Summit to be held in India) and [the Shanghai summit](<https://dpdksummit.com/>), which ws held in June. Many [videos from past events are available](<https://dpdksummit.com/us/en/past-events>); there is also more information in the [Intel Developer Zone](<https://software.intel.com/en-us/networking/dpdk>) and in the [Intel Network Builders University Program](<https://builders.intel.com/university/networkbuilders>). 

#### Summary

The DPDK project has become a popular open-source, user-space network and cryptographic acceleration solution based on bypassing the kernel. This project is gaining momentum, especially with the recent move to the Linux Foundation; it is worth following, experimenting with, and contributing to.
