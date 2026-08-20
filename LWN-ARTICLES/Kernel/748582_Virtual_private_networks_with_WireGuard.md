---
title: Virtual private networks with WireGuard
url: https://lwn.net/Articles/748582/
date: "March 6, 2018"
category: "Networking-Virtual private networks"
author: "By Jonathan Corbet March 6, 2018"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
March 6, 2018

Virtual private networks (VPNs) offer a lot in the way of increased security and privacy. They have also tended to offer less desirable features like administrative complexity and reduced performance, though; as a result, many potential VPN users decide not to bother. A relatively new project called [WireGuard](<https://www.wireguard.com/>) hopes to address both of those problems with an in-kernel solution that is both simple and fast. 

A VPN works by establishing an encrypted connection from an endpoint system to a trusted host elsewhere on the network. That host becomes the router through which some or all network traffic from the endpoint passes. Since this tunnel is encrypted, traffic that travels over the VPN is protected from eavesdroppers — until it reaches the trusted host, at least. Setting up the VPN connection in the first place requires authentication between the endpoints; that, in turn, allows hosts to place some trust in the packets coming over the VPN connection. It is thus a common configuration to only allow internal resources to be accessed via a VPN connection. 

There are other advantages to VPNs as well. Today's youth tend to be well acquainted with the use of a VPN to bypass various types of content filtering. A VPN can be used to change a user's apparent network location, helping to circumvent annoyances like country-specific content blocking. The first eavesdropper many of us are likely to encounter is our own Internet service provider, which tends to take a great interest in collecting data on web traffic and such to sell to advertisers; using a VPN will frustrate this kind of prying. 

Linux users have a number of VPN options available to them. In many cases, a set of OpenSSH tunnels will do the trick for specific applications. [OpenVPN](<https://openvpn.net/>) is a popular, open-core system that provides comprehensive VPN functionality. But OpenVPN suffers from a fair amount of complexity and, since it is a user-space implementation, it takes a toll on networking performance. [IPsec](<https://en.wikipedia.org/wiki/IPsec>) is built into the kernel and has fewer performance problems, but it makes up for that with even more complexity. 

#### Enter WireGuard

In June 2016, Jason Donenfeld showed up with [a new VPN implementation called WireGuard](<https://lwn.net/Articles/693015/>) that claims to avoid the problems associated with other options. It is an in-kernel implementation (though still out of tree) that has been developed with performance in mind. The implementation is quite small (about 4,000 lines of code), making it relatively easy to verify. Configuration of the system is relatively simple though, as with any sort of network configuration it seems, the "relatively" qualification is important. 

Donenfeld has gone out of his way to make it easy to experiment with WireGuard; there are [prebuilt packages](<https://www.wireguard.com/install/>) available for a wide range of distributions. Those packages contain the source for the WireGuard implementation; it is built on the fly using the [DKMS](<https://github.com/dell/dkms>) framework. Once the installation is done, the user is left with a kernel module (`wireguard.ko`) and the `wg` tool for configuration. 

Every host connecting to a WireGuard implementation must use a public/private key pair for communication. The first step, thus, is to generate a new private key with a command like: 

```
# wg genkey
        uHCQ+Damh4F5zNVr9PvHiflW2aRU1SE0GQCVYkvxiEc=
```

The keys are generated using the [Curve25519](<https://en.wikipedia.org/wiki/Curve25519>) elliptic curve; as a result they are quite a bit shorter than keys used by other algorithms. The associated public key can be created from the private key with the `wg pubkey` command. 

WireGuard presents itself as a new type of network interface that can be used to route packets into a VPN. Thus, setting up a WireGuard implementation requires creating and configuring this interface, using a command series like: 

```
# ip link add wg0 type wireguard
        # ip addr add 10.0.0.1/24 wg0
        # wg set wg0 private-key <private-key-file>
        # ip link set wg0 up
```

These commands create a new network interface called `wg0`, loading the `wireguard` kernel module in the process. This interface is assigned the network address `10.0.0.1`, and its private key is set to a key generated with `wg genkey`. Just running a bare `wg` command at this point will produce output like: 

```
# wg
        interface: wg0
          public key: FNqV9pbUECLd7SNQ98jDlDRxqtppMTT9CEE8p1w6bTU=
          private key: (hidden)
          listening port: 41415
```

Like many recent protocols, WireGuard is based on UDP. Packets at one end are encrypted, then sent to the remote endpoint encapsulated within UDP packets, where they are decrypted and sent on their way. The above output tells us that port 41415 was chosen to listen for these UDP packets; the port number can also be explicitly configured with the `wg` command. 

A command series like this must be carried out at both ends of the VPN connection; the IP addresses should be different, of course, but on the same subnet (`10.0.0.0/24` in this case). Imagine we did something like that on the remote host, giving it IP address `10.0.0.2` and putting it on port 44556. The next step is to connect those two interfaces together so that they may pass packets back and forth. On the original machine (the one whose `wg0` interface has address `10.0.0.1`), we would run something like: 

```
wg set wg0 peer <public-key> allowed-ips 10.0.0.2/32 endpoint <ip-addr>:44556
```

Here, `ip-addr` is the real-world (not VPN) address of the other end of the connection. A similar command would be required on that other system, using the appropriate public key and IP address. At that point, it will be possible to communicate between the two hosts by using the appropriate addresses. Once the connection has been established the IP addresses can change; if one end is a laptop, for example, the VPN will still work after moving to a new network. 

In a sense, that's really about all there is to it. But the real world does tend to be a bit more complicated, of course. For example, it is common to want the endpoint to send _all_ of its network traffic over the VPN. That could be accomplished by setting the `allowed-ips` parameter in the above command to `0.0.0.0/0` and using `ip route` to set the default route to go through `wg0`. A slightly more complex setup (turning on IP forwarding, probably setting up NAT) would then be required on the other end to make the routing work. 

The advantage of the WireGuard approach can be seen here, though; it creates interfaces that can be connected to each other. After that all of the normal networking routing and traffic-control features can be used to cause that VPN link to be used in a wide variety of ways. There are also some special features designed to allow WireGuard interfaces to be used within network namespaces. 

I ran a test, using WireGuard to set up a link between the desktop machine and a remote cloud instance. It took a little while, but that is mostly a matter of being extremely rusty with the `ip` command set. The VPN tunnel worked as advertised in the end. Before enabling the tunnel, a [SpeedOf.Me](<https://speedof.me/>) test showed 137Mbps bandwidth down and 12.9Mbps up; the ping time to an LWN server was 76ms. With all traffic routing over the WireGuard VPN link, downward bandwidth dropped to 131Mbps and upward to 12.4Mbps; ping times were nearly unchanged. That is not a zero cost, but it is not huge and one should bear in mind that going through a NAT gateway at the far end will be a big chunk of the total performance hit. So WireGuard does indeed appear to be reasonably fast. 

One test is not a comprehensive evaluation, of course. It will be interesting to try WireGuard at the next conference with an overloaded network to see how well it copes with packet loss, for example, and no attempt was made to verify the cryptographic aspects of the protocol. WireGuard does seem like a relatively simple and fast VPN implementation, though, that could go a long way toward making VPN use nearly universal on Linux systems. 

#### Next steps

Getting to that point will require that WireGuard be merged into the mainline kernel, though. Donenfeld has stated that upstreaming the code was his intent from the beginning, but there have been almost no postings of the code on the kernel mailing lists. It is, thus, unsurprising that WireGuard remains out of tree. Donenfeld did post an [upstreaming roadmap](<https://lwn.net/Articles/748584/>) in November; it suggests that the code is unlikely to be merged right away since, for example, an overhaul of the cryptographic API is evidently a precondition. That overhaul has not yet happened, and neither has the promised near-term posting of the WireGuard code. 

Chances are that this all will happen eventually, though. WireGuard seems to have generated a high level of interest, and it appears to have been deployed in many settings already. It has, for example, been [integrated into OpenWrt](<https://danrl.com/blog/2017/luci-proto-wireguard/>) with a set of configuration screen in the LuCi web interface. So there is clearly an audience for this functionality. Once the process of getting it upstream begins in earnest, it may run its course relatively quickly. 

See [this white paper [PDF]](<https://www.wireguard.com/papers/wireguard.pdf>) for lots of details on how WireGuard works.
