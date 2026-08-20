---
title: "Namespaces in operation, part 7: Network namespaces"
url: https://lwn.net/Articles/580893/
date: "January 22, 2014"
category: Namespaces
author: "By Jake Edge January 22, 2014 Namespaces in operation"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jake Edge**  
January 22, 2014

* * *

[Namespaces in operation](<https://lwn.net/Articles/531114/#series_index>)

It's been a while since last we looked at Linux namespaces. Our series has been missing a piece that we are finally filling in: network namespaces. As the name would imply, network namespaces partition the use of the network—devices, addresses, ports, routes, firewall rules, etc.—into separate boxes, essentially virtualizing the network within a single running kernel instance. Network namespaces entered the kernel in 2.6.24, almost exactly five years ago; it took something approaching a year before they were ready for prime time. Since then, they seem to have been largely ignored by many developers. 

#### Basic network namespace management

As with the others, network namespaces are created by passing a flag to the `clone()` system call: `CLONE_NEWNET`. From the command line, though, it is convenient to use the `ip` networking configuration tool to set up and work with network namespaces. For example: 

```
# ip netns add netns1
```

This command creates a new network namespace called `netns1`. When the `ip` tool creates a network namespace, it will create a bind mount for it under `/var/run/netns`; that allows the namespace to persist even when no processes are running within it and facilitates the manipulation of the namespace itself. Since network namespaces typically require a fair amount of configuration before they are ready for use, this feature will be appreciated by system administrators. 

The "`ip netns exec`" command can be used to run network management commands within the namespace: 

```
# ip netns exec netns1 ip link list
        1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT 
            link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
```

This command lists the interfaces visible inside the namespace. A network namespace can be removed with: 

```
# ip netns delete netns1
```

This command removes the bind mount referring to the given network namespace. The namespace itself, however, will persist for as long as any processes are running within it. 

#### Network namespace configuration

New network namespaces will have a loopback device but no other network devices. Aside from the loopback device, each network device (physical or virtual interfaces, bridges, etc.) can only be present in a single network namespace. In addition, physical devices (those connected to real hardware) cannot be assigned to namespaces other than the root. Instead, virtual network devices (e.g. virtual ethernet or veth) can be created and assigned to a namespace. These virtual devices allow processes inside the namespace to communicate over the network; it is the configuration, routing, and so on that determine who they can communicate with. 

When first created, the `lo` loopback device in the new namespace is down, so even a loopback `ping` will fail: 

```
# ip netns exec netns1 ping 127.0.0.1
        connect: Network is unreachable
```

Bringing that interface up will allow pinging the loopback address: 

```
# ip netns exec netns1 ip link set dev lo up
        # ip netns exec netns1 ping 127.0.0.1
        PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
        64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.051 ms
        ...
```

But that still doesn't allow communication between netns1 and the root namespace. To do that, virtual ethernet devices need to be created and configured: 

```
# ip link add veth0 type veth peer name veth1
        # ip link set veth1 netns netns1
```

The first command sets up a pair of virtual ethernet devices that are connected. Packets sent to `veth0` will be received by `veth1` and vice versa. The second command assigns `veth1` to the netns1 namespace. 

```
# ip netns exec netns1 ifconfig veth1 10.1.1.1/24 up
        # ifconfig veth0 10.1.1.2/24 up
```

Then, these two commands set IP addresses for the two devices. 

```
# ping 10.1.1.1
        PING 10.1.1.1 (10.1.1.1) 56(84) bytes of data.
        64 bytes from 10.1.1.1: icmp_seq=1 ttl=64 time=0.087 ms
        ...
        
        # ip netns exec netns1 ping 10.1.1.2
        PING 10.1.1.2 (10.1.1.2) 56(84) bytes of data.
        64 bytes from 10.1.1.2: icmp_seq=1 ttl=64 time=0.054 ms
        ...
```

Communication in both directions is now possible as the `ping` commands above show. 

As mentioned, though, namespaces do not share routing tables or firewall rules, as running `route` and `iptables -L` in netns1 will attest. 

```
# ip netns exec netns1 route
        # ip netns exec netns1 iptables -L
```

The first will simply show a route for packets to the 10.1.1 subnet (using `veth1`), while the second shows no iptables configured. All of that means that packets sent from netns1 to the internet at large will get the dreaded "Network is unreachable" message. There are several ways to connect the namespace to the internet if that is desired. A [bridge](<http://www.linuxfoundation.org/collaborate/workgroups/networking/bridge>) can be created in the root namespace and the veth device from netns1. Alternatively, IP forwarding coupled with [network address translation](<http://en.wikipedia.org/wiki/Network_address_translation>) (NAT) could be configured in the root namespace. Either of those (and there are other configuration possibilities) will allow packets from netns1 to reach the internet and for replies to be received in netns1. 

Non-root processes that are assigned to a namespace (via `clone()`, `unshare()`, or `setns()`) only have access to the networking devices and configuration that have been set up in that namespace—root can add new devices and configure them, of course. Using the `ip netns` sub-command, there are two ways to address a network namespace: by its name, like netns1, or by the process ID of a process in that namespace. Since `init` generally lives in the root namespace, one could use a command like: 

```
# ip link set vethX netns 1
```

That would put a (presumably newly created) veth device into the root namespace and it would work for a root user from any other namespace. In situations where it is not desirable to allow root to perform such operations from within a network namespace, the PID and mount namespace features can be used to make the other network namespaces unreachable. 

#### Uses for network namespaces

As we have seen, a namespace's networking can range from none at all (or just loopback) to full access to the system's networking capabilities. That leads to a number of different use cases for network namespaces. 

By essentially turning off the network inside a namespace, administrators can ensure that processes running there will be unable to make connections outside of the namespace. Even if a process is compromised through some kind of security vulnerability, it will be unable to perform actions like joining a botnet or sending spam. 

Even processes that handle network traffic (a web server worker process or web browser rendering process for example) can be placed into a restricted namespace. Once a connection is established by or to the remote endpoint, the file descriptor for that connection could be handled by a child process that is placed in a new network namespace created by a `clone()` call. The child would inherit its parent's file descriptors, thus have access to the connected descriptor. Another possibility would be for the parent to send the connected file descriptor to a process in a restricted network namespace via a Unix socket. In either case, the lack of suitable network devices in the namespace would make it impossible for the child or worker process to make additional network connections. 

Namespaces could also be used to test complicated or intricate networking configurations all on a single box. Running sensitive services in more locked-down, firewall-restricted namespace is another. Obviously, container implementations also use network namespaces to give each container its own view of the network, untrammeled by processes outside of the container. And so on. 

Namespaces in general provide a way to partition system resources and to isolate groups of processes from each other's resources. Network namespaces are more of the same, but since networking is a sensitive area for security flaws, providing network isolation of various sorts is particularly valuable. Of course, using multiple namespace types together can provide even more isolation for both security and other needs.
