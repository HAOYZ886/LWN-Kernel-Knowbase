---
title: "KS2007: Kernel quality"
url: https://lwn.net/Articles/248388/
date: "September 6, 2007"
category: "Development model-Kernel quality"
author: "By Jonathan Corbet September 6, 2007 LWN.net Kernel Summit 2007 coverage"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
September 6, 2007

* * *

LWN.net [Kernel Summit 2007](<https://lwn.net/Articles/KernelSummit2007/>) coverage 

The quality of Linux kernel releases is of interest to every developer. Few people are better placed to discuss this topic than Andrew Morton, who has put years of his life into improving how kernels are created. In the session he led on this topic, Andrew was unable to say whether he thought our kernel releases were getting better or worse. But he had no doubt that we could be doing a better job than we are now. 

For some time, Andrew has wanted a kernel bugmaster to help him in tracking and fixing problems. That person has materialized in the form of Natalie Protasevich. Andrew says he had hoped for "a nasty person" who would hang out on the mailing lists and beat up on developers who do not fix bugs; Natalie is not that person. But she is doing a good job of managing the bug database and getting reports to the relevant people. 

There are currently 1500 open bugs in the kernel bugzilla; Natalie is cleaning them up and passing them on to developers. When those developers respond, the results appear to be good. But not very many developers are doing that. 

We have a big problem in that a whole lot of bug reports are getting lost. They get reported to subsystem mailing lists and subsequently forgotten about. Bug reporters can go away for a number of reasons, many of which have little to do with the bug being fixed. In general, lost bugs are not necessarily fixed bugs. 

Andrew asserts that bug reports must be responded to quickly or the chances of getting that bug fixed drop tremendously. He made productive use of his flight to Cambridge by digging through a few thousand linux-kernel messages; he found some 50 bug reports which had not been responded to in any way. There were reports which were relevant to people who had found the time to participate fully in the recent BSD licensing flame war, but who were unable to get around to dealing with bugs. Andrew says that he is getting tired of nagging people; in general, this behavior does not seem particularly mature. 

Linus noted that many people no longer read the linux-kernel mailing list. The traffic has reached a level where it is simply overwhelming. It was suggested that perhaps a separate list for bug reports is needed. 

Dave Jones reiterated his statement that developers are not responding to bug reports. We are, as a result, losing bug reporters. Andrew brought up the old idea of doing an occasional bugfix-only release. Such a release would not just avoid the addition of new features; developers would be expected to spend time actively fixing bugs. Perhaps this would send a message that the developers have gotten serious about bug fixing and would inspire long-frustrated users to start reporting bugs again. 

Ted Ts'o said that once upon a time, ten years ago, we cared about our bug-free kernel and would help users with problems. It is kind of sad that we no longer have that attitude. In response it was pointed out that the volume of users (and their bug reports) has increased greatly, and that the level of technical knowledge held by our users is now much lower. As Alan Cox put it, ten years ago every user had a screwdriver and most of them did not have the case on their computer. 

Alan also claimed that the larger number of users could also be helpful in finding bugs, but that we are not making use of their reports. With adequate information and a bit of statistics, it could become much easier to identify combinations of hardware that lead to problems. To that end, better bug reporting tools would be most helpful. And, in particular, having a single tool (or at least a tool by a single name) supported by all distributors would be a good thing. If developers could ask users to run this tool to generate a comprehensive picture of the afflicted system, they would be better positioned to track down the problems. 

Andrew's response is that the discussion had taken a wrong turn. There is no point, he says, in having better tools if developers cannot be bothered to respond to bugs in the first place. He then moved on to the review problem. Code review, he says, has a large multiplier effect. One hour of serious code review can help to get a 100-hour patch into the kernel, and it can help developers to make better patches in the future. But we suffer from a shortage of reviewers. 

One problem is that code can quickly disappear into git repositories, from which it speeds into the mainline without much (or any) review. In response to previous pleas, developers have been posting code to the mailing lists more often, but these postings are often late, just before the code heads toward the mainline. Quite a bit of code is going into subsystem trees during the merge window, ensuring that the review period is quite brief. In response it was stated that subsystem maintainers should not accept patches during the merge window. Some maintainers already enforce such a policy, but others do not. 

One idea which was discussed was the creation of a pre-merge tree which contains all of the patches expected to go in during the next merge window. It would be much like -mm, but with the full set of queued patches and nothing which is not planned for the next merge window. Andrew claimed, once again, that the wrong problem was being discussed. Why talk about getting more patches posted when the patches which are sent out now are not being reviewed? 

Andrew's proposal is that patches, to be accepted in the mainline, must carry a new `Reviewed-by:` tag identifying who has looked them over. The patch information should also include information like pointers to the discussions with the reviewer so that the depth of the review can be judged; a Reviewed-by tag from a reviewer who is mostly concerned with white space might not have a whole lot of value. If nobody reviews patches, the system will clog up until developers _have_ to do some reviewing just to get things moving again. 

The idea was well received, though Linus expressed a concern that most patches, being in general quite small, will never really get serious review. A patch which looks trivial is hard to give a lot of attention to. But some of those patches will certainly carry bugs. So stronger review requirements will, in a world inhabited by humans, still not succeed in catching all of the silly problems. This point was generally understood, but there is still interest in trying out this system in an informal way. So expect to start seeing Reviewed-by tags on patches before too long.
