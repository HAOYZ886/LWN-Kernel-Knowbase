---
title: Linux gets DCCP
url: https://lwn.net/Articles/149756/
date: "August 30, 2005"
category: "DCCP; Networking-Protocols"
author: 
---

For many years, the bulk of networking over IP has made use of just two protocols: transmission control protocol (TCP) and user datagram protocol (UDP). TCP offers a reliable, stream-oriented connection which works well for a large variety of higher-level network protocols. UDP, instead, makes a best effort to move individual packets from one host to another, but makes no promises regarding reliability or ordering. Most higher-level protocols are built upon TCP, but there are applications which are better served by UDP. These include: 

  * Protocols involving brief exchanges which will be slowed unacceptably by TCP's connection overhead. A classic example is the domain name system, which can often achieve a name lookup with a single packet in each direction. 

  * Protocols where timely delivery is more important than reliability. These include internet telephony, streaming media, and certain kinds of online games. If the network drops a packet, TCP will stall the data flow until the sending side gets a successful retransmission through. But a telephony application would rather keep the data flowing and just do without the missing packet. 

The second type of application listed above is an increasingly problematic user of UDP. Streaming applications are a growing portion of the total traffic on the net, and they can be the cause of significant congestion. Unlike TCP, however, UDP has no concept of congestion control. In the absence of any sort of connection information, there is no way to control how any given application responds to network congestion. Early versions of TCP, lacking congestion control, brought about the virtual collapse of the early Internet; some fear that the growth of UDP-based traffic could lead to similar problems in the near future. 

This concern has led to the creation of the datagram congestion control protocol (DCCP), which is described by [this draft RFC](<http://www.icir.org/kohler/dcp/draft-ietf-dccp-spec-11.txt>). Like UDP, DCCP is a datagram protocol. It differs from UDP, however, in that it includes a congestion control mechanism. Eventually, it is hoped that users of high-bandwidth, datagram-oriented protocols will move over to DCCP as a way of getting better network utilization while being fair to the net as a whole. Further down the road, after DCCP has proved itself, it would not be surprising to see backbone network routers beginning to discriminate against high bandwidth UDP users. 

DCCP is a connection-oriented protocol, requiring a three-packet handshake before data can be transferred. For this reason, it is unlikely to take over from UDP in some areas, such as for DNS lookups. (There is a provision in the protocol for sending data with the connection initiation packet, but implementations are not required to accept that data). The higher-bandwidth applications tend to use longer-lived connections, however, so they should not even notice the connection setup overhead. 

Actually, DCCP uses a concept known as "half connections." A DCCP half connection is a one-way, unreliable data pipe; most applications will create two half connections to send data in both directions. The two half connections can be tied together to the point that, as with TCP, a data packet traveling in one direction can carry an acknowledgment for data received from the other. In other respects, however, the two half connections are distinctly separate from each other. 

One way in which this separation can be seen is with congestion control. TCP hides congestion control from user space entirely; it is handled by the protocol, with the system administrator having some say over which algorithms are used. DCCP, on the other hand, recognizes that different protocols will have different needs, and allows each half connection to negotiate its own congestion control regime. There are currently two "congestion control ID profiles" (CCIDs) defined: 

  * [CCID 2](<http://www.icir.org/kohler/dcp/draft-ietf-dccp-ccid2-10.txt>) uses an algorithm much like that used with TCP. A congestion window is used which can vary rapidly depending on net conditions; this algorithm will be quick to take advantage of available bandwidth, and equally quick to slow things down when congestion is detected. (See [this LWN article](<http://lwn.net/Articles/128681/>) for more information on how TCP congestion control works). 

  * [CCID 3](<http://www.icir.org/kohler/dcp/draft-ietf-dccp-ccid3-11.txt>), called "TCP-friendly rate control" or TFRC, aims to avoid quick changes in bandwidth use while remaining fair to other network users. To this end, TFRC will respond more slowly to network events (such as dropped packets) but will, over time, converge to a bandwidth utilization similar to what TCP would choose. 

It is anticipated that applications which send steady streams of packets (telephony and streaming media, for example) would elect to use TFRC congestion control. For this sort of application, keeping the data flowing is more important than using every bit of bandwidth which is available at the moment. A control connection for an online game, instead, may be best served by getting packets through as quickly as possible; applications using this sort of connection may opt for the traditional TCP congestion control mechanism. 

DCCP has a number of other features aimed at minimization of overhead, resistance to denial of service attacks, and more. For the most part, however, it can be seen as a form of UDP with explicit connections and congestion control. Porting UDP applications to DCCP should not be particularly challenging - once platforms with DCCP support have been deployed on the net. 

To that end, one of the first things which was merged for 2.6.14 was a DCCP implementation for Linux. This work was done by Arnaldo Carvalho de Melo, Ian McDonald, and others. It is a significant bunch of code; beyond the DCCP implementation itself, Arnaldo has done a lot of work to generalize parts of the Linux network stack. Much of the code which was once useful only for TCP or UDP can now also be shared with DCCP. 

For now, only CCID 3 (TFRC) has been implemented. A CCID 2 implementation, taking advantage of the TCP congestion control code, will follow. Even before that, however, the 2.6.14 kernel will be the first widely deployed DCCP implementation on the net. As such, it will likely help to find some of the remaining glitches in the protocol and shape its future evolution. When DCCP hits the mainstream, one can be reasonably well sure that the Linux implementation will be second to none.
