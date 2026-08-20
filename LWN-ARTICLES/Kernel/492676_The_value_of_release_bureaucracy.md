---
title: The value of release bureaucracy
url: https://lwn.net/Articles/492676/
date: "April 17, 2012"
category: "Development model-Stable tree"
author: "By Jonathan Corbet April 17, 2012"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
April 17, 2012

Those who read the linux-kernel mailing list will, over time, develop an ability to recognize certain types of discussions by the pattern of the thread. One of those types must certainly be "lone participant persistently argues that the entire kernel community is doing it wrong." Such discussions can often be a good source for inflammatory quotes, but they often lack much in the way of redeeming value otherwise. [This thread](<https://lwn.net/Articles/492677/>) on the rules for merging patches into stable releases would seem to fit the pattern, but a couple of the points discussed there may be worthy of highlighting. If nothing else, perhaps a repeat of that discussion can be avoided in the future. 

[This patch](<https://git.kernel.org/?p=linux/kernel/git/torvalds/linux-2.6.git;a=commitdiff;h=c1afdaff90538ef085b756454f12b29575411214>) to the ath9k wireless driver was meant to fix a simple bug; it was merged for 3.4. Since it was a bug fix, it was duly marked for the stable updates and shipped in 3.3.1. It turns out to not have been such a good idea, though; some 3.3.1 users have reported that the "fix" can break the driver and sometimes make the system as a whole unusable. That is not the sort of improvement that stable kernel users are generally hoping for. Naturally, they hoped to receive a fix to the fix as soon as possible. 

When the 3.3.2 update went into the review process without a revert for the offending commit, some users asked why. The answer was simple: the rules for the stable tree do not allow the inclusion of any patch that has not already been merged, in some form, into the mainline. Since this particular fix had not yet made it to Linus (it was still in the wireless tree), Greg Kroah-Hartman, the stable kernel maintainer, declined to take it for the 3.3.2 cycle. And that is where the trouble started. 

Our lone participant (Felipe Contreras) denounced this decision as a triumph of bureaucratic rules over the need to actually deliver working kernels to users. Beyond that, he said, since reverting the broken patch simply restored the relevant code to its form in the 3.3 release, the code was, in effect, already upstream. Accepting the revert, he said, would have the same effect as dropping the bad patch before 3.3.1 was released. In this view, refusing to accept the fix made little sense. 

Several kernel developers tried to convince him otherwise using arguments based on the experience gained from many years of kernel maintenance. They do not appear to have succeeded. But they did clearly express a couple of points that are worth repeating; even if one does not agree with them, they explain why certain things are done the way they are. 

The first of those was that experience has proved, all too many times, that fixes applied only to stable kernel releases can easily go astray before getting into the mainline. So problems that get fixed in a stable release may not be fixed in current development kernels - which are the base for future stable kernels. So stable kernel users may see a problem addressed, only to have it reappear when they upgrade to a new stable series. Needless to say, that, too, is not the experience stable kernel users are after. On the other hand, people who like to search for security holes can learn a lot by watching for fixes that don't make it into the mainline. 

It is true that dropped patches used to be a far bigger problem than they are now. A patch applied to, say, a 2.2 release had no obvious path into the 2.3 development series; such patches often fell on the floor despite the efforts of developers who were specifically trying to prevent such occurrences. In the current development model, a fix that makes it into a subsystem maintainer's tree will almost certainly get all the way into the mainline. But, even now, it's not all that rare for a patch to get stranded in a forgotten repository branch somewhere. When the system is handling tens of thousands of patches every year, the occasional misrouted patch is just not a surprise. 

The simple truth of the matter is that many bugs are found by stable kernel users; there are more of them and they try to use their kernels for real work. As this thread has shown, those users also tend to complain if the specific fixes they need don't get into stable releases; they form an effective monitoring solution that ensures that fixes are applied. The "mainline first" rule takes advantage of this network of users to ensure that fixes are applied for the long term and not just for a specific stable series. At the cost of (occasionally) making users wait a short while for a fix, it ensures that they will not need the same fix again in the future and helps to make the kernel less buggy in general. 

Developers also took strong exception to the claim that applying a revert is the same as having never applied the incorrect fix in the first place. That can almost never be strictly true, of course; the rest of the kernel will have changed between the fix and the revert, so the end product differs from the initial state and may misbehave in new and interesting ways. But the real issue is that both the fix and the revert contain information beyond the code changes: they document a bug and why a specific attempt to fix that bug failed. The next developer who tries to fix the bug, or who makes other changes to the same code, will have more information to work with and, hopefully, will be able to do a better job. The "mainline first" rule helps to ensure that this information is complete and that is it preserved in the long term. 

In other words, some real thought has gone into the creation of the stable kernel rules. The kernel development community, at this point, has accumulated a great deal of experience that will not be pushed aside lightly. So the stable kernel rules are unlikely to be relaxed anytime soon. The one-sided nature of the discussion suggests that most developers understand all of this. That probably won't be enough to avoid the need to discuss it all again sometime in the near future, though.
