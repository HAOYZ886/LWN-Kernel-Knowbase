---
title: IPv6 segment routing
url: https://lwn.net/Articles/722804/
date: "May 17, 2017"
category: "Networking-IPv6"
author: "May 17, 2017 This article was contributed by David Lebrun"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

May 17, 2017

This article was contributed by David Lebrun

In November 2016, a new networking feature, IPv6 segment routing (also known as "IPv6 SR" or "SRv6"), was [merged](<https://git.kernel.org/pub/scm/linux/kernel/git/davem/net-next.git/commit/
?id=5db5b395150186d4a177ebfa563894af302ab3ad>) into net-next and subsequently included in Linux 4.10. In this article, we explain this new feature, describe key elements of its implementation, and present a few performance measurements.

#### Routing at the source

Segment routing is a modern source-routing architecture that is being [developed](<https://tools.ietf.org/html/draft-ietf-spring-segment-routing-11>) within the IETF. Traditional IP routing uses destination-based hop-by-hop forwarding. Each intermediate router decides where to forward packets using its local routing table. This table is usually computed according to a shortest-path algorithm. As such, packets traverse networks following the shortest path toward their destination. This behavior is not always optimal, as the shortest path can have undesirable properties, such as congestion. 

Existing traffic-engineering techniques enable coarse-grained control over a packet's path by tuning parameters, such as link weights, to adjust the shortest path algorithm. However, these techniques cannot control traffic on a per-flow basis. Segment routing, instead, can forward packets along non-shortest paths toward their destination by specifying a list of detours or waypoints called "segments". This list is specified within the packet itself. Packets are forwarded along the shortest path from the source to the first segment, then through the shortest path from the first segment to the second segment, and so on. The source of the packet can thus specify its path with an arbitrary precision. Segment routing defines two main types of segments representing topological instructions. A "node" segment is used to steer packets through a specific network node, while an "adjacency" segment steers packets through a particular link.

The segment routing architecture has been instantiated in two different data planes: [Multiprotocol Label Switching](<https://en.wikipedia.org/wiki/Multiprotocol_Label_Switching>) ([MPLS](<https://tools.ietf.org/html/draft-ietf-spring-segment-routing-mpls-07>)) and [IPv6](<https://tools.ietf.org/html/draft-ietf-6man-segment-routing-header-06>). The MPLS variant of SR is already implemented by network vendors and deployed by operators.

SRv6 is implemented with an IPv6 extension header, the [routing header](<https://tools.ietf.org/html/rfc2460#section-4.4>). The [segment routing header](<https://tools.ietf.org/html/draft-ietf-6man-segment-routing-header-06#sect
ion-3>) (SRH) contains a list of segments, encoded as IPv6 addresses. A segment represents a topological instruction (node or link traversal) or any operator-defined instruction (e.g., a virtual function). An SRH can be used to steer packets through paths with given properties (e.g., bandwidth or latency) and through various network functions (such as a firewall). The list of segments present in the SRH thus specifies the network policy that applies to the packet. Each SRH contains at least a list of segments and a "segments left" pointer that references the current active segment (a value of zero refers to the last segment). In addition, an SRH can optionally include extended information encoded as type-length-value (TLV) fields. One example is the optional HMAC TLV, used for authenticity and integrity checks.

When a router must impose an SRH on a forwarded packet, the packet is encapsulated in an outer IPv6 header containing the SRH. The original packet is left unmodified as the payload. The router is called the "SR ingress node". The destination address of the outer IPv6 header is set to the first segment of the list and the packet is forwarded to the corresponding node following the shortest path. This node (called a "segment endpoint") then processes the packet by updating the destination address to the next segment. The last segment of the list is the "SR egress node". It decapsulates the inner packet and forwards it to its original destination. 

#### Data-plane implementation

The core of the SRv6 implementation is the SRH processing capability. It enables a Linux node to act as both a segment endpoint and an SR egress node. When a segment endpoint receives an IPv6 packet containing an SRH, the destination address of the packet is local to the segment endpoint. To process this packet, a new `ipv6_srh_rcv()` function is called whenever the IPv6 input function encounters an SRH in the header chain. An SRH is defined by the following structure:

```
struct ipv6_sr_hdr {
        	__u8    nexthdr;
        	__u8    hdrlen;
        	__u8    type;
       	__u8    segments_left;
        	__u8    first_segment;
        	__u8    flags;
        	__u16   reserved;
        	struct in6_addr segments[0];
        };
```

The `ipv6_srh_rcv()` function performs several operations. First, it checks that the node is allowed to act as a segment endpoint for SR-enabled packets coming from the ingress interface (`skb->dev`). This policy is configured through the sysctl boolean `net.ipv6.conf.<interface>.seg6_enabled`. If it is set to 0, then the packet is discarded. Otherwise, the processing continues. 

The packet then goes through an optional HMAC validation, controlled with the sysctl knob `net.ipv6.conf.<interface>.seg6_hmac_require`. This parameter can take three different values. If `seg6_hmac_require` is set to -1, the node accepts all SR packets, regardless of the status (absent/present, valid/invalid) of the HMAC TLV. When it is set to zero, the node must accept packets that contain an SRH and either present a valid HMAC TLV or do not include an HMAC TLV at all. Finally, a setting of 1 means that the node accepts only the SR packets that include a valid HMAC. 

The next operations depend on the value of the "segments left" counter. If it is greater than zero, then the packet must be forwarded to the next segment. The field is decremented and the destination address of the packet is updated to the next segment. A route lookup is performed and the packet is forwarded.

If, instead, there are no segment left, the operation will depend on the next header. If it is another extension header or a final payload such as TCP or UDP, then the kernel assumes that the packet must be delivered to a local application. The packet processing then continues as normal. On the other hand, if the next header is another IPv6 header, then the kernel assumes that the packet is an encapsulation. The inner packet is then decapsulated and reinjected in the ingress interface using `netif_rx()`.

#### Control plane implementation

To support SRH encapsulation, the SRv6 implementation uses the [lightweight tunnels](<https://lwn.net/Articles/651497/>) (LWT) infrastructure. LWTs can associate custom `input` and `output` function pointers (in `struct dst_entry`) to route objects. Those functions are called when a packet is forwarded by the kernel (input) and generated by a local application (output). A per-route stateful data structure (the "tunnel state") is also maintained. 

For SRv6, the function pointers are set to `seg6_input()` and `seg6_output()`. Those functions encapsulate a packet within an outer IPv6 header that contains a pre-configured SRH. The SRH is stored as the tunnel state, along with a `struct dst_cache`. This structure stores the route associated with the first segment of the SRH. Indeed, all packets that receive an SRH will be immediately forwarded to their first segment. Using a cache thus prevents an unnecessary route lookup for each processed packet.

To configure the routes, the `iproute2` tool has been extended. An SRv6 encapsulation route can be defined as follows:

```
# ip -6 route add fc42::/64 encap seg6 mode encap \
             segs fc00::1,2001:db8::1,fc10::7 dev eth0
```

This command inserts a route for the prefix `fc42::/64`. Each matching packet receives an SRH with three segments. Note that it is mandatory to specify a non-loopback interface.

The routes configured by `iproute2` are namespace-wide. To support finer-grained control of the SRH insertion, the SRv6 implementation also supports a per-socket interface, using the `setsockopt()` system call. This interface enables an application to specify its own per-socket SRH. The following piece of code shows how an application can use this feature:

```
struct ipv6_sr_hdr *srh;
        int srh_len;
    
        srh_len = build_srh(&srh);
        fd = socket(AF_INET6, SOCK_STREAM, IPPROTO_TCP);
        setsockopt(fd, IPPROTO_IPV6, IPV6_RTHDR, srh, srh_len);
```

#### HMAC

The authenticity and integrity of an SRH can be ensured using an HMAC TLV. This HMAC covers the source address of the packet and the full SRH (excluding TLVs). The HMAC TLV contains an opaque, operator-defined key ID. In the Linux SRv6 implementation, this identifier refers to a hashing algorithm (currently, SHA-1 and SHA-256 are supported) and to the HMAC secret.

The HMAC support is implemented using static per-CPU ring buffers to store the input data and pre-allocated, per-CPU algorithm descriptors to store the auxiliary buffer/context of the hashing algorithms. Pre-allocation prevents doing expensive dynamic memory allocation for each processed packet, and per-CPU data prevents lock contention.

To associate an HMAC with an SRv6 route, the `iproute2` tool provides an `hmac` keyword. The command `ip sr` enables configuration of the hashing algorithm and secret associated with a given key ID.

#### Performance

An evaluation shows that the SRH encapsulation performance is within 12% of plain IPv6 forwarding. The following figure summarizes the results, performed on Xeon X3440 machines with Intel 82599 10 Gbps interface cards:

> ![Performance chart](https://static.lwn.net/images/2017/srv6-perf.png)

In red is shown the performance for a single flow, processed by a single CPU. The blue bars show the performance for an arbitrarily large number of flows, with the receive-side scaling feature enabled (i.e., the flows are evenly processed by all the CPUs). With the multi-flow measurements, plain IPv6 forwarding peaks at about 4.9Mpps, while SRv6 encapsulation reaches 4.3Mpps. The HMAC computation (using the SHA-256 algorithm) performance reaches only 1.2Mpps. Profiling shows that most of the CPU cycles are used in the `sha256_transform()` function, which is the main processing function for the hashing algorithm.

The multi-flows measurements are about four times better than the single-flow measurements. This is explained by the fact that the test machine actually has four physical cores with HyperThreading enabled. Considering this, the SRv6 encapsulation performance seems to scale linearly with respect to the available CPU power.

SRv6 is a promising new source-routing technique. For obvious security reasons, it is intended to run within the bounds of an administrative domain. As such, it provides network operators with a powerful traffic engineering and flow management tool. With the help of a central controller, segment routing can implement a software-defined network paradigm, providing an alternative to [OpenFlow](<https://en.wikipedia.org/wiki/OpenFlow>)-based networks. Currently, multiple service providers are exploring the potential of SRv6. Enterprise networks may also benefit from this technique, as IPv6 deployments increase in this kind of environment.
