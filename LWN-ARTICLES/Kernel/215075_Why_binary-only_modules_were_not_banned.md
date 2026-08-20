---
title: "Why binary-only modules were not banned"
url: https://lwn.net/Articles/215075/
date: "December 20, 2006"
category: Copyright issues
author: 
---

For a moment, it seemed like things could happen pretty quickly. Martin Bligh [suggested](<http://lwn.net/Articles/213974/>) that, rather than trying to nickel-and-dime binary modules to death, it would be more honest to just ban them outright. Andrew Morton [spoke out in favor](<http://lwn.net/Articles/213975/>) of the idea as long as a one-year warning was provided. Greg Kroah-Hartman [hacked up a patch](<https://lwn.net/Articles/215078/>) to insert the warning. And Linus, at the outset, restricted himself to [commenting on Greg's poetry](<https://lwn.net/Articles/215079/>). 

The tide turned just as quickly, however. Linus [spoke out against the change](<http://lwn.net/Articles/214149/>), and Greg [withdrew it](<http://lwn.net/Articles/214150/>). It would appear that binary-only modules will continue to be loadable into the kernel for the foreseeable future - though other hazards may await those who distribute them. 

The loading of proprietary modules was not banned for a few reasons, the first of which being that there is, in fact, nothing wrong with doing so. The GPL is quite clear in its statement that somebody who is in possession of GPL-licensed code can use it in any way they wish. If they want to combine their nice free kernel with a big, proprietary binary blob, they are fully within their rights to do so. So banning proprietary modules in the kernel source attacks the problem in the wrong place and attempts to forbid an activity which is allowed by the license. 

Even if the GPL could be interpreted as forbidding the loading of binary-only modules, there is the fair use issue to consider. As a community, we tend to be generally in favor of a broad interpretation of fair-use rights. But fair use cuts both ways. A number of people in the discussion warned against adopting the tactics favored by the entertainment industry and taking an overly broad view of what the law allows copyright owners to do. As Ben Collins [put it](<https://lwn.net/Articles/215082/>): 

The gradual changes to lock down kernel modules to a particular license(s) tends to mirror the slow lock down of content (music/movies) that people complain about so loudly. It's basically becoming DRM for code. 

The fact that some people were willing to discuss making use of the DMCA to make sure that nobody could patch a proprietary module ban out of the code tends to reinforce this view. Alan Cox [noted](<https://lwn.net/Articles/215083/>) that people tend to become that which they fight. Most people in the community would _probably_ agree that the entertainment industry is not something we wish to become; this realization has, arguably, done a lot to erode support for the idea of banning proprietary modules. 

What the GPL does cover is distribution; anybody who distributes something derived from GPL-licensed code must do so under the terms of the GPL. So it is the act of distributing proprietary modules which enters legally questionable territory. But, as [Linus points out](<https://lwn.net/Articles/215085/>), the fact that a module can be loaded into the kernel does not imply that the module is necessarily a derived work of the kernel. The determination of derived work status is a complicated business, and can often require a court to provide the definitive word. But banning all proprietary modules on the idea that they are all illegal derived works is a hard action to defend. 

The end result is that there will be no technical measures for the blocking of binary modules added to the kernel anytime soon. Unhappiness with these modules remains, however, as can be seen in Greg's message withdrawing the patch: 

It's just that I'm so damn tired of this whole thing. I'm tired of people thinking they have a right to violate my copyright all the time. I'm tired of people and companies somehow treating our license in ways that are blatantly wrong and feeling fine about it. Because we are a loose band of a lot of individuals, and not a company or legal entity, it seems to give companies the chutzpah to feel that they can get away with violating our license. 

It seems clear that the issue will not go away, even though this particular approach to addressing it has been rejected. The course which appears to be open to disgruntled kernel developers is legal action: if the distribution of a specific binary module can be shown to be a copyright violation, then the copyright owners have the right to go to court to put a stop to it. GPL enforcement efforts have, so far, tended to be successful. So it would not be surprising to see one or more developers decide to bring a suit against a binary module distributor in the next year or so. The discontent which is so visibly out there is unlikely to just fade away.
