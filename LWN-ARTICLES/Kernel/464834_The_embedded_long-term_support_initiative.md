---
title: "The embedded long-term support initiative"
url: https://lwn.net/Articles/464834/
date: "October 29, 2011"
category: "Long-term support initiative"
author: "By Jonathan Corbet October 29, 2011"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
October 29, 2011

The Linux Foundation's Consumer Electronics Working Group (the group known as the Consumer Electronics Linux Forum before the two organizations merged) chose the Embedded Linux Conference Europe as the forum for the announcement of a new mechanism for the maintenance of stable kernels for the embedded industry. If all goes well, this "long-term support initiative" (LTSI) will provide better kernels for embedded systems with less effort while increasing the amount of code that the embedded industry sends upstream. 

The initiative was presented by Tsugikazu Shibata, a manager at NEC and a member of the Linux Foundation's board. According to Shibata-san, current long-term supported kernels do not really meet the embedded industry's needs. Those kernels are well suited to the needs of the enterprise [![\[Tsugikazu Shibata\]](https://static.lwn.net/images/conf/2011/lc-europe/TsugikazuShibata-sm.jpg)](<https://lwn.net/Articles/464838/>) distributions, where the same kernel can be used and supported for periods up to ten years. A number of those distributions are built on 2.6.32, which can expect to be supported for some time yet. Enterprise distributions last for a long time, so the kernels they use are picked rarely and supported for many years. 

In the embedded world, 2.6.32 is very old news. Product lifetimes are much shorter for most embedded products; manufacturers in this area need a new, supported kernel every year. This industry, however, has no infrastructure for the support of kernels on that sort of schedule. Last year the industry came together and decided to standardize on 2.6.35, but it was a one-shot action; no plans were made to support any later kernel versions. That is a problem: but products being designed now are likely to need something newer than 2.6.35. 

Another problem is that finding common ground for a standard embedded kernel is hard. Much of the industry is currently driven by Android, which releases with new kernels every six months or so. Manufacturers, though, tend to get their kernels from their suppliers; those kernels can be based on almost any version and lack any support going forward. Those manufacturers need that support. They also would really like to use the same kernel for a few generations of a product; that requires support for a period of a few years, but it also requires a certain amount of backporting of drivers and features. 

Yet another problem, one that has characterized the embedded industry for years, is that there still are not enough contributions to the mainline from embedded developers (though, in all fairness, things have improved considerably). Manufacturers have a lot of patches in house, many of which do good stuff, but they are not going upstream. That imposes costs on those manufacturers, who have to carry those patches forward, and it impoverishes the mainline. 

The LTSI project will have three components to address these problems. The first of those will be a long-term stable tree for the embedded industry. A new tree will be selected roughly once each year; 3.0 has been picked as the initial kernel in this series. Each long-term kernel will receive updates for two years as part of the usual stable update process; as with the other stable kernels, only bug fixes and security updates will be considered. Some sort of advisory board will be set up with a number of industry developers to pick subsequent stable kernels. 

The second component will be the "LTS industry tree," which will be maintained, by CE Working Group members, independently from the regular stable updates. This is the tree that, it is expected, will actually be used in products. It will be based on the long-term releases, but will include some extras: backported drivers, perhaps some backported features, and various other vendor patches. In addition, there will be an associated "staging tree" where interesting code can be tested out prior to its inclusion in the industry tree. A separate quality assurance group will devote itself to testing the code in the staging tree and deciding when it can graduate. 

The normal path for getting code into the industry tree will be to get it upstream; a backport can then be done. There is, however, an intended mechanism to take code directly via the LTSI staging tree in unusual or urgent cases. Shibata-san was clear, though, that "upstream first" will be the usual rule for this tree. 

Finally, there will be an initiative to help industry engineers get their code upstream. Yet another staging tree will hold these patches while they are made suitable for inclusion into the mainline. 

While it is assumed that the embedded industry is carrying a lot of code internally, nobody has ever really known for sure. To get a better handle on how much out-of-tree code exists, the Working Group launched the "Yami-nabe Project." Yami-nabe parties are, evidently, an old, no longer observed Japanese custom. Everybody would show up to a dark room containing a large pot of water; each would bring some item of food and toss it in. Everybody would then eat the resulting soup without knowing what was in it or where it came from - whether they liked it or not. 

The modern form of a Yami-nabe party, it seems, involves collecting out-of-tree patches from manufacturers without tracking where they came from. None of the companies involved want to have their work compared to that of others. The collected code was examined to see how much of the kernel had been modified, and how much duplicated code there was. Turns out there were a lot of files in the kernel that most or all manufacturers felt the need to modify; much was in architecture-specific and driver code, but there were a lot of core kernel code changes too. Quite a few vendors had made similar changes in the same places. 

So clearly there is some duplication of effort going on. The LTSI tree, Shibata-san said, should help to reduce this duplicated effort and to reduce the kernel fragmentation in the embedded industry. Throw in some help with the upstreaming of code and, they hope, fragmentation in this area can be eliminated entirely - or at least something close to that. 

The first kernel will be 3.0-based which is, not coincidentally, the base for the Android "ice cream sandwich" kernel. They hope to have an industry kernel release available sometime in the first half of 2012. After that, it's a matter of seeing what kind of uptake there is in the embedded industry. It seems, though, that quite a few companies are interested in this project, so the chances that they will at least look at its output are pretty good. 

[Thanks to the Linux Foundation for supporting your editor's travel to Prague to attend this event]
