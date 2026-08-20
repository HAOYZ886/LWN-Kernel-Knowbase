---
title: Routing Open vSwitch into the mainline
url: https://lwn.net/Articles/469775/
date: "November 30, 2011"
category: "Networking; Open vSwitch; Virtualization-Network"
author: "By Jonathan Corbet November 30, 2011"
---

By **Jonathan Corbet**  
November 30, 2011

Visitors to the [features page](<http://openvswitch.org/features/>) on the [Open vSwitch web site](<http://openvswitch.org/>) may be forgiven if they do not immediately come away with a good understanding of what this package does. The feature list is full of enlightening bullet points like "LACP (IEEE 802.1AX-2008)", "802.1ag link monitoring", and "Multi-table forwarding pipeline with flow-caching engine". Behind the acronyms, Open vSwitch is a virtual switch that has already seen a lot of use in the Xen community and which is applicable to most other virtualization schemes as well. After some years as an out-of-tree project, Open vSwitch has recently made a push for inclusion into the mainline kernel. 

Open vSwitch is a network switch; at its lowest level, it is concerned with routing packets between interfaces. It is aimed at virtualization users, so, naturally, it is used in the creation of virtual networks. A switch can be set up with a number of virtual network interfaces, most of which are used by virtual machines to communicate with each other and the wider world. These virtual networks can be connected across hosts and across physical networks. One of the key features of Open vSwitch appears to be the ability to easily migrate virtual machines between physical hosts and have their network configuration (addresses, firewall rules, open connections, etc.) seamlessly follow. 

Needless to say, there is no shortage of features beyond making it easier to move guests around. Open vSwitch offers a long list of options for access control, quality-of-service control, network bridging, traffic monitoring, and more. The [OpenFlow](<http://www.openflow.org/>) protocol is supported, allowing the integration of interesting protocols and controllers into the network. Open vSwitch has been shipped as part of a number of products and it shows; it has the look of a polished, finished offering. 

Most of Open vSwitch is implemented in user space, but there is one kernel module that makes the whole thing work; that module was [submitted for review](<https://lwn.net/Articles/468522/>) in mid-November. Open vSwitch tries to make use of existing networking features to the greatest extent possible; the kernel module mostly implements a control interface allowing the user-space code to make routing decisions. Routing packets through user space would slow things down considerably, so the interface is set up to avoid the user-space round trip whenever possible. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

When the Open vSwitch module receives a packet on one of its interfaces, it generates a "flow key" describing the packet in general terms. An example key from the submission is: 

```
in_port(1), eth(src=e0:91:f5:21:d0:b2, dst=00:02:e3:0f:80:a4),
        eth_type(0x0800), ipv4(src=172.16.0.20, dst=172.18.0.52, proto=17, tos=0,
        frag=no), tcp(src=49163, dst=80)
```

Most of the fields should be fairly self-explanatory; this key describes a packet that arrived on port (interface) 1, aimed at TCP port 80 on host 172.18.0.52. If Open vSwitch does not know how to process the packet, it will pass it to the user-space daemon, along with the generated flow key. The daemon can then decide what should be done; it will also, normally, pass a rule back to the kernel describing how to handle related packets in the future. These rules start with the flow key, which may be generalized somewhat, and include a set of associated actions. Possible actions include: 

  * Output the packet to a specific port, forwarding it on its way to its final destination. 

  * Send the packet to user space for further consideration. The destination process may or may not be the main Open vSwitch control daemon. 

  * Make changes to the packet header on its way through; network address translation could be implemented this way, for example. 

  * Add an 802.1Q virtual LAN header in preparation for tunneling the packet to another host; there is also an action for stripping such headers at the receiving end. 

  * Record attributes of the packet for statistics generation. 

Once a rule for a given type of packet has been installed into the kernel, future packets can be routed quickly without the need for further user-space intervention. If the switch is working properly, most packets should never need to go through the control daemon. 

Open vSwitch, by all appearances, is a useful and powerful mechanism; the networking developers seem to agree that it would be a good addition to the kernel. There is, however, some disagreement over the implementation. In particular, the patch adds a new packet classification and control mechanism, but the kernel already has a traffic control system of its own; duplicating that infrastructure is not a popular idea. As Jamal Hadi Salim [put it](<https://lwn.net/Articles/469793/>): 

You are replicating a lot of code and semantic that exist (not just on classifier actions). You could improve the existing infrastructure instead. 

Jamal suggested that Open vSwitch could add a special-purpose classifier for its own needs, but that classifier should fit into the existing traffic control subsystem. 

That said, there seems to be some awareness within the networking community that the kernel's traffic controller may not quite be up to the task. Eric Dumazet [noted](<https://lwn.net/Articles/469794/>) that its scalability is not what it could be and that the code reflects its age; he said: ""Maybe its time to redesign a new model, based on modern techniques."" Others seemed to agree with this assessment. The traffic controller, it appears, is in need of serious improvements or replacement regardless of what happens with Open vSwitch. 

The fact that the traffic controller is not everything Open vSwitch needs will not normally be considered an adequate justification for duplicating its infrastructure, though. The obvious options available to the Open vSwitch developers will be to (1) improve the traffic controller to the point that it does work, or (2) position the Open vSwitch controller as a plausible long-term replacement. Neither task is likely to be easy. The outcome of this discussion may well be that developers who were hoping to merge their existing code will find themselves tasked with a fair amount of infrastructural work. 

That can be the point where those developers take option (3): go away and continue to maintain their code out of tree. Requiring extra work from contributors can cause them to simply give up. But if the networking maintainers accept duplicated subsystems, the likely outcome is a lot of wasted work and multiple implementations of the same functionality, none of which is as good as it should be. There are solid reasons behind the maintainers' tendency to push back against that kind of contribution; without that pushback, the long-term maintainability of the kernel will suffer. 

How things will be resolved in the case of Open vSwitch remains to be seen; the discussion is ongoing as of this writing. Open vSwitch is a healthy and active project; it may well have the resources and the desire to perform the necessary work to get into the mainline and ease its own long-term maintenance burden. Meanwhile, as was discussed at the 2011 Kernel Summit, code that is being shipped and used has value; sometimes it is best to get it into the mainline and focus on improving it afterward. Some developers (such as [Herbert Xu](<https://lwn.net/Articles/469937/>)) seem to think that may be the best approach to take in this case. So Open vSwitch may yet find its way into the mainline in the near future with the idea that its internals can be fixed up thereafter.
