---
title: "Fingerprinting systems with TCP source-port selection"
url: https://lwn.net/Articles/910435/
date: "October 6, 2022"
category: "Networking-Security; Security-Vulnerabilities"
author: "By Jonathan Corbet October 6, 2022"
---

By **Jonathan Corbet**  
October 6, 2022

Back in May 2022, a mysterious set of patches titled [insufficient TCP source port randomness](<https://lwn.net/ml/linux-kernel/20220502084614.24123-1-w@1wt.eu/>) crossed the mailing lists and was subsequently merged (at -rc6) into the 5.18 kernel. Little information was available at the time about why significant changes to the networking stack needed to be made so late in the development cycle. That situation has finally changed with the publication of [this paper](<https://arxiv.org/pdf/2209.12993.pdf>) by Moshe Kol, Amit Klein, and Yossi Gilad. It seems that the way the kernel chose port numbers for outgoing network connections made it possible to uniquely fingerprint users. 

#### Selecting a source port

A TCP connection can be described as a four-tuple consisting of the source and destination IP addresses and the source and destination port numbers. The addresses and destination port number will all be fixed by the specific connection needed, but the originating side can choose any number for the source port number. It has long been understood that there is value in making those numbers unpredictable; to do otherwise would make connections more vulnerable to hazards like [reset attacks](<https://en.wikipedia.org/wiki/TCP_reset_attack>) or even data injection. So the Linux kernel has, since [this patch](<https://git.kernel.org/linus/190cc82489f4>) by Eric Dumazet was merged for 5.12, duly implemented source-port randomization as described in [RFC 6056](<https://www.rfc-editor.org/rfc/rfc6056>). 

The randomization algorithm needs to be difficult to predict and also fast; the Linux implementation meets those goals. But it turns out that there are other reasons to choose source-port numbers correctly. To understand why, it's worth looking quickly at how the Linux implementation, prior to 5.18, worked. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

In short, the kernel calculates two hashes, which the paper calls F and G, from the three given parts of the four-tuple (the addresses and the destination port number). To ensure that different systems produce different hashes for the same tuples, it also mixes in a 32-bit random key that is generated at boot time. F can be an arbitrarily large number, but G is constrained to the size of an array of counters. The port number is chosen with a calculation that looks approximately like: 

```
port = (counter_table[G] + F) % port_number_range;
```

The given counter is then incremented. Naturally, there are a number of complications, including checks for whether the port number is already in use. 

A key aspect of this algorithm is the sizing of the counter table. As [the 5.17 source](<https://elixir.bootlin.com/linux/v5.17/source/net/ipv4/inet_hashtables.c#L714>) (just prior to the fixes) notes, RFC 6056 suggests a ten-entry table, but Dumazet decided to go with 256 entries instead ""to really give more isolation and privacy"". 

#### The attack

Kol and company were able to come up with an interesting attack on this algorithm. A hostile web page (otherwise known as almost any page on today's Internet) could load a JavaScript fragment that, through a series of iterations, creates a mapping between destination port numbers and the counter-table entries used to assign source-port numbers. It is, in other words. looking for hash-table collisions on the counter table. This table, remember, has only 256 entries, so hash collisions will not be rare or hard to find. 

Specifically, the attack initiates a series of outgoing connections, all to the same remote address, but each to a different destination port. It then looks at the assigned source-port number for each connection attempt (note that the connection need not actually be established). Since any given counter-table entry is incremented after being used to generate a source-port number, two connection attempts that hit that counter-table entry will result in source-port numbers that differ by one — if the source and destination addresses are the same. So the attack looks for connection attempts that resulted in sequential source-port numbers and concludes that the destination-port numbers used in those attempts map to the same counter-table entry. 

The optimal number of outgoing connections for one iteration of this attack is said to be one less than the size of the counter table, or 255. A single iteration of this algorithm will produce at most a small number of collisions, which do not tell an attacker much, but it can be run over and over again to come up with more of them. So the above process is repeated until collisions have been found for each entry in the counter table. Once that is done, a second phase uses a similar technique, but mixing connections to a loopback address with connections to the remote-server destination ports found in the first phase. The purpose here is to find which destination ports, when used with a loopback destination, map to the same table cell as one of those remote-server port pairs. This second phase generates pairs of destination port numbers that, when used with the loopback address, generate collisions in the counter table; these port-number pairs are independent of any remote address. 

Each pair of colliding loopback port numbers, in effect, tells the attacker a little more about the secret key that the kernel generated at boot time. The key itself is never disclosed, but there is no need for that; a sufficient number of colliding port-number pairs is sufficient to uniquely identify the system involved. The key point is that these port-number pairs are a function of the secret key — which is different for every system — and can thus be used to create a unique device identifier. 

It evidently takes about 40 connection attempts per counter-table entry to generate enough collisions, so about 10,000 attempts to identify a system. (The paper describes how to calculate "enough" but doesn't give a number). The time required to carry out this attack is about ten seconds, and the resources used are small enough that chances are good that it will go undetected. (Naturally, this discussion has passed over a lot of important details and is almost certainly wrong somewhere along the way; see the paper for the full story). 

This unique identifier has some interesting characteristics. It is entirely independent of the software being run, so it will remain the same even if the user switches browsers. ~~or just fetches a page with a tool like` curl`.~~ It is also the same regardless of the site that is being connected to, so it works well for tracking users across multiple sites. Different containers running on the same host will all have the same identifier. Even systems with identical hardware and software configurations will produce different identifiers. 

In other words, this ability to identify a system looks like a gift to the surveillance capitalists out there. It does have a few limitations, though. It does not work through networks like Tor, since connections are terminated within the Tor network and initiated anew at an exit node. Network-address translation (NAT) systems, which reassign port numbers, can also interfere with the identification. As the authors point out, though, increasing use of IPv6 is likely to reduce the use of NAT, making NAT interference less of a problem. 

The identifier will also change when a system reboots. There is, however, a widespread class of devices — those running Android — that tend not to reboot frequently. The threat to Android seems to have been of special concern to the authors. It is not described as an immediate threat; Android devices are running, at the latest, 5.10 kernels, while the vulnerable port-selection code was added in 5.12. That said, the authors may have overlooked the fact that the "improved" port-selection code was also part of the 5.10.119 stable update, and may well be running on some Android systems. 

#### The fix

The patch set addressing the problem, posted by Willy Tarreau, makes a number of changes. One of these is to change the hash calculation to mix in yet another number that changes every ten seconds (it is, in fact, `jiffies/10*HZ`). That will perturb the selection of which counter to use and, as a result, will disrupt any identification attempt that is underway at the time. Another change is to increment the chosen counter by a random number (between zero and seven) rather than by one as a way of adding more noise to the chosen port numbers. 

Those changes might be sufficient to thwart the described attack, but only barely. The core of the response is, instead, to increase the size of the counter table to 65,536 entries. That bloats the table from 1KB in size to 256KB, but it also makes collisions much more uncommon and, thus, much harder to find, significantly increasing the time required to carry out a successful identification. The end result is a set of defenses that prevent the identification of systems via the source-port-number selection mechanism. 

The kernel's policy regarding security problems is normally to require disclosure shortly after the report is made. A brief embargo can be allowed while a fix is developed, but that is the extent of it. In this case, though, the fixes were initially posted in April, with no description of the problem that was motivating them. And, when your editor inquired into the issue at the time, the answer was that the explanation would not be forthcoming for several months. 

In this case, the lengthy period of secrecy seemingly had nothing to do with security. The fixes were public and were quickly incorporated into any kernel that is being maintained with an eye toward security problems. Instead, this delay was entirely created by the requirements of the journal publishing the article describing the vulnerability. That journal's demand for exclusivity, in a way that was convenient for its own publication schedule, prohibited the posting of an explanation of the vulnerability elsewhere. 

As a result, few developers were able to review the patches with regard to whether they actually fixed the problem they were targeted at. The kernel community had to rely on its trust of the developers involved (Dumazet had a hand in their creation). That is not really how the process is supposed to work. The kernel community has little patience with distributors seeking lengthy embargoes; it's not clear that academic journals merit more deference. 

Be that as it may, the problem appears to be well solved, and we now have an explanation of why those patches, first posted nearly six months ago, were needed. Whether it will ever be possible to eliminate all of the ways in which individual systems can be fingerprinted is an open question, but at least one readily available mechanism has been closed off.
