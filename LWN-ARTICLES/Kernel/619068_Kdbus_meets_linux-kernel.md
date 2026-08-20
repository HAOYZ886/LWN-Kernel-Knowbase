---
title: "Kdbus meets linux-kernel"
url: https://lwn.net/Articles/619068/
date: "November 4, 2014"
category: Message passing; kdbus
author: "By Jonathan Corbet November 4, 2014"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
November 4, 2014

There has been a long history of attempts to put interprocess messaging systems into the Linux kernel; in general, these attempts have not gotten very far. From the beginning, though, the expectations around "kdbus," an in-kernel implementation of the widely used D-Bus mechanism, have been higher than the usual. Kdbus has been under development for more than two years, and was [unveiled](<https://lwn.net/Articles/580194/>) at linux.conf.au in January. But it had never been posted to the linux-kernel mailing list for review and, with luck, eventual inclusion — until October 29, when Greg Kroah-Hartman [posted a twelve-part series](<https://lwn.net/Articles/618608/>) for consideration. 

In short, kdbus is a mechanism by which processes can find each other and exchange messages. It is meant to facilitate certain kinds of interprocess communications in a way that is both secure and reasonably fast. For those wanting details, [this document](<https://lwn.net/Articles/619069/>) covers kdbus functionality in a fairly thorough way. 

#### A whirlwind overview

For those not wanting to read an 1800-line file, here's a brief summary. When kdbus starts up, it creates a set of device nodes under `/dev/kdbus`; any actions involving kdbus require opening one or more of those nodes. A "bus" is essentially a namespace within which processes can communicate with each other. A fairly normal default configuration involves a single "system" bus for communicating with privileged services, and one "user" bus for each logged-in user. The user bus would be used, for example, to allow the processes implementing the user's desktop environment to talk to each other. 

While there is a single bus namespace at boot time, things need not remain that way. A set of buses exists within a kdbus "domain"; domains are organized into a hierarchy. So, for example, a container-management system would create a new domain for each container, then use a bind mount to make the appropriate subtree of `/dev/kdbus` available within the container. Thereafter, processes within the container can communicate with each other without having any access to communications outside of the container. There is currently no provision for using kdbus to communicate between containers. 

Messages are, at their simplest, a set of bytes with no interpretation by the kernel at all. Messages can pass file descriptors between processes; the passing of [sealed files and memfds](<https://lwn.net/Articles/593918/>) is also supported. Message recipients can specify a set of sender credentials that must be supplied with a message for policy checking; those credentials are attached to the message by the kernel. There is also a built-in policy mechanism describing which processes can adopt "well-known names" and which processes can communicate with which others. 

Kdbus is intended to be fast with both large and small messages. For the largest of messages, zero-copy transfer between processes is supported. Experience has shown, though, that a message must be about 512KB or larger before page-mapping tricks become cheaper than just copying the data. There is support for broadcast messages, along with a mechanism based on [bloom filters](<http://en.wikipedia.org/wiki/Bloom_filter>) for filtering out unwanted broadcasts without waking up the (uninterested) recipients. 

In general, kdbus is meant to be a replacement for D-Bus that addresses the various issues that have come up with the latter over time. The goal is not to be the ultimate messaging system for all possible applications. While the kdbus developers are open to the idea of adding more functionality in the future, they are trying to keep a lid on the complexity at this stage. 

#### Reviews

Given the (systemd-ish) origins of the kdbus code, one might well have expected the discussion to be somewhat hostile at times. In truth, while there have been concerns expressed, the discussion has remained mostly friendly and entirely technical. Developers are taking a deep look at the code and discussing how it can be improved; one cannot say that kdbus is not getting a fair hearing. 

One of the initial questions was, inevitably, why does this functionality need to be in the kernel in the first place? The kernel already provides a number of interprocess communication primitives, and tools like D-Bus have successfully used them for many years. See [this message from Greg](<https://lwn.net/Articles/619071/>) for a detailed answer. In short, it comes down to performance (fewer context switches to send a message), security (the kernel can ensure that credentials passed with messages are correct), race-free operation, the ability to use buses in early boot, and more. There do seem to be legitimate reasons to want this kind of functionality built into the kernel. 

#### Credentials

The handling of credentials drew a couple of different criticisms; the first was that credentials are checked when a message is sent — not when the connection to the bus is first created. Eric Biederman raised [concerns](<https://lwn.net/Articles/619108/>) that failure to capture credentials at `open()` time could lead to exploitable vulnerabilities. He did not actually point out any such vulnerabilities, though, and, in the past, such vulnerabilities have tended to be associated with later `read()` and `write()` calls. Since kdbus does not support either call on any of its file descriptors, that kind of vulnerability should not be an issue here. Still, there is some discomfort among the more security-oriented reviewers that the late capture of credentials is asking for trouble. 

Another problem, [raised](<https://lwn.net/Articles/619109/>) by Andy Lutomirski, is that checking credentials at message-sending time makes privilege-separation architectures impossible: 

The issue is the following: if have the privilege needed to talk to journald, I may want to enhance security by opening a connection to journald (and capture that privilege) and then drop privilege. I should still be able to talk to journald. 

If that privilege is checked every time a message is sent, the ability to drop privileges in this way is lost. Kdbus developer Daniel Mack [responded](<https://lwn.net/Articles/619110/>) that, in the D-Bus world (which carries over into the kdbus design), there is no concept of "opening a connection" to a service like journald. Instead, one connects to a bus and sends messages to services; each message has to stand on its own. As Daniel put it: 

This is why we have this functionality of passing over the caller creds every time a method call is made. The focus is really on the individual method call transaction, each one is individually routed, dispatched and checked for permission. Hence, it should carry individual credential information from the time the call is issued. 

This particular disagreement reflects a fundamental difference in how developers see kdbus being used. It does not look like an easy one to resolve without some significant design changes on the kdbus side; any such changes would move it away from the D-Bus model and are likely to encounter resistance from the kdbus developers. 

A related issue, also [raised](<https://lwn.net/Articles/619112/>) by Andy, is that the recipient of a message specifies which credential information should accompany that message. This information can include user and group IDs, process command line, control group information, capabilities, security labels, the audit session information, and more. The sender of a message has no control over whether this information is sent. Andy thinks that sending this information will lead to information leaks and security problems. 

Instead, Andy said, the sending process should explicitly specify which credential information should accompany a message and that security-related requests should explicitly document what credentials are required. ""Otherwise it becomes unclear what things convey privilege when, and that will lead immediately to incomprehensible security models, and that will lead to exploits."" The [response](<https://lwn.net/Articles/619113/>) from kdbus developer Tom Gundersen is that ""by simply connecting to the bus and sending a message to some service, you implicitly agree to passing some metadata along to the service"". It allows the recipient to be sure that the necessary information will be supplied, even if the recipient's security model changes (requiring different information) in the future. Again, Andy disagrees, insisting that the provision of credentials should be a matter of negotiation between both sides. 

#### Namespace and device issues

Both Eric and Andy also raised an entirely different set of concerns having to do with the way the domain namespace works. The decision to attach globally visible names to domains leads to some unfortunate consequences in their view. The first (and smaller) of these is that the existence of a namespace forces kdbus domains into a hierarchical structure, even though there is nothing that is actually hierarchical about them. Each domain is an independent entity with no particular relation to its parent domain outside of the naming scheme. 

The real problem, though, is that a global namespace implies the need for some sort of control to keep malicious processes from polluting that namespace. That, in turn, means that creating a kdbus domain is a privileged operation. Quite a bit of work has gone into allowing unprivileged users to create containers. But if a new container cannot be given a kdbus domain without privilege, that model breaks down. Lennart Poettering acknowledged this concern in [an apparently private email](<https://lwn.net/Articles/619114/>) publicly responded to by Andy; he said that allowing unprivileged domain creation should be possible, as long as the checks for namespace collisions remain in place. 

Andy's reply there was that none of the other container-oriented primitives have global names, and that there is a reason for that: avoidance of just this type of namespace collision possibility. Kdbus domains, he asserts, would be better off without the globally visible names. There would appear to be a couple of reasons why these names exist. One would be to make it easy for a privileged process to tap into any domain and watch traffic for debugging purposes. That particular need could probably be met by way of a domain pointer in each process's `/proc` area. 

The bigger problem relates to another fundamental kdbus design decision: to base the whole thing around device nodes found in `/dev`. If there are kdbus devices for multiple domains in `/dev`, they must be organized into that directory's hierarchical namespace. Such a namespace is essentially unavoidable if the device nodes are to be available to (and, importantly, locatable by) processes in the system. For this reason, a couple of reviewers have said that the device abstraction is a mistake. Rather than implementing kdbus operations as a set of `ioctl()` calls on a device, perhaps kdbus should have a set of dedicated system calls that would eliminate the need for the device nodes altogether. That would also eliminate the need for a global kdbus domain namespace. 

Eric [expressed](<https://lwn.net/Articles/619116/>) a related concern: the use of device nodes implies the existence of dynamically allocated device numbers. That will interfere with the checkpointing and restoring of containers, since there is no way to guarantee that the same device numbers will be available when the container is restored. That breaks a use case that works with D-Bus today, so Eric has described it as a regression. 

#### Going forward

From one perspective, the response on the mailing list should be encouraging for the kdbus developers. While the obligatory "why do this in the kernel?" questions were asked, there does not appear to be much fundamental opposition to putting this kind of functionality into the kernel. That suggests that, sooner or later, the kernel will have an answer for users who have asked for a native messaging solution. 

The form of that solution remains up in the air, though. Kdbus will clearly have to change to address the review comments that have been posted (and those yet to come); how radical that change needs to be remains to be seen. It could be that, as Alan Cox [put it](<https://lwn.net/Articles/619142/>), ""it would be far more constructive to treat the current kdbus as a proof of concept/prototype or even a draft requirements specification"". Or perhaps the concerns that have been raised can be addressed with a simpler set of changes. 

Either way, it does not look like the long-playing kdbus story will come to a close anytime soon. That may be frustrating for those who are waiting for this functionality to become available in a mainline kernel. But this process can only be hurried so much if the end result is to be a solution that will stand the test of time. Once kdbus goes into the kernel it will become much harder to change, so it is worth taking the time to get the interface (and its semantics) right first.
