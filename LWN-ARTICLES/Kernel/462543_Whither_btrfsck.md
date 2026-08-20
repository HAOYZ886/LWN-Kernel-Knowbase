---
title: "Whither btrfsck?"
url: https://lwn.net/Articles/462543/
date: "October 11, 2011"
category: Btrfs
author: "By Jonathan Corbet October 11, 2011"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
October 11, 2011

The btrfs filesystem was merged into the mainline in January, 2009 for the 2.6.29 kernel release. Since then, development on the filesystem has accelerated to the point that many consider it ready for production use and some distributions are considering using it by default. The filesystem itself is nearly functionally complete and increasingly stable, but there is still one big hole: there is no working filesystem checker for Btrfs. As user frustration over the lack of this essential utility grows, an interesting question arises: is some software too dangerous to be released early? 

This tool (called "btrfsck") has been under development for some time, but, despite occasional hints to the contrary, it has never escaped from Chris Mason's laptop into the wild. This delay has had repercussions elsewhere; Fedora's plan to move to btrfs by default, for example, cannot go forward without a working filesystem checker. Most recently, Chris [said](<https://lwn.net/Articles/461660/>) that he hoped to be able to demonstrate the program at the upcoming [LinuxCon Europe](<https://events.linuxfoundation.org/events/linuxcon-europe>) event. That, however, was not enough for some vocal users who have started to let it be known that their patience has run out. Thus we've seen [accusations](<https://lwn.net/Articles/462549/>) that Oracle really intends to keep btrfs as a private, proprietary tool and [statements](<https://lwn.net/Articles/462554/>) that ""It's really time for Chris Mason to stop disgracing the open source community and tarnishing Oracle's name."" Those are strong words directed at somebody who has done a lot to create a next-generation filesystem for Linux. 

Your editor would like to be the first to say that both the open source community and Oracle benefit greatly from Chris's presence. The cynical might add that Oracle has delegated the task of "tarnishing its name" to employees who are more skilled in that area. That said, it is worth examining why btrfsck remains under wraps; had the tool been put out in the open - the way the filesystem itself was - chances are good that others would have helped with its development. One could argue that the failure to release btrfsck in any form has almost certainly retarded its development and, thus, the adoption of btrfs as a whole. 

[According to Chris](<https://lwn.net/Articles/462561/>), the early merging of btrfs was important for the creation of the filesystem's development community: 

Keep in mind that btrfs was released and ran for a long time while intentionally crashing when we ran out of space. This was a really important part of our development because we attracted a huge number of contributors, and some very brave users. 

But, he says, the filesystem checker ("fsck") is a bit different, and is not ready yet even for the braver users: 

For fsck, even the stuff I have here does have a way to go before it is at the level of an e2fsck or xfs_repair. But I do want to make sure that I'm surprised by any bugs before I send it out, and that's just not the case today. The release has been delayed because I've alternated between a few different ways of repairing, and because I got distracted by some important features in the kernel. 

Josef Bacik [expressed the fears](<https://lwn.net/Articles/462562/>) that keep btrfsck out of the community more clearly: 

Fsck has the potential to make any users problems worse, and given the increasing number of people putting production systems on btrfs with no backups the idea of releasing a unpolished and not fully tested fsck into the world is terrifying, and would likely cause long term "I heard that file system's fsck tool eats babies" sort of reputation. 

He went on to say ""Release early and release often is nice for web browsers and desktop environments, it's not so nice with things that could result in data loss"". This is a claim that raises some interesting questions, to say the least. 

One could start by questioning the wisdom of running a new filesystem like btrfs in production with no backups _and_ no working filesystem repair tool. How is it that releasing the filesystem itself is OK, but releasing the repair tool presents too much of a risk for users? How does that tool really differ from a web browser, especially given that the browser is exposed to all the net can throw at it and bugs can easily lead to exposure of users' credentials or the compromise of their systems? There is no shortage of software out there that can badly bite its users when things go wrong. 

That said, there are some unique aspects to the development of filesystem repair tools. They are invoked when things have already gone wrong, so the usual rules of how the filesystem should be structured are out the window. They must perform deep surgery on the filesystem structure to recover from corruptions that may be hard to anticipate and correct; one could paraphrase Tolstoy and say that happy filesystems are all alike, but every corrupted filesystem is unhappy in its own way. As the checker tries to cope with a messed-up filesystem, it works in an environment where any change it makes could turn a broken-but-recoverable filesystem into one that is a total loss. In summary, btrfsck will not be an easy tool to write; it is a job that is almost certainly best left to developers with a lot of filesystem experience and who understand btrfs to its core. That narrows the development pool to a rather small and select group. 

And, in the end, no responsible developer wants to release a tool which, in his or her opinion, could create misery for its users. Those users _will_ run btrfsck on their filesystems regardless of any blood-curdling warnings that it may put up first; if it proceeds to destroy their data, they will not blame themselves for their loss. If Chris does not yet believe that he can responsibly release btrfsck for wider use, it is not really our place to second-guess his reasoning or to tell him that he should release it anyway. Anybody who feels they cannot trust him to make that decision probably should not be running the filesystem he designed to begin with. 

Releasing software early and often is, in general, good practice for free software development; keeping code out of the public eye often does not benefit it in the long run. Perhaps btrfsck has been withheld for too long, but that is not our call to make. The need for the tool is clear - if nothing else, Oracle has decided to go with btrfs by default in the near future. There can be no doubt that this need is creating a fair amount of pressure. The LinuxCon demonstration may or may not happen, but btrfsck seems likely to make its much-delayed debut before too much longer.
