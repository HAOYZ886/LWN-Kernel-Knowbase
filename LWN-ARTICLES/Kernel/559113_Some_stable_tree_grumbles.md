---
title: Some stable tree grumbles
url: https://lwn.net/Articles/559113/
date: "July 17, 2013"
category: "Development model-Stable tree; Sucker tree"
author: "By Jonathan Corbet July 17, 2013"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
July 17, 2013

In the dim and distant past (March 2005), the kernel developers were having [a wide-ranging discussion](<https://lwn.net/Articles/126763/>) about various perceived problems with the kernel development process, one of which was the inability to get fixes for stable kernel releases out to users. Linus suggested that a separate tree for fixes could be maintained if a suitable "sucker" could be found to manage it, but, he predicted, said sucker would ""go crazy in a couple of weeks"" and quit. As it turns out, Linus had not counted on just how stubborn Greg Kroah-Hartman can be; Greg (along with Chris Wright at the time) stepped forward and volunteered to maintain this tree, starting with the release of [2.6.11.1](<https://lwn.net/Articles/126324/>). Greg has continued to maintain the stable trees ever since. Recently, though, he has expressed some frustrations about how the process is working. 

In particular, the [announcement](<https://lwn.net/Articles/559122/>) of the review stage for the 3.10.1 release included a strongly-worded complaint about how subsystem maintainers are managing patches for the stable tree. He called out two behaviors that he would like to see changed: 

  * Some patches are being marked for stable releases that clearly do not belong there. Cosmetic changes to debug messages were called out as an example of this type of problem. 

  * More importantly: a lot of the patches marked as being for the stable tree go into the mainline during the merge window. In many cases, that means that the subsystem maintainer held onto the patches for some time — months, perhaps — rather than pushing them to Linus for a later -rc release. If the patches are important enough to go into the stable tree, Greg asked, why are they not going to Linus immediately? 

Starting with the second complaint above, the explanation appears to be relatively straightforward: getting Greg to accept changes for the stable tree is rather easier than getting Linus to accept them outside of the merge window. In theory, the rules for inclusion into the stable tree are the same as for getting patches into the mainline late in the cycle: the patches in question must fix some sort of "critical" problem. In practice, Linus and Greg are at least perceived to interpret the rules differently. So developers, perhaps unwilling to risk provoking an outburst from Linus, will simply hold fixes until the next merge window comes around. As James Bottomley [put it](<https://lwn.net/Articles/559137/>): 

You mean we delay fixes to the merge window (tagged for stable) because we can't get them into Linus' tree at -rc5 on? Guilty ... that's because the friction for getting stuff in rises. It's a big fight to get something marginal in after -rc5 ... it's easy to silently tag it for stable. 

Greg's plan for improving things involves watching linux-next starting around the -rc4 mainline release. If patches marked for the stable series start appearing in linux-next, he'll ask the maintainers why those patches have not yet found their way to Linus. Some of those patches may well find themselves refused entry into the stable tree if they only show up in the mainline during the merge window. 

The topic of fully inappropriate patches, while the lesser part of Greg's complaint, became the larger part of the discussion. There are, it seems, any number of reasons for patches to be directed at the stable tree even if they are not stable material. At one extreme, [Ben Herrenschmidt's description](<https://lwn.net/Articles/559134/>) of how the need to get code into enterprise kernels drives the development process is well worth reading. For most other cases, though, the causes are probably more straightforward. 

For years, people worried that important fixes were being overlooked and not getting into the stable updates; that led to pressure on developers to mark appropriate patches for the stable tree. This campaign has been quite successful, to the point that now, often, developers add a stable tag to a patch that fixes a bug as a matter of reflex. Subsystem maintainers are supposed to review such tags as part of their review of the patch as a whole, but that review may not always happen — or those maintainers may agree that a patch should go into the stable tree, even if it doesn't adhere to the rules. And sometimes subsystem maintainers can't remove the tag even if they want to. All this led James to [propose](<https://lwn.net/Articles/559135/>) doing away with the stable tag altogether: 

The real root cause of the problem is that the cc: stable tag can't be stripped once it's in the tree, so maintainers only get to police things they put in the tree. Stuff they pull from others is already tagged and that tag can't be changed. This effectively pushes the problem out to the lowest (and possibly more inexperienced) leaves of the Maintainer tree. 

James (along with others) proposes that putting a patch into the stable tree should require an explicit action on the subsystem maintainer's part. But Greg [dislikes](<https://lwn.net/Articles/559136/>) this idea, noting that maintainers are already far too busy. The whole point of the stable tree process is to make things as easy for everybody else as possible; adding work for maintainers would endanger the success of the whole exercise. That is especially true, he said, because some developers might encounter resistance from their employers: 

And that annoys the hell out of some Linux companies who feel that the stable kernels compete with them. So people working for those companies might not get as much help with doing any additional work for stable kernel releases (this is not just idle gossip, I've heard it directly from management's mouths.) 

Another proponent of explicit maintainer involvement is Jiri Kosina, who, in his work with SUSE's kernels, has [encountered](<https://lwn.net/Articles/559138/>) a few problems with stable kernels. While the stable tree is highly valuable to him, some of the patches in it cause regressions, some are just useless, and, for some, there is no real indication of why the patches are in the stable tree in the first place. Forcing maintainers to explicitly nominate and justify patches for the stable tree would, he said, address all three types of problem. 

The first type — patches that introduce bugs of their own — will probably never be eliminated entirely; that is just how software development works. Everybody in the discussion has acknowledged that, once a buggy fix is identified, Greg quickly makes a stable release with that patch removed, so regressions tend not to stay around for long. Useless patches include those that are backported to kernels that predate the original bug; this problem could be addressed by placing more information in the changelog describing when the bug was introduced. The final type of problem raised by Jiri — mysterious patches — turned out to be security fixes. Jiri (and others) would like security fixes marked as such in the changelog, but that [is unlikely to happen](<https://lwn.net/Articles/559201/>); instead, more effort is being made to notify distributors of security fixes via private channels. 

In other words, while changes are likely to be made, they will not be fundamental in nature. Greg is likely to become fussier about the patches he accepts for the stable tree. Chances are, though, that he will never be as hard to please as Linus in this regard. In the end, the consumers of the stable tree — distributors and users both — _want_ fixes to be included there. The stable kernel series is one of the biggest successes of the kernel development process; any changes to how they are created are likely to be relatively small and subtle. For most of us, the fixes will continue to flow as usual.
