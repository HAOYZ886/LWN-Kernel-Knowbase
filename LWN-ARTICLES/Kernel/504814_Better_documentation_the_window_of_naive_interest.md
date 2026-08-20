---
title: "Better documentation: the window of naive interest"
url: https://lwn.net/Articles/504814/
date: "July 3, 2012"
category: Documentation
author: "July 3, 2012 This article was contributed by Neil Brown"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

July 3, 2012

This article was contributed by Neil Brown

Sometimes a casual comment can capture your imagination and not let go until you do something with it. So it was for me with a comment made by Heikki Orsila on [some observations](<https://plus.google.com/u/0/111049168280159033135/posts/4njRkH5jCa5>) that Greg Kroah-Hartman made about documentation in the Linux kernel tree: 

Greg, the documentation is very bad. 

The specific documentation that he or Greg were thinking of may well be very different from the specific documentation that my thoughts turned to, but as both a producer and a consumer of some parts of `linux/Documentation` I can at least agree that some of it isn't very good. 

Heikki continued: ""Linux is badly documented, but so what?"" and often that would have been the end of it. But we are an open development community where putting up with mediocrity is neither necessary nor encouraged. If things are broken then there is always the possibility of fixing them if only we know how. How, then, can we fix the documentation? 

"Documentation" is a broad category and I would like to start by narrowing our focus a little and excluding reference documentation from consideration. By this I mean documents used by a person knowledgeable in the subject who needs to clarify some detail such as the arguments to some function or the required ordering between two locks. For these details the source code is by far the best resource - as it cannot get out-of-date - and, when the code itself is not sufficient, placing the documentation in the source code will provide the greatest likelihood of it being found, read, and kept up-to-date. It doesn't really belong in a separate `Documentation` directory. 

The class of documentation that is of interest is documentation for the new developer, not necessarily new to development but new to a particular project or subsystem. Such a developer combines a lack of knowledge with a genuine interest and this is a combination that is not stable: if one component does not disappear soon, the other is likely to. The task of good documentation is to ensure the lack of knowledge disappears before the interest. 

I was exposed to this instability when trying to understand some details of power management in Linux. The documentation simply didn't help and I had to look elsewhere. However when I went back to assess the documentation while preparing for this article I discovered that it wasn't as bad as I remembered. I now had enough experience that it all made sense. The paucity of the documentation was now only in my memory and I couldn't be sure I had given that documentation a fair trial. The temptation to just move on might have won had Heikki Orslia's observation not encouraged me on. 

To understand what makes good documentation we need to mine the experiences from that short window of naive interest to find out what works and what doesn't. A question that seems most suited for digging is "What were you looking for that you didn't find". I'm sure my kind reader will have their own answers to offer, but here are three that I have found on my travels. 

#### Wire-frame outlines

When I first went to the Linux power management documentation I was after a "big picture" understanding. I wanted more detail than "this code manages power" but not quite "these are the entry points that a driver must provide". I wanted to know what the important parts were and, significantly, how they connected together and impacted each other. I picture this as a collection of key concepts together with the linkage between them. These are nodes and edges in a graph, entities and relationships, or for the more spatially oriented, vertices and edges of a wire-frame polyhedron. This gives the shape of the project without getting bogged down in details. 

For me it is vital to have this framework first as I can only take in and retain new details if I have something to attach them to and a place to attach them. Without it I'll either attach new ideas to the wrong place, or forget them completely - which is probably the safer of the two. 

The image of a wire-frame is a little misleading as it presents all vertices as of equal value and this is rarely the case. Some concepts are bigger and should be named and described first. Others can come later. So maybe a ball-and-stick model might be a better picture, with big and small balls, joined by thin and fat sticks. 

In the case of Linux power management, one key concept that gives shape to the whole is the number of multiplicities: there are multiple sequencing states when moving away from or towards full functionality, multiple power saving approaches such as runtime, suspend, and hibernate, and many multiple different sorts of devices that need to fit into the frameworks. Another concept, already hinted at but often recurring, is that there is generally one "fully functional" state but several "low power" states, where moving between two low power states involves returning to fully-functional and then reducing power a different way. 

#### Why, not what.

"Swap over NFS" is a set of functionality that some people find valuable, but is not at all straightforward to implement. There is a need to avoid deadlocks in memory management, and to do so without slowing down either the networking code or the memory allocation code, both of which are quite performance sensitive. There is [a set of patches](<https://lwn.net/Articles/504262/>) which provides this functionality but getting it ready for mainline inclusion has been a slow process. 

Andrew Morton was recently good enough to provide some review of these patches and, while reading the commit-log entries and code comments is a little different from seeking out more coherent documentation, it does provide a good window into the thoughts of someone who, while generally knowledgeable, is both new to the project specifics and still interested. It can thus answer the question "what were you looking for that you didn't find?". 

One observation that he made repeatedly is most clearly [embodied](<https://lkml.org/lkml/2012/5/1/307>) in 

The comment should explain "why", not "what". Particularly when the "what" was bleedin obvious ;) 

or more humorously in: ""`s/"what"/"why"/ !`"". 

Documenting _what_ a function does is very important in closed-source projects, but less so in open source where the code can be directly read. Of course if the code is long and complex it might be easier to read some documentation, however the effort of writing the documentation might be better spent in breaking up the code and making it more readable. 

Documenting _why_ is much more valuable, whether it is "why do it this way" or "why even do this at all". The "why" of a project is rarely explicit in just one place of the code. Rather it permeates throughout and can touch various fragments in different ways. Sometimes the "why" is not technical at all but is historical, cultural, or simply subjective. In these cases it really cannot be extracted by reading the code and must be documented, or lost. 

Were I to properly document the Linux "md" driver, for which I get occasional requests, I would need to explain its relationship with "dm" - for it isn't only internal edges of our wire-frame that are interesting, but also external edges. The "why"s here are mostly historical accident, though there would be value in observing that "md" focuses on reliability through redundancy, while "dm" focuses more on flexibility by hiding all the other restrictions imposed by storage hardware. This, I think, gives the "why" for continuing to have two separate frameworks, even if it isn't a strong technical justification. 

To continue with the analogy of the wire-frame model, if the concepts and relationships provide the shape of the model, then the "Why"s provide the fabric that they give shape to. They are the substance that gives purpose and the force that gives direction. They may not always be visible, especially once we put some skin on our model, but understanding them is key to understanding the whole. 

#### Examples, examples, examples.

One of the documents that I maintain is the set of manual pages for [mdadm](<http://linux.die.net/man/8/mdadm>). I recall some years ago being challenged that there weren't enough examples in that documentation. At the time I didn't really know what to do with the challenge as, after all, there was an "Examples" section at the end of the man page and there was plenty of explanatory material from which you can deduce your own examples. Though I didn't give it much attention then, this challenge clearly stuck in my mind even to today and on reflection I now think quite differently to how I thought then. Examples matter. 

For those of us who enjoy binary taxonomies, there are two sorts of reasoning processes: deductive and inductive. These are described in various ways in the literature. One that is particularly succinct and helpful is from [ Naked Science](<http://www.nakedscience.org/mrg/Deductive%20and%20Inductive%20Reasoning.htm>) which describes the distinction as: 

Deductive reasoning arrives at a specific conclusion based on generalizations. Inductive reasoning takes events and makes generalizations. 

In the context of documentation, reasoning is the process of turning the words in the document into a model in your mind. Different people appear to vary in which style of reasoning they are most comfortable with, so good documentation must attempt to play to both styles. 

Documentation that plays to deductive reasoning will be filled with generalizations. This doesn't mean that it avoid details (as generalities would) but that it attempts to describe exactly - in complete generality - what each interface does, or how each concept applies, or what role each interaction plays. Such documentation can be very useful, but is can also lead to a feeling that you are drowning in detail. It can be a challenge to extract meaning and importance from such details. A lot of technical documentation seems to tend to this extreme. 

Documentation that plays to inductive reasoning will be full of examples of specific cases. It may explain each case very well but the coverage of the cases can never be perfect and it will inevitably leave out some information, typically the particular information that the reader is looking for. "How-tos" are a good example of this sort of document with maybe the extreme case being recipe books for cooking - they are full of sample recipes with very little space dedicated to explaining what makes a good recipe. These are very good if they chose just the right example, fairly good and quite accessible if they have chosen a good variety of examples, but usually lacking when you want to get down to the nitty-gritty. 

Documentation that plays to both types of reasoning will mix examples in with the generalizations, using them to embellish and explain those generalizations and as an excuse to make diversions into tangentially related topics. Examples are particularly good at highlighting contrasts which are themselves an important part of describing key concepts and clarifying why choices are made. The various multiplicities noted for Linux power management can doubtlessly provide lots of contrasts such as that between a "UART" serial driver that must be ready to receive full-rate data whenever it is not off, as opposed to a "USB" serial driver which only needs to be able to respond to a wake-up signal and has plenty of time to prepare itself for full data-rate messages. These would necessarily make different decisions about allowable power states. 

Returning to our wire-frame model which gives shape to some substance, it hopefully is not too much of a stretch to see examples as the skin on the model. These are the bits we can directly see, they reveal the texture or taste of the whole, and only hint at the bigger picture behind them. But they are an important part in closing the gaps that are left out of the big-picture descriptions. 

#### A worked example?

Having all these goals for introductory documentation may be nice, but are they actually useful? Can they lead to truly "good" documentation? Clearly they are not enough by themselves, but when combined with enough knowledge and experience, with some story-telling ability and an occasional touch of humor I believe that they can. To put this to the test, I've used them as a guide to producing some introductory documentation on Linux power management. The results will be presented next week when you, dear reader, can be the judge of whether the resulting documentation is actually "good".
