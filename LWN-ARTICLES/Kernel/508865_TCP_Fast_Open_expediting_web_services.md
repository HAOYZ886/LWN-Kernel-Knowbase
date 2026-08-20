---
title: "TCP Fast Open: expediting web services"
url: https://lwn.net/Articles/508865/
date: "August 1, 2012"
category: Networking; TCP
author: "By Michael Kerrisk August 1, 2012"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Michael Kerrisk**  
August 1, 2012

Much of today's Internet traffic takes the form of short TCP data flows that consist of just a few round trips exchanging data segments before the connection is terminated. The prototypical example of this kind of short TCP conversation is the transfer of web pages over the Hypertext Transfer Protocol (HTTP). 

The speed of TCP data flows is dependent on two factors: transmission delay (the width of the data pipe) and propagation delay (the time that the data takes to travel from one end of the pipe to the other). Transmission delay is dependent on network bandwidth, which has increased steadily and substantially over the life of the Internet. On the other hand, propagation delay is a function of router latencies, which have not improved to the same extent as network bandwidth, and the speed of light, which has remained stubbornly [constant](<http://en.wikipedia.org/wiki/Speed_of_light>). (At intercontinental distances, this physical limitation means that--leaving aside router latencies--transmission through the medium alone requires several milliseconds.) The relative change in the weighting of these two factors means that over time the propagation delay has become a steadily larger component in the overall latency of web services. (This is especially so for many web pages, where a browser often opens several connections to fetch multiple small objects that compose the page.) 

Reducing the number of round trips required in a TCP conversation has thus become a subject of keen interest for companies that provide web services. It is therefore unsurprising that Google should be the originator of a series of patches to the Linux networking stack to implement the TCP Fast Open (TFO) feature, which allows the elimination of one round time trip (RTT) from certain kinds of TCP conversations. According to the implementers (in ["TCP Fast Open", _CoNEXT 2011_ [PDF]](<http://conferences.sigcomm.org/co-next/2011/papers/1569470463.pdf>)), TFO could result in speed improvements of between 4% and 41% in the page load times on popular web sites. 

[We first wrote](<https://lwn.net/Articles/458610/#tfo>) about TFO back in September 2011, when the idea was still in the development stage. Now that the TFO implementation is [starting](<http://thread.gmane.org/gmane.linux.network/237733>) to make its way into the kernel, it's time to visit it in more detail. 

#### The TCP three-way handshake

To understand the optimization performed by TFO, we first need to note that each TCP conversation begins with a round trip in the form of the so-called three-way handshake. The three-way handshake is initiated when a client makes a connection request to a server. At the application level, this corresponds to a client performing a `connect()` system call to establish a connection with a server that has previously bound a socket to a well-known address and then called `accept()` to receive incoming connections. Figure 1 shows the details of the three-way handshake in diagrammatic form. 

> ![\[TCP Three-Way Handshake\]](https://static.lwn.net/images/2012/tfo/3whs.png)

_Figure 1: TCP three-way handshake between a client and a server_

During the three-way handshake, the two TCP end-points exchange `SYN` (synchronize) segments containing [options](<http://www.iana.org/assignments/tcp-parameters/tcp-parameters.txt>) that govern the subsequent TCP conversation--for example, the [maximum segment size (MSS)](<http://en.wikipedia.org/wiki/Maximum_segment_size>), which specifies the maximum number of data bytes that a TCP end-point can receive in a TCP segment. The SYN segments also contain the initial sequence numbers (ISNs) that each end-point selects for the conversation (labeled _M_ and _N_ in Figure 1). 

The three-way handshake serves another purpose with respect to connection establishment: in the (unlikely) event that the initial `SYN` is duplicated (this may occur, for example, because underlying network protocols duplicate network packets), then the three-way handshake allows the duplication to be detected, so that only a single connection is created. If a connection was established before completion of the three-way handshake, then a duplicate `SYN` could cause a second connection to be created. 

The problem with current TCP implementations is that data can only be exchanged on the connection after the initiator of the connection has received an `ACK` (acknowledge) segment from the peer TCP. In other words, data can be sent from the client to the server only in the third step of the three-way handshake (the ACK segment sent by the initiator). Thus, one full round trip time is lost before data is even exchanged between the peers. This lost RTT is a significant component of the latency of short web conversations. 

Applications such as web browsers try to mitigate this problem using [HTTP persistent connections](<http://en.wikipedia.org/wiki/HTTP_persistent_connections>), whereby the browser holds a connection open to the web server and reuses that connection for later HTTP requests. However, the effectiveness of this technique is decreased because idle connections may be closed before they are reused. For example, in order to limit resource usage, busy web servers often aggressively close idle HTTP connections. The result is that a high proportion of HTTP requests are _cold_ , requiring a new TCP connection to be established to the web server. 

#### Eliminating a round trip

Theoretically, the initial `SYN` segment _could_ contain data sent by the initiator of the connection: [RFC 793](<http://www.ietf.org/rfc/rfc793.txt>), the specification for TCP, _does_ permit data to be included in a `SYN` segment. However, TCP is prohibited from delivering that data to the application until the three-way handshake completes. This is a necessary security measure to prevent various kinds of malicious attacks. For example, if a malicious client sent a `SYN` segment containing data and a spoofed source address, and the server TCP passed that segment to the server application before completion of the three-way handshake, then the segment would both cause resources to be consumed on the server and cause (possibly multiple) responses to be sent to the victim host whose address was spoofed. 

The aim of TFO is to eliminate one round trip time from a TCP conversation by allowing data to be included as part of the `SYN` segment that initiates the connection. TFO is designed to do this in such a way that the security concerns described above are addressed. ([T/TCP](<http://tools.ietf.org/html/rfc1644>), a mechanism designed in the early 1990s, also tried to provide a way of short circuiting the three-way handshake, but fundamental [security flaws](<http://www.phrack.org/issues.html?issue=53&id=6#article>) in its design meant that it never gained wide use.) 

On the other hand, the TFO mechanism does _not_ detect duplicate `SYN` segments. (This was a deliberate choice made to simplify design of the protocol.) Consequently, servers employing TFO must be [idempotent](<http://en.wikipedia.org/wiki/Idempotent>)--they must tolerate the possibility of receiving duplicate initial `SYN` segments containing the same data and produce the same result regardless of whether one or multiple such `SYN` segments arrive. Many web services are idempotent, for example, web servers that serve static web pages in response to URL requests from browsers, or web services that manipulate internal state but have internal application logic to detect (and ignore) duplicate requests from the same client. 

In order to prevent the aforementioned malicious attacks, TFO employs security cookies (TFO cookies). The TFO cookie is generated once by the server TCP and returned to the client TCP for later reuse. The cookie is constructed by encrypting the client IP address in a fashion that is reproducible (by the server TCP) but is difficult for an attacker to guess. Request, generation, and exchange of the TFO cookie happens entirely transparently to the application layer. 

At the protocol layer, the client requests a TFO cookie by sending a `SYN` segment to the server that includes a special TCP option asking for a TFO cookie. The `SYN` segment is otherwise "normal"; that is, there is no data in the segment and establishment of the connection still requires the normal three-way handshake. In response, the server generates a TFO cookie that is returned in the `SYN-ACK` segment that the server sends to the client. The client caches the TFO cookie for later use. The steps in the generation and caching of the TFO cookie are shown in Figure 2\. 

> ![\[Generating the TFO cookie\]](https://static.lwn.net/images/2012/tfo/foc_creation.png)

_Figure 2: Generating the TFO cookie_

At this point, the client TCP now has a token that it can use to prove to the server TCP that an earlier three-way handshake to the client's IP address completed successfully. 

For subsequent conversations with the server, the client can short circuit the three-way handshake as shown in Figure 3\. 

> ![\[Employing the TFO cookie\]](https://static.lwn.net/images/2012/tfo/foc_use.png)

_Figure 3: Employing the TFO cookie_

The steps shown in Figure 3 are as follows: 

  1. The client TCP sends a `SYN` that contains both the TFO cookie (specified as a TCP option) _and_ data from the client application. 

  2. The server TCP validates the TFO cookie by duplicating the encryption process based on the source IP address of the _new_ `SYN`. If the cookie proves to be valid, then the server TCP can be confident that this `SYN` comes from the address it claims to come from. This means that the server TCP can immediately pass the application data to the server application. 

  3. From here on, the TCP conversation proceeds as normal: the server TCP sends a `SYN-ACK` segment to the client, which the client TCP then acknowledges, thus completing the three-way handshake. The server TCP can also send response data segments to the client TCP _before_ it receives the client's `ACK`. 

In the above steps, if the TFO cookie proves not to be valid, then the server TCP discards the data and sends a segment to the client TCP that acknowledges just the `SYN`. At this point, the TCP conversation falls back to the normal three-way handshake. If the client TCP is authentic (not malicious), then it will (transparently to the application) retransmit the data that it sent in the `SYN` segment. 

Comparing Figure 1 and Figure 3, we can see that a complete RTT has been saved in the conversation between the client and server. (This assumes that the client's initial request is small enough to fit inside a single TCP segment. This is true for most requests, but not all. Whether it might be technically possible to handle larger requests--for example, by transmitting multiple segments from the client before receiving the server's `ACK`--remains an open question.) 

There are various details of TFO cookie generation that we don't cover here. For example, the algorithm for generating a suitably secure TFO cookie is implementation-dependent, and should (and can) be designed to be computable with low processor effort, so as not to slow the processing of connection requests. Furthermore, the server should periodically change the encryption key used to generate the TFO cookies, so as to prevent attackers harvesting many cookies over time to use in a coordinated attack against the server. 

There is one detail of the use of TFO cookies that we will revisit below. Because the TFO mechanism allows a client that submits a valid TFO cookie to trigger resource usage on the server before completion of the three-way handshake, the server can be the target of resource-exhaustion attacks. To prevent this possibility, the server imposes a limit on the number of pending TFO connections that have not yet completed the three-way handshake. When this limit is exceeded, the server ignores TFO cookies and falls back to the normal three-way handshake for subsequent client requests until the number of pending TFO connections falls below the limit; this allows the server to employ [traditional measures against SYN-flood attacks](<http://tools.ietf.org/html/rfc4987>). 

#### The user-space API

As noted above, the generation and use of TFO cookies is transparent to the application level: the TFO cookie is automatically generated during the first TCP conversation between the client and server, and then automatically reused in subsequent conversations. Nevertheless, applications that wish to use TFO must notify the system using suitable API calls. Furthermore, certain system configuration knobs need to be turned in order to enable TFO. 

The changes required to a server in order to support TFO are minimal, and are highlighted in the code template below. 

```
sfd = socket(AF_INET, SOCK_STREAM, 0);   // Create socket
    
        bind(sfd, ...);                          // Bind to well known address
        **int qlen = 5;                            // Value to be chosen by application
        setsockopt(sfd, SOL_TCP, TCP_FASTOPEN, &qlen, sizeof(qlen));
        **
        listen(sfd, ...);                        // Mark socket to receive connections
    
        cfd = accept(sfd, NULL, 0);              // Accept connection on new socket
    
        // read and write data on connected socket cfd
    
        close(cfd);
```

Setting the `TCP_FASTOPEN` socket option requests the kernel to use TFO for the server's socket. By implication, this is also a statement that the server can handle duplicated `SYN` segments in an idempotent fashion. The option value, `qlen`, specifies this server's limit on the size of the queue of TFO requests that have not yet completed the three-way handshake (see the remarks on prevention of resource-exhaustion attacks above). 

The changes required to a client in order to support TFO are also minor, but a little more substantial than for a TFO server. A normal TCP client uses separate system calls to initiate a connection and transmit data: `connect()` to initiate the connection to a specified server address and (typically) `write()` or `send()` to transmit data. Since a TFO client combines connection initiation and data transmission in a single step, it needs to employ an API that allows both the server address and the data to be specified in a single operation. For this purpose, the client can use either of two repurposed system calls: `sendto()` and `sendmsg()`. 

The `sendto()` and `sendmsg()` system calls are normally used with datagram (e.g., UDP) sockets: since datagram sockets are connectionless, each outgoing datagram must include both the transmitted data and the destination address. Since this is the same information that is required to initiate a TFO connection, these system calls are recycled for the purpose, with the requirement that the new `MSG_FASTOPEN` flag must be specified in the `flags` argument of the system call. A TFO client thus has the following general form: 

```
sfd = socket(AF_INET, SOCK_STREAM, 0);
        **sendto(sfd, data, data_len, MSG_FASTOPEN, 
                    (struct sockaddr *) &server_addr, addr_len);
            // Replaces connect() + send()/write()
        **
        // read and write further data on connected socket sfd
    
        close(sfd);
```

If this is the first TCP conversation between the client and server, then the above code will result in the scenario shown in Figure 2, with the result that a TFO cookie is returned to the client TCP, which then caches the cookie. If the client TCP has already obtained a TFO cookie from a previous TCP conversation, then the scenario is as shown in Figure 3, with client data being passed in the initial `SYN` segment and a round trip being saved. 

In addition to the above APIs, there are various knobs--in the form of files in the `/proc/sys/net/ipv4` directory--that control TFO on a system-wide basis: 

  * The `tcp_fastopen` file can be used to view or set a value that enables the operation of different parts of the TFO functionality. Setting bit 0 (i.e., the value 1) in this value enables client TFO functionality, so that applications can request TFO cookies. Setting bit 1 (i.e., the value 2) enables server TFO functionality, so that server TCPs can generate TFO cookies in response to requests from clients. (Thus, the value 3 would enable both client and server TFO functionality on the host.) 

  * The `tcp_fastopen_cookies` file can be used to view or set a system-wide limit on the number of pending TFO connections that have not yet completed the three-way handshake. While this limit is exceeded, all incoming TFO connection attempts fall back to the normal three-way handshake. 

#### Current state of TCP fast open

Currently, [TFO is an Internet Draft](<http://tools.ietf.org/html/draft-ietf-tcpm-fastopen-01>) with the IETF. Linux is the first operating system that is adding support for TFO. However, as yet that support remains incomplete in the mainline kernel. The client-side support has been merged for Linux 3.6. However, the server-side TFO support has not so far been merged, and from conversations with the developers it appears that this support won't be added in the current merge window. Thus, an operational TFO implementation is likely to become available only in Linux 3.7. 

Once operating system support is fully available, a few further steps need to be completed to achieve wider deployment of TFO on the Internet. Among these is assignment by [IANA](<http://www.iana.org/>) of a dedicated TCP Option Number for TFO. (The current implementation employs the [TCP Experimental Option Number](<http://www.iana.org/assignments/tcp-parameters/tcp-parameters.txt>) facility as a placeholder for a real TCP Option Number.) 

Then, of course, suitable changes must be made to both clients and servers along the lines described above. Although each client-server pair requires modification to employ TFO, it's worth noting that changes to just a small subset of applications--most notably, web servers and browsers--will likely yield most of the benefit visible to end users. During the deployment process, TFO-enabled clients may attempt connections with servers that don't understand TFO. This case is handled gracefully by the protocol: transparently to the application, the client and server will fall back to a normal three-way handshake. 

There are other deployment hurdles that may be encountered. In their _CoNEXT 2011_ paper, the TFO developers note that a minority of middle-boxes and hosts drop TCP `SYN` segments containing unknown (i.e., new) TCP options or data. Such problems are likely to diminish as TFO is more widely deployed, but in the meantime a client TCP can (transparently) handle such problems by falling back to the normal three-way handshake on individual connections, or generally falling back for all connections to specific server IP addresses that show repeated failures for TFO. 

#### Conclusion

TFO is promising technology that has the potential to make significant reductions in the latency of billions of web service transactions that take place each day. Barring any unforeseen security flaws (and the developers seem to have considered the matter quite carefully), TFO is likely to see rapid deployment in web browsers and servers, as well as in a number of other commonly used web applications.
