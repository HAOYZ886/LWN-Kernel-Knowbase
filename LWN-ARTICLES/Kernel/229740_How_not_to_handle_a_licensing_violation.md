---
title: How not to handle a licensing violation
url: https://lwn.net/Articles/229740/
date: "April 11, 2007"
category: Broadcom 43xx; Copyright issues
author: 
---

For a long time, Broadcom 43xx wireless interfaces had no free Linux driver. Happily, a dedicated group of developers reverse engineered the device, and, over time, were able to create the missing driver. In the process, they implemented some features which were not available in Broadcom's proprietary driver. Not wanting their code to become part of the proprietary version, the Linux bcm43xx developers chose the GPL for their code - a choice that most other Linux driver developers make as well. 

More recently, the bcm43xx developers noticed that the OpenBSD "bcw" driver looked very much like their code. It would appear that the developer of this driver looked to the Linux code for inspiration and took a bit more than just ideas. GPL-licensed code is meant to be shared and reused, but it is not meant to be relicensed unilaterally by third parties. So the bcm43xx hackers decided to talk to the OpenBSD developer about the apparent copying which had taken place. 

Unfortunately, [their message](<https://lwn.net/Articles/229742/>) was copied to a rather large number of people, along with a few mailing lists. The response from the OpenBSD side took two forms, neither of which will be at all surprising to those who have watched how that community operates: 

  * The OpenBSD developers do honestly care about the provenance and legitimacy of their code. So the claims were taken seriously; OpenBSD leader Theo de Raadt [remarked](<https://lwn.net/Articles/229746/>) ""This is a major problem in our code base"" and said that the issue would be resolved. 

  * Those developers immediately launched a counterattack as if they were a beehive which had just been hit by a rock. They complained about the wide distribution of the mail, tore into the bcm43xx developers ([example](<https://lwn.net/Articles/229747/>): ""You are a very poor example of humankind""), repeatedly put down the "precious GPL," and, inevitably, dragged their maintenance of OpenSSH into the discussion. To many, it looked like an overt attempt to attack the messenger and take attention away from the real problem. 

In theory, this situation should be simple to resolve. The OpenBSD developer, Marcus Glocker, has [acknowledged](<https://lwn.net/Articles/229748/>) the problem and stated that he was aware of it before the discussion began. He says: 

I wanted to make some quick progress (maybe too quick), and rewrite the functions in question after seeing some first success, e.g. receivment of first frames, which isn't the case right now. 

The bcm43xx developers have said from the outset that they would be willing to relicense at least some of the affected code. The two groups should be able to sit down, talk things through, and end up with everybody being happy. 

That has not happened. Instead, we got a nasty flame war, the outright deletion of the OpenBSD bcw driver, and the bizarre sight of Theo de Raadt [claiming](<https://lwn.net/Articles/229749/>) that _he_ is the person with ""at least some fucking empathy in my soul."" That is not how things should have gone. There need be no enmity between the Linux and BSD communities; when something like this happens it's worth looking at why in the hope of avoiding a recurrence in the future. 

The initial contact from the Linux side was clearly mishandled. When licensing issues come up, the generally-accepted first step is a quiet, polite, _private_ message seeking a solution. People rarely respond well when the first communication about a problem is broadcast to the world. Had the bcm43xx developers conducted a private chat with the OpenBSD bcw developer, chances are that the issues would have been worked out with relatively little fuss. Most developers are interested in solving problems, after all. 

The OpenBSD crowd also missed its chance for a quiet solution when it went on the attack. Attempts to divert the discussion through _ad hominem_ attacks, profanity, and general bluster will never lead to a civil conversation or a peaceful resolution of a problem. Deleting the bcw driver (and blaming the Linux community for its loss) seems childish at best. The use of OpenSSH as a sort of trump card is just strange, and a little worrying. 

Needless to say, it would also have been better if the code had not been used contrary to its license in the first place. But code licensing issues are complex. In a world where vast amounts of code are floating around under mutually-incompatible licenses, the occasional problem is certain to turn up. That's why the "open source licensing compliance" companies are able to make a living. But licensing disagreements between free software projects are rarely so intractable that they cannot be solved by rational discussion. The next time a situation like this comes up - something which is certain to happen, sooner or later, and the Linux community might just find itself on the other side of the table \- we can only hope that all of the people involved will approach a solution in a way which allows that rational discussion to take place.
