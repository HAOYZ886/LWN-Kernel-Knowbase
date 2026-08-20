---
title: Pluggable congestion avoidance modules
url: https://lwn.net/Articles/128681/
date: "March 22, 2005"
category: "Networking-Congestion control"
author: 
---

Many years ago, when the TCP/IP protocols were young, the early Internet went through a bad period. As the number of systems on the net grew, the high-speed (56K) long-haul links which tied the backbone sites together became clogged and the net became very difficult to use. The TCP implementations in use at that time did not understand how to deal with (or even detect) congestion, and, as a result, made the problem worse. Some people began to ask if TCP was going to work at all. 

Van Jacobson saved the situation with a simple observation: there is no point in sending data faster than the slowest link between the endpoints can handle it, even if the local network connection is very fast. Overwhelming the long-haul link just causes lots of dropped packets, retransmissions, and even more congestion. The solution was to start transmitting data slowly on a new connection, then to ramp up the speed until packets started getting dropped. The optimal speed was deemed to be one at which just a very small number of packets would fail to arrive. That speed would be adjusted over the life of the connection as conditions on the network changed. With TCP tweaked in this way, communicating systems would scale back their transmissions as the network got more congested, but would ramp up when the bandwidth became available. The result was a net which actually worked for everybody involved. It became possible, for example, to download the entire GNU emacs distribution without having to split it into dozens of small pieces first. 

We had to content ourselves with what we could get in those days. 

Since then, the net has become much larger, more complex, and faster. The congestion avoidance problem has grown as well, to the point that there are several competing algorithms seeking to provide the best TCP performance while being fair to other network users. Several of these algorithms have found their way into Linux, with a corresponding increase in the complexity of the TCP code. As a way of helping those experimenting with congestion avoidance and eliminating the need to patch the TCP code directly, Stephen Hemminger has posted [a new infrastructure](<https://lwn.net/Articles/128626/>) which allows congestion avoidance algorithms to be written as pluggable modules. He has also reworked the existing algorithms in the kernel to use the new infrastructure. The result is, among other things, an opportunity to look at how these algorithms work. 

The core of the TCP protocol is the concept of a "window," being the amount of data which one side is willing to accept from the other at any given time. The window size reflects what the receiving system can handle - how much buffer space it has available - but it says nothing about what the routers in between can deal with. Congestion avoidance algorithms try to account for the slowest link serving a connection with a "congestion window," which is the maximum amount of data which can be in transit without an acknowledgment from the remote end. An ideal congestion window setting will allow a system to maximize throughput on a connection without excessive packet loss rates, and without taking an unfair amount of the shared network bandwidth. Finding that setting is still more of an art than a science. 

Stephen's patches create a new structure to identify a congestion avoidance algorithm: 

```
struct tcp_ca_type {
    	void (*start)(struct tcp_sock *tp);
    	u32 (*ssthresh)(struct tcp_sock *tp);
    	u32 (*min_cwnd)(struct tcp_sock *tp);
    	void (*cong_avoid)(struct tcp_sock *tp, u32 ack, 
    			   u32 rtt, u32 in_flight, int good);
    	void (*rtt_sample)(struct tcp_sock *tp, u32 rtt);
    	void (*set_state)(struct tcp_sock *tp, u8 new_state);
    
    	void (*cwnd_event)(struct tcp_sock *tp, enum tcp_ca_event ev);
    	u32  (*undo_cwnd)(struct tcp_sock *tp);
    	void (*get_info)(struct tcp_sock *tp, u32 ext, struct sk_buff *skb);
    
    	struct list_head	list;
    	struct module 		*owner;
    	const char 		*name;
        };
```

Each of the methods in this structure is a hook into the TCP code which allows the algorithm to obtain information on network conditions and react accordingly: 

  * The `start()` method initializes the algorithm when a new batch of data is being transmitted; this can happen for new sockets, or when one has been idle for a while. 

  * The `ssthresh()` method calculates the "slow start threshold"; when the congestion window is below that threshold, the connection is in slow start mode rather than full congestion avoidance mode. This method is called when congestion occurs. 

  * The actual initial window may be set by `min_cwnd()` to be less than the threshold value as a starting point for the slow start algorithm. 

  * When an acknowledgment arrives from the remote end, the `cong_avoid()` method is invoked; it may respond to successful packet delivery by enlarging the congestion window. 

  * `rtt_sample()` tells the algorithm about a measured round-trip time - the time taken between sending a packet and receiving the corresponding acknowledgment. 

  * `set_state()` indicates that the TCP state of the socket has changed. 

  * Various events of interest can be communicated to the algorithm via `cwnd_event()`. 

  * Sometimes, transient situations can cause the congestion window to be reduced; the `undo_cwnd()` method can be called when such a situation is detected to restore a larger window. 

  * The `get_info()` method can be used to make congestion avoidance information available to user space. 

The TCP "Reno" algorithm is Van Jacobson's original; it remains wired into the kernel in a non-pluggable form (though it can be overridden). The congestion window starts at the `min_cwnd()` value, and increases by one segment for each sequential acknowledgment received from the remote end until it hits the slow-start threshold. At that point, the congestion window increases much more slowly until it either hits the TCP window size or packet loss happens. When congestion is detected, the congestion window is cut in half (to a minimum of two segments) and the process starts over. 

The [Westwood](<http://www.cs.ucla.edu/NRL/hpi/tcpw/>) algorithm is a tweak to the Reno approach. The Westwood code carefully tracks the round-trip times of the packets sent, and uses that information to estimate the effective bandwidth of the network connection. When packets get dropped, the congestion window and slow start thresholds are set relative to that bandwidth estimate. As a result, Westwood tends to back off more slowly than Reno, and should, thus, get better bandwidth overall. Its authors claim that Westwood is especially good for wireless links or other situations where the loss of an occasional packet may have nothing to do with congestion. 

[TCP Vegas](<http://www.cs.arizona.edu/protocols/>) also makes use of detailed round-trip time information. In particular, it tries to address a perceived failure in the Reno algorithm, which determines the optimal packet rate by increasing the congestion window until that rate is exceeded. Vegas, instead, monitors changes to the packet round-trip time as the congestion window is increased. If a larger window leads to longer round-trip times, the algorithm concludes that congestion is about to set in and the window is reduced slightly. The Vegas algorithm (or at least the Linux implementation thereof) does not perform well in all environments, so it is not enabled by default. 

[Binary Increase Congestion Control (BIC) [PDF]](<http://www.csc.ncsu.edu/faculty/rhee/export/bitcp.pdf>) tries to be smarter about how the congestion window size is adjusted. Among other things, it is aimed at high-performance networks where the TCP window may be quite large. The other algorithms may, in congestion avoidance mode, make large changes to the congestion window which can result in abrupt increases in network traffic. The BIC code combines two algorithms as a way of trying to quickly converge on the proper congestion window while avoiding massive packet dumps. The core technique is a binary search; if the window is to be increased, the point midway between the current value and the maximum size is chosen. Decreases are handled by picking the midpoint between the current value and the threshold. If, however, the endpoints are too far apart, an "additive increase" is done instead - the congestion window is resized by a constant value. 

The [High-speed TCP](<ftp://ftp.rfc-editor.org/in-notes/rfc3649.txt>) algorithm is optimized for very fat pipes - 10G Ethernet and such. When things are congested, it behaves much like the Reno algorithm. When the congestion window is being increased, however, the high-speed algorithm makes use of a table to pick large increment values. This approach lets the congestion window get very large (i.e. tens of thousands of segments) quickly, and to stay large, without requiring that the network function for long periods of time without a single dropped packet. 

The last of the pluggable modules is the [TCP Hybla](<https://lwn.net/Articles/124483/>) implementation. Hybla is based on the observation that the other algorithms, which use round-trip times heavily in their calculations, tend to be biased against satellite links and other high-latency connections. So Hybla includes a calculation which allows the congestion window to become larger, more quickly, when the round-trip time is very high. In this way, it tries to keep the pipe full enough to make use of the available bandwidth, even though the time to turn around any individual packet is long. 

Stephen is currently suggesting that this patch set should go into 2.6.13, after a good shakedown period in the -mm tree. There does not seem to be a whole lot of opposition, so things may well happen just that way.
