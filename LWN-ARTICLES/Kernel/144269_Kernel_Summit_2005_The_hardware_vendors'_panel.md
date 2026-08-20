---
title: "Kernel Summit 2005: The hardware vendors' panel"
url: https://lwn.net/Articles/144269/
date: "July 19, 2005"
category: Development model
author: 
---

From LWN's [2005 Kernel Summit coverage](<https://lwn.net/Articles/KernelSummit2005/>).   
---  
The second day of the 2005 Kernel Summit started with a panel session on device drivers, hardware vendors, and interactions with the development community. Bringing hardware vendors into the community remains a difficult issue, one which creates a fair amount of confusion and frustration on both sides. Sessions like this one may slowly help to bring a greater level of understanding, but real solutions will be some time in coming. 

The first panelist was James Smart from Emulex, which first started working toward inclusion of its drivers when the distributions insisted on it. If a driver is not in the mainline, the distributors are increasingly uninterested in shipping or supporting it. Mainline inclusion also makes life easier from a customer support point of view; it is better if you do not have tell your customers to apply patches to their kernels to be able to use your products. 

Working toward inclusion requires some changes in the corporate development process. When code is to be reviewed by the community, it must meet that community's standards; that includes following common coding patterns and generally writing the code to be easy to understand quickly. Drivers are expected to use common code, but, more importantly, the "second man in" is often expected to recognize commonality with existing code and help _create_ the common code to be shared with other drivers. Developers who are not prepared to work with the community in this way may find themselves surprised. Similarly, a driver writer should submit fixes for problems found elsewhere in the kernel, rather than simply working around them. 

Managers tend to like fixed schedules, but the development community does not work that way. Part of getting a company into the community is teaching its management to cope with uncertainty in scheduling. Trying to impose a deadline on the inclusion process is a sure recipe for frustration. 

One of the big frustrations James mentioned - and which was echoed by other members of the panel - is the need to support multiple versions of the kernel. Different distributors ship different versions of the kernel, and each needs to be supported. The community frowns on compatibility glue, so it is hard to keep a single version of a driver around. This is simply a fact of life, an expense which must be dealt with. 

Attempting to maintain a single driver which works on other operating systems as well is completely out of the question. 

There are advantages to all this: the resulting driver will be smaller, cleaner, and easier to maintain into the future. Reviews from kernel developers can also be helpful in overcoming internal barriers to change. On the other hand, the inconsistencies between kernel versions and distributions can be painful. Add in customers with their own patches, and the situation gets even worse. 

Another problem can be waiting for other, important parts of the kernel to mature. The device mapper code was given as an example here. 

The next speaker was Andrew Vasquez from QLogic. His brief talk went over some of the hassles he has had to deal with. At the top of the list was firmware blobs. They create big patches and have GPL issues. Interestingly, he said that the firmware issues, along with pressure from "a major distribution," are motivating the company to move its firmware back into the device. If the driver does not have to load firmware to make the device function, these issues go away. 

Since, as he put it, "a double-digit percentage" of QLogic's sales are for Linux systems, providing good support (and keeping the community happy) matters to the company. 

Another big problem was, again, maintaining drivers over multiple versions of the kernel. Failover is another one: QLogic does not want to handle failover issues in its drivers, but getting that capability into the higher levels of the kernel has taken some time. At least, it was pointed out, it is possible to work toward that sort of change; drivers written for proprietary systems must, by force, cope with the limitations of those systems. 

Then came James Ketrenos from Intel, who has been working with the IPW2100 and IPW2200 drivers. He noted that the IPW2100 Linux driver project began 12 months after the product was launched. The IPW2200 driver project started six months after the product came out. For the 2915ABG driver, the project started three months _before_ the product launch. This is the sort of trend that the kernel developers like to see. 

The biggest problem, according to James, is that customers only want to run Intel-certified drivers. But how can that come about when Intel does not control the software? A couple of possible approaches were mentioned: 

  * Keep everything in the mainline, but require vendor signoff on any patches which are proposed for merging. James knew that the kernel developers were not going to accept this one. 

  * Keep the driver out of the tree, and maintain control of it. 

The problem here, of course, is that the vendor is trying to keep control over an open source driver, but such control is contrary to the very idea of open source. It is simply not going to happen. Vendors need to understand that they will not have the ability to require certification for any drivers (and any patches) merged into the mainline kernel. The truth of the matter is that most customers don't want that anyway; they want a well-maintained driver which is present in the mainline. 

Nonetheless, Linus promised that, if Intel could get the top laptop manufacturers to support Linux (and only Linux), he would guarantee that nobody else could touch their particular driver. 

Beyond that, Intel has run into a common vendor problem. Vendors do not want to post code, even for review, until it has passed through the whole corporate quality control and legal gauntlet. By that time, however, it is far too late. Chances are that there will be a fundamental problem which will require substantial rewriting, and that will set the process back by months (at best). It is better to post code early, deal with the comments, and avoid expensive setbacks later in the process. To this end, the Intel developers are currently trying to get approval to post pre-certification patches to the netdev tree for testing and review. One can only wish them luck. 

The final speaker was Tim Bird, representing the Consumer Electronics Linux Forum. Vendors in this space also have version support issues, but of a slightly different variety: they tend to stick with old kernels for a long time. Many of them are still using 2.4. This situation has not been helped by the fact that support for the boards used in embedded systems tends to lag behind - though 2.6 has been better in that regard. 

Consumer electronics firms have different ideas of code quality. Much code which is considered good enough for a closed platform is not something one would want to post to the world. Embedded systems development tends not to emphasize generality and long-term maintainability; this industry shoves a product out the door and moves on. So much of its code is not suitable for contribution back to the community. 

In addition, these companies tend not to put their best developers on community work; those guys are locked in the basement trying to get the next product out. When a developer _does_ get the opportunity to contribute something, it tends to be in a two-week window before the next product crunch begins. As a result, embedded developers have a tendency to dump some (often terrible) code on a maintainer, then disappear again. These developers never get the chance to learn how to work with or earn trust within the community. To make things worse, consumer electronics developers tend to deal in lots of proprietary secret stuff, have little time for after-work coding, and lack confidence in their English language skills. 

So getting embedded systems developers to work more with the community is going to be a challenge. CELF is going to try to help by hiring a developer of its own; this person will, among other things, help to serve as a liaison between embedded developers and the community. 

Tim also allowed as to how a nice, stable kernel programming API would be a nice thing to have. He knew better than to push that issue in this crowd, however. 

James Bottomley, the leader of this session, summarized with a few points of his own. He stressed the need for involvement with the community from the start. The 2.6 development process makes stabilization harder; James seems to think that it is not working entirely well. Negative feedback from the community remains an issue; if we want to be an inclusive community, we need to treat people more nicely on the mailing lists. Finally, more educational outreach to manufacturers can only be a good thing. Corporations can learn to work with free software, given time and patience.
