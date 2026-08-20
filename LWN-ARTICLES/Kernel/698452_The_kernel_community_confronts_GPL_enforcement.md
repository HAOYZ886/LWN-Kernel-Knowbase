---
title: The kernel community confronts GPL enforcement
url: https://lwn.net/Articles/698452/
date: "August 31, 2016"
category: Copyright issues
author: "By Jonathan Corbet August 31, 2016"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
August 31, 2016

Some of the most important discussions associated with the annual Kernel Summit do not happen at the event itself; instead, they unfold prior to the summit on the [planning mailing list](<https://lists.linuxfoundation.org/mailman/listinfo/ksummit-discuss>). There is value in learning what developers feel needs to be talked about and, often, important issues can be resolved before the summit itself takes place. That list has just hosted (indeed, is still hosting as of this writing) a voluminous discussion on license enforcement that was described by some participants as being "pointless" or worse. But that discussion has served a valuable purpose: it has brought to the light a debate that has long festered under the surface, and it has clarified where some of the real disagreements lie. 

It all started when Karen Sandler, the executive director of the Software Freedom Conservancy (SFC), [proposed](<https://lists.linuxfoundation.org/pipermail/ksummit-discuss/2016-August/003542.html>) a session on "GPL defense issues." Interest in these issues is growing, she said, and it would be a good time to get the kernel community together for the purposes of information sharing and determining what community consensus exists, if any, on enforcement issues. It quickly became clear that some real differences of opinion exist though, in truth, the differences of opinion within the community may not be as large as they seem. Rather than attempt to cover the entire thread, this article will try to extract some of the most significant points from it. 

#### When to sue

Many emails were expended on the question of when — if ever — the GPL should be enforced in the courts. To many, this is the core point, and many think that lawsuits are almost never in the community's interest; instead, compliance issues should be resolved via constructive engagement with the companies involved. Greg Kroah-Hartman [made that point clearly](<https://lists.linuxfoundation.org/pipermail/ksummit-discuss/2016-August/003555.html>) when he said ""I do [want companies to comply], but I don't ever think that suing them is the right way to do it, given that we have been _very_ successful so far without having to do that."" Linus Torvalds [stated his agreement with this position](<https://lists.linuxfoundation.org/pipermail/ksummit-discuss/2016-August/003580.html>) (in classic Linus style) as well. 

There are, these developers said, plenty of reasons to avoid taking companies to court, starting with the fact that the legal system is nondeterministic at best and one never knows what the end result will be. The [recent outcome in the VMware case](<https://lwn.net/Articles/696936/>) was given as an example here; some see it as having made it harder to pursue enforcement actions in the future — though others disagree strongly and expect the forthcoming appeal to change the situation. But nobody was willing to argue that one can go to court and be certain of the outcome, and some fear the consequences of a severely adverse ruling. 

The stronger objection to legal action, though, it that it forces the targeted company into a defensive mode, isolates developers who support Linux internally, and, likely as not, estranges the company from the development community for a long time. Many developers said, over the course of the discussion, that it is far better to get companies to change their ways through engagement with their developers; the process may take years but, when it ends well, those companies join the community and become enthusiastic contributors. This process is said to have worked many times over the years; some of the kernel's largest contributors were once on the list of GPL infringers. 

The BusyBox lawsuits were cited as an example of how legal action can go wrong. The prevailing (though not universal) opinion seems to be that those suits led to the release of little, if any useful code while driving many companies out of our community and killing the BusyBox project itself. That is the sort of experience that lawsuit-averse kernel developers hope to avoid; as Linus put it: ""Lawsuits destroy community. They destroy trust. They would destroy all the goodwill we've built up over the years by being nice."" 

Implicit in that argument is the claim that license compliance is not a big problem; the current approach is working well and should not be changed. But it is clear that not all developers think that all is well, and there is certainly a contingent that is unwilling to forswear legal action — an action that strikes many as simply giving in to the infringers. As David Woodhouse [said](<https://lists.linuxfoundation.org/pipermail/ksummit-discuss/2016-August/003628.html>): ""Without the option of a lawsuit, there *is* no negotiation. The GPL has certain requirements and they are backed up by law. If you don't have that, you might as well have a BSD licence."" Many participants in the discussion echoed the thought that reticence to enforce the GPL will lead to its demise in favor of _de facto_ permissive licensing. 

Jeremy Allison (an SFC board member), speaking from his experience at the Samba project, [argued](<https://lists.linuxfoundation.org/pipermail/ksummit-discuss/2016-August/003650.html>) that lawsuits should remain an option, saying that, despite the fact that the project has never sued an infringing company, the threat of enforcement is the only thing that makes companies listen to him sometimes. He also [said](<https://lists.linuxfoundation.org/pipermail/ksummit-discuss/2016-August/003627.html>) that a lawsuit need not necessarily drive a company out of the community; he gave Microsoft, which lost a $1 billion judgment in a suit involving Samba, as an example; despite having been sued successfully, Microsoft is now a significant contributor to Samba. 

Supporters of legal action pointed out that, contrary to claims that "we have never had to do that", there _have_ been lawsuits in the past, and the results have not been as bad as some seem to fear. Harald Welte, who has probably done more GPL enforcement in the courts than anybody else, [wrote about his experience](<https://lists.linuxfoundation.org/pipermail/ksummit-discuss/2016-August/003708.html>). He still strongly believes that using the courts can be the right thing to do, and that it need not necessarily push companies out of the community: 

The point here is: Legal threats (or sometimes lawsuits) are wake-up calls to companies that _want_ to do the right thing, but who simply haven't been aware of it, or who haven't given it the right attention/priority before. They are *not* upset about enforcement being brought against them. 

Companies, he said, appreciate clarity on what compliance with the license actually means, and the only way to get that clarity is through opinions from the courts. He described using the GPL without enforcement as ""useless"", saying that the BSD license would be a better choice if there is no wish to enforce the terms of the GPL. 

In truth, nobody is willing to forswear legal action entirely; even Linus [said](<https://lists.linuxfoundation.org/pipermail/ksummit-discuss/2016-August/003695.html>) that there are times when it makes sense. But the example he gave — IBM's use of GPL-infringement charges in the SCO suit — demonstrates that he sees legal action as a move to be made only in extreme situations. In general, one might say that there is [a consensus in the community](<https://lists.linuxfoundation.org/pipermail/ksummit-discuss/2016-August/003625.html>) that lawsuits should only be used as the last resort. But there are significant disagreements over when the last resort has been reached. 

#### What is the objective?

Another interesting divide that came to light concerned what the purpose of the GPL and the goal of compliance is. Linus [let it be known](<https://lists.linuxfoundation.org/pipermail/ksummit-discuss/2016-August/003582.html>) that he is primarily concerned with maintaining the flow of code contributions: 

What I care about is getting code contributions back. That's kind of the whole *point* of the GPLv2. Not the legalese. Growing the source code base by having participation in the project. 

If the goal is to increase contributions, then anything that might alienate contributors is to be avoided. But bringing in the largest amount of code is not the primary concern for everybody involved; some are more focused on gaining access to code that vendors have distributed. Matthew Garrett [responded](<https://lists.linuxfoundation.org/pipermail/ksummit-discuss/2016-August/003583.html>) to Linus, saying: 

That's what you care about. That's not what your users care about. They care about code *availability*, not contribution. They don't care whether their vendor participates upstream. They just care about being able to fix their shitty broken piece of hardware when the vendor won't ship updates. 

A non-confrontational approach can work, he [said](<https://lists.linuxfoundation.org/pipermail/ksummit-discuss/2016-August/003585.html>), when the objective is ""4 more enterprise clustering filesystems"". But if, instead, one wants the next generation of developers to be able to hack on their devices, then manufacturers have to be convinced to ship source for those devices. Projects like OpenWrt exist as the result of previous enforcement actions, he said; if we want to see similar projects coalesce around today's devices, we need to be prepared to enforce the license and get vendors to provide the source. 

Linus [feels](<https://lists.linuxfoundation.org/pipermail/ksummit-discuss/2016-August/003603.html>), though, that OpenWrt has been helped far more by the success of the relaxed "open source" approach, which has focused on producing more and better code, over the GPL "hardliners" who are focused on software freedom. The latter approach, he claimed, has had the effect of driving developers and companies away from the GPL and has been counterproductive overall. 

#### Who is trusted to make the decision?

A fair amount of energy went into the question of whether the SFC can be trusted as an agent of enforcement for the kernel. Some developers (notably but not exclusively Linus) worry that the SFC is pursuing its own goals and that the kernel is not at the top of its list of priorities. SFC members, and Bradley Kuhn in particular, have made enough comments about needing to litigate the GPL in many courts to obtain precedents, defending the GPL as a moral necessity, and seeing the kernel as the final GPL battleground to make a number of people nervous. So, for example, Ted Ts'o [said](<https://lists.linuxfoundation.org/pipermail/ksummit-discuss/2016-August/003784.html>): ""I've asked Bradley point blank whether the GPL or the long-term health of Linux development was more important to him, and he very candidly said 'the GPL'."" Greg [put it this way](<https://lists.linuxfoundation.org/pipermail/ksummit-discuss/2016-August/003643.html>): 

You value the GPL over Linux, and I value Linux over the GPL. You are willing to risk Linux in order to try to validate the GPL in some manner. I am not willing to risk Linux for anything as foolish as that. 

For his part, Bradley [did not deny](<https://lists.linuxfoundation.org/pipermail/ksummit-discuss/2016-August/003648.html>) that assertion, but qualified it somewhat: 

You said that you "care more about Linux than the GPL". I would probably agree with that. But, I do care about software freedom generally much more than I care about Linux *or* the GPL. I care about Linux because it's the only kernel in the world that brings software freedom to lots of users. 

Bradley and the SFC had many defenders in the discussion, who said that the SFC should be judged by its actions rather than by what Bradley has said. They point out that, rather than being a litigious group, the SFC has only been involved in two lawsuits ever. They said that the SFC is willing to take on the unpleasant task of talking to management and legal departments about compliance issues — something that developers are generally unwilling to do. And, as Jeremy [emphasized](<https://lists.linuxfoundation.org/pipermail/ksummit-discuss/2016-August/003631.html>), the SFC is not pursuing its own agenda, but that of the developers it represents: 

The other thing to remember is Bradley isn't the one making the decisions here. It's the developers - *ALWAYS* the developers. Bradley and the Conservancy staff can give advice, but they do not do *anything* without a direct mandate from the copyright holders. 

Such testimonials notwithstanding, it is clear that a number of developers feel that the SFC's objectives do not necessarily line up with their own. Getting over that trust barrier could be hard for the organization to do. Karen has [said](<https://lists.linuxfoundation.org/pipermail/ksummit-discuss/2016-August/003742.html>) that the SFC will be having meetings with developers over the coming six months in order to answer questions and get feedback on its enforcement activities. It will be interesting to see what sort of changes happen — both within and outside of the SFC — as the result of these meetings. 

The current discussion on the list has slowed, which is undoubtedly a relief to everybody involved. There may not have been much that was resolved, but there should, at a minimum, be a better mutual understanding of the issues and concerns involved with GPL enforcement. The area is complex and full of risks — risks that are associated with both action and inaction. Figuring out what the community wants to do about GPL infringement will, if it is possible at all, require more discussions like this one. The prospect may be painful, but the possibility of frustrated developers acting rashly on their own could be even more so.
