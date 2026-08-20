---
title: On kernel mailing list behavior
url: https://lwn.net/Articles/559061/
date: "July 17, 2013"
category: "Development model-Developer conduct"
author: "By Jonathan Corbet July 17, 2013"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
July 17, 2013

As has been widely reported, the topic of conduct on kernel-related mailing lists has, itself, been the topic of a heated discussion on the linux-kernel mailing list. While numerous development communities have established codes of conduct over the years, the kernel has never followed suit. Might that situation be about to change? Your editor will attempt a factual description of the discussion, followed by some analysis. 

#### What was said

The setting was an extensive discussion on policies for the management of the stable kernel series and, in particular, the selection of patches for stable updates. It was an interesting discussion in its own right (which will be covered here separately), and it was generally polite. Even so, there came a point where Sarah Sharp [couldn't take it anymore](<https://lwn.net/Articles/559065/>): 

Seriously, guys? Is this what we need in order to get improve -stable? Linus Torvalds is advocating for physical intimidation and violence. Ingo Molnar and Linus are advocating for verbal abuse. 

Not *fucking* cool. Violence, whether it be physical intimidation, verbal threats or verbal abuse is not acceptable. Keep it professional on the mailing lists. 

For the record, she was responding to [this note](<https://lwn.net/Articles/559067/>) from Linus: 

Greg, the reason you get a lot of stable patches seems to be that you make it easy to act as a door-mat. Clearly at least some people say "I know this patch isn't important enough to send to Linus, but I know Greg will silently accept it after the fact, so I'll just wait and mark it for stable". 

You may need to learn to shout at people. 

Ingo's [contribution](<https://lwn.net/Articles/559068/>) was: 

So Greg, if you want it all to change, create some _real_ threat: be frank with contributors and sometimes swear a bit. That will cut your mailqueue in half, promise! 

Whether these messages constitute "advocating for physical intimidation and violence" or even "advocating for verbal abuse" will be left for the reader to decide. But Sarah's point was clearly not that these specific messages were out of line; she is concerned with the environment on the linux-kernel mailing list in general. She has since [taken the discussion to other forums](<https://plus.google.com/116960357493251979546/posts/KPLcrKLtcGo>) (with more examples) and, in general, seems intent on changing the nature of the community's discourse. 

Needless to say, responses on the list were mixed, though they were generally polite and restrained. A number of people, Linus included, pointed out that the number of personal attacks on the list is actually quite small, and that Linus tends to reserve his strongest language for high-level maintainers who (1) are able to take it, and (2) "should know better" than to do whatever it was that set Linus off. Opinions differ on whether that is a good thing. Jens Axboe [said](<https://plus.google.com/111643045511375507360/posts/cwaU6k5DprE>): 

I've been flamed plenty in the past, and it's been deserved (most of the time). Perhaps I have a thick skull and/or skin, but it doesn't really bother me. Or perhaps I'm just too much of an old kernel fart these days, so I grew accustomed to it. As long as I don't have to see Linus in his bathrobe, then that's enough "professionalism" for me. 

On the other hand, Neil Brown [echoed](<https://lwn.net/Articles/559074/>) the feelings of a number of participants who worry that the tone of the discussion tends to discourage people from joining the community: ""He is scolding people senior developers in front of newcomers. That is not likely to encourage people to want to become senior developers."" Being flamed can be hard on the recipient, but it can also affect the community by deterring other developers from participating. 

For his part, Linus has [made it clear](<https://lwn.net/Articles/559075/>) that he feels little need to change his tone on the list: 

The fact is, people need to know what my position on things are. And I can't just say "please don't do that", because people won't listen. I say "On the internet, nobody can hear you being subtle", and I mean it. 

And I definitely am not willing to string people along, either. I've had that happen too - not telling people clearly enough that I don't like their approach, they go on to re-architect something, and get really upset when I am then not willing to take their work. 

Sarah [responded](<https://lwn.net/Articles/559077/>) that one can be clear without being abusive; she also suggested that Linus use his power directly (by threatening not to pull patches from the offending maintainer) rather than using strong words. For what it's worth, Linus did [acknowledge](<https://lwn.net/Articles/559178/>), later in the discussion, that one of his more famous rants was ""Not my proudest moment."" 

Unsurprisingly, there were few concrete outcomes from the discussion (which is still in progress as of this writing). Sarah has [called for](<https://lwn.net/Articles/559078/>) the creation of a document (written by ""a trusted third party"") describing acceptable conduct in the kernel community. There will almost certainly be a Kernel Summit discussion on this topic; as Linus [pointed out](<https://lwn.net/Articles/559080/>), this kind of process-oriented discussion is the reason why the Kernel Summit exists in the first place. 

#### Some analysis

There are, it seems, some simple statements that should not be overly controversial in the context of a discussion like this. Most people prefer an environment where people are pleasant to one another to an environment where people are harsh or abusive. An abusive community can certainly deter some potential contributors from joining; consider, for example, whether OpenBSD might have more developers if its communications were more congenial. Various development communities have set out to improve the quality of their communications, sometimes with clear success. 

How do these thoughts apply in the kernel context? 

It is worth pointing out that this is not the first time people have expressed concerns about how the kernel community works; it was, for example, [a topic of discussion](<https://lwn.net/Articles/249104/>) at the 2007 Kernel Summit. Numerous developers have pushed for improvements in how kernel people communicate; these efforts have happened both publicly and in private. Even Linus has said, at times, that he wished the discussion on linux-kernel were more constructive. 

Your editor will assert that, in fact, the situation has improved considerably over the years. Much of that improvement is certainly due to the above-mentioned efforts. Abusive personalities have been confronted, managers have occasionally been contacted, trolls have been ignored, and more. The improvement is also certainly a result of changes in the kernel development community. We are as a whole older (and thus more restrained); the community is also much more widely paid to do its work, with the result that image-conscious companies have an incentive to step in when their developers go overboard. The tone is far more "professional," and true personal attacks are rare (though [examples](<http://permalink.gmane.org/gmane.linux.ports.arm.kernel/230497>) can certainly be found if one [looks](<https://lwn.net/Articles/540291/>)). 

Over the years, the kernel development community has continued to grow. One might argue that it would have grown much more rapidly with a different culture in its mailing lists, but that is hard to verify. It is true, though, that much of that growth has come from parts of the world where people are said to be especially sensitive to direct criticism. For all its troubles, the kernel community is still sufficiently approachable that over 3,000 people per year are able to get their work reviewed and merged. 

That said, the kernel is still viewed as one of the harshest communities in the free software world. It seems fairly clear that the tone of the discussion could bear some improvement, and that the current state of affairs repels some people who could otherwise be valuable contributors. So efforts like Sarah's to make things better should be welcomed; they deserve full consideration on the part of the community's leaders. But this kind of effort will be working against some constraints that make this kind of social engineering harder. 

One of them is that the kernel absolutely depends on the community's unwillingness to accept substandard code. The kernel has to work in a huge variety of settings for an unbelievable number of use cases. It must integrate the work of thousands of developers and grow rapidly while staying maintainable over the long term. It is a rare software project indeed that has attained the size of the kernel and sustained its rate of change without collapsing under its own weight. If we want to still have a viable kernel a decade from now, we must pay close attention to the code that we merge now. 

So it must be possible for developers to speak out against code that they see as being unsuitable for merging into the kernel. And the sad fact is that, sometimes, this message must be conveyed forcefully. Some developers are either unwilling to listen or they fail to receive the full message; as Rusty Russell [put it](<https://lwn.net/Articles/559099/>): 

You have to be harsh with code: People mistake politeness for uncertainty. Whenever I said 'I prefer if you XYZ' some proportion didn't realize I meant 'Don't argue unless you have new facts: do XYZ or go away.' This wastes my time, so I started being explicit. 

The size of the community, the fact that some developers are unwilling to toss aside code they have put a lot of time into, and pressure from employers can all lead to a refusal to hear the message and, as a consequence, the need to be explicit. Any attempt to make it harder for developers to express their thoughts on the code could damage the community and, more to the point, is almost certain to fail. 

That said, Rusty concluded the above message with this advice: ""But be gentle with people. You've already called their baby ugly."" There are certainly times when the community could be gentler with people without compromising on their code. That, of course, is exactly what people like Sarah are asking for. 

Whether a documented code of conduct would push things in that direction is hard to say, though. Simply obtaining a consensus on the contents of such a document is likely to be a difficult process, though the discussion itself could be helpful in its ability to produce counterexamples. But, even if such a document were to be created, it would run a real risk of languishing under `Documentation/` unheeded. Communities that have tried to establish codes of conduct have also typically included enforcement mechanisms in the mix. Groups like Fedora's "hall monitors" or Gentoo's "proctors" typically have the ability to ban users from lists and IRC channels when abuses are seen. Mozilla's [community participation guidelines](<https://www.mozilla.org/about/policies/participation.html>) describe a number of escalation mechanisms. It is not at all clear that the kernel is amenable to any such enforcement mechanism, and, indeed, Sarah does not call for one; instead, she suggests: 

Some people won't agree with everything in that document. The point is, they don't have to agree. They can read the document, figure out what the community expects, and figure out whether they can modify their behavior to match. If they are unwilling to change, they simply don't have to work with the developers who have signed it. 

It is far from clear, though, that a document calling for any sort of substantive change would acquire signatures from a critical mass of kernel developers, or that developers who are unwilling to sign the document would be willing (or able) to avoid dealings with those who have. 

So proponents of more polite discourse on linux-kernel are almost certainly left with tools like calling out undesirable behavior and leading by example — precisely the methods that have been applied thus far. Those methods have proved to be frustratingly slow at best, but, helped by the overall changes in the development community, they have proved effective. It was probably about time for another campaign for more civility to push the community subtly in the right direction. Previous efforts have managed to make things better without wrecking the community's ability to function efficiently; indeed, we have only gotten better at kernel development over time. With luck and some support from the community, we should see similar results this time.
