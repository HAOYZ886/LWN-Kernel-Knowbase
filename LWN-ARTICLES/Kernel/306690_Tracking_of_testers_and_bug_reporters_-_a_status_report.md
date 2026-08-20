---
title: "Tracking of testers and bug reporters - a status report"
url: https://lwn.net/Articles/306690/
date: "November 11, 2008"
category: "Development model-Kernel quality; Releases-2.6.27"
author: "By Jonathan Corbet November 11, 2008"
---

By **Jonathan Corbet**  
November 11, 2008

A recurring topic at kernel summits is proper recognition for users who report bugs and test fixes. These people help the development process considerably, but they are far less visible than the developers who are creating those bugs in the first place. Since we would like to have more testers and reporters, it makes sense to reward them in whatever way we can. One of the strongest currencies we hold is credit for work done. So it stands to reason that crediting those who help the development process is in the interest of everybody involved. 

One mechanism developed for this purpose is a set of tags applied to patches before they are merged into the mainline. When a patch fixes a bug, the user(s) who reported that bug should be credited through the addition of a `Reported-by:` tag. Similarly, testers are credited with the `Tested-by:` tag. As it happens, some developers have adopted the habit of using `Reported-and-tested-by:` as a way of saving valuable newlines in the common case where a user fills both roles. 

There is a certain warm feeling that comes with having one's name stored in a changelog entry in the kernel source repository. But the amount of visibility which comes from this event is relatively small. So your editor decided to hack up his git data mining utility to track these tags. Without further ado, here are the top problem reporters and patch testers for the 2.6.27 development cycle: 

> Most credited 2.6.27 testers  
> ---  
> | `Reported-by` credits  
> ---  
> Adrian Bunk| 43| 21.0%  
> Robert P. J. Day| 12| 5.9%  
> Eric Sesterhenn| 5| 2.4%  
> Andrew Morton| 4| 2.0%  
> Alexey Dobriyan| 4| 2.0%  
> Denys Fedoryshchenko| 4| 2.0%  
> Yinghai Lu| 3| 1.5%  
> David S. Miller| 3| 1.5%  
> Vegard Nossum| 3| 1.5%  
> Stephen Rothwell| 3| 1.5%  
> Juha Leppanen| 3| 1.5%  
> Russell King| 2| 1.0%  
> Andi Kleen| 2| 1.0%  
> Ingo Molnar| 2| 1.0%  
> Benjamin Herrenschmidt| 2| 1.0%  
> Daniel J Blueman| 2| 1.0%  
> Daniel Exner| 2| 1.0%  
> Manuel Lauss| 2| 1.0%  
> Atsushi Nemoto| 2| 1.0%  
> Mikael Pettersson| 2| 1.0%  
> | `Tested-by:` credits  
> ---  
> Ingo Molnar| 7| 4.6%  
> Andrew Savchenko| 6| 3.9%  
> Rene Herman| 4| 2.6%  
> Mariusz Kozlowski| 3| 2.0%  
> Alexey Dobriyan| 3| 2.0%  
> Tino Keitel| 3| 2.0%  
> Robert Jarzmik| 3| 2.0%  
> KOSAKI Motohiro| 2| 1.3%  
> Benjamin Herrenschmidt| 2| 1.3%  
> Larry Finger| 2| 1.3%  
> Kenji Kaneshige| 2| 1.3%  
> Jack Howarth| 2| 1.3%  
> Gerald Schaefer| 2| 1.3%  
> Dennis Jansen| 2| 1.3%  
> Daniel J Blueman| 2| 1.3%  
> Daniel Exner| 2| 1.3%  
> Steven Noonan| 2| 1.3%  
> Rus| 2| 1.3%  
> Lawrence Greenfield| 2| 1.3%  
> Mark Langsdorf| 2| 1.3%  
  
> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

All told, there were a total of 205 `Reported-by:` and 153 `Tested-by:` credits entered during the 2.6.27 kernel cycle. This is arguably a reasonable start for a new tag, but it seems clear that a lot of problem reporters are not, yet, being credited in this manner. Your editor became curious to see just who is taking the time to credit these people; they, too, deserve some credit. A bit more script hacking yielded these tables: 

> Developers giving credits in 2.6.27  
> ---  
> | `Reported-by` credits  
> ---  
> Adrian Bunk| 44| 21.5%  
> Linus Torvalds| 12| 5.9%  
> Ingo Molnar| 8| 3.9%  
> Andrew Morton| 7| 3.4%  
> Peter Zijlstra| 7| 3.4%  
> Bartlomiej Zolnierkiewicz| 6| 2.9%  
> Yinghai Lu| 5| 2.4%  
> Jarek Poplawski| 5| 2.4%  
> Jiri Kosina| 5| 2.4%  
> Hugh Dickins| 4| 2.0%  
> FUJITA Tomonori| 4| 2.0%  
> Paul Mundt| 4| 2.0%  
> Vegard Nossum| 3| 1.5%  
> Russell King| 3| 1.5%  
> Jeremy Fitzhardinge| 3| 1.5%  
> Roland McGrath| 3| 1.5%  
> Haavard Skinnemoen| 3| 1.5%  
> Dmitry Torokhov| 3| 1.5%  
> David Woodhouse| 3| 1.5%  
> Oleg Nesterov| 3| 1.5%  
> | `Tested-by:` credits  
> ---  
> Pekka Enberg| 7| 4.6%  
> Linus Torvalds| 7| 4.6%  
> Takashi Iwai| 5| 3.3%  
> Bartlomiej Zolnierkiewicz| 5| 3.3%  
> Peter Zijlstra| 4| 2.6%  
> Rafael J. Wysocki| 4| 2.6%  
> Yinghai Lu| 4| 2.6%  
> Hugh Dickins| 4| 2.6%  
> Alan Stern| 4| 2.6%  
> Eric Miao| 4| 2.6%  
> Thomas Gleixner| 3| 2.0%  
> Lennert Buytenhek| 3| 2.0%  
> Alex Chiang| 3| 2.0%  
> Krzysztof Helt| 3| 2.0%  
> Stefan Richter| 3| 2.0%  
> Andy Whitcroft| 3| 2.0%  
> KOSAKI Motohiro| 2| 1.3%  
> Dennis Jansen| 2| 1.3%  
> Andrew Morton| 2| 1.3%  
> David S. Miller| 2| 1.3%  
  
The end result: Adrian Bunk gave over 20% of the total bug reporting credits - to himself. Beyond that, a number of the core developers are taking at least some time to credit those who report bugs and test patches. But, in the end, the 10,628 changesets merged for 2.6.27 probably contained quite a few more patches which could have carried such tags. If the reporting and testing tags are to become truly useful and significant, they will have to be more universally used. 

While your editor was at it, he also collected statistics for `Reviewed-by:` tags. These tags differ in that they are offered by the reviewer, who thereby states that a reasonably thorough review has been done and the code has not been found seriously wanting. Code review is perennially in short supply in just about any free software project, so, again, proper credit for reviewers seems like more than just a good idea. Here's the top 2.6.27 credited reviewers: 

> Developers with the most reviews (total 123)  
> ---  
> Ingo Molnar| 23| 18.7%  
> Paul Jackson| 12| 9.8%  
> Peter Zijlstra| 11| 8.9%  
> Christoph Lameter| 10| 8.1%  
> Aneesh Kumar K.V| 7| 5.7%  
> KOSAKI Motohiro| 6| 4.9%  
> Paul E. McKenney| 6| 4.9%  
> Jeff Moyer| 5| 4.1%  
> Robert P. J. Day| 4| 3.3%  
> Nadia Derbey| 3| 2.4%  
> Paul E. McKenney| 3| 2.4%  
> Mingming Cao| 2| 1.6%  
> Michael Buesch| 2| 1.6%  
> Li Zefan| 2| 1.6%  
> Matthew Wilcox| 2| 1.6%  
> Ingo Oeser| 2| 1.6%  
> Badari Pulavarty| 2| 1.6%  
  
If these numbers are to be believed, only 123 reviews were performed over the 2.6.27 development cycle. Even the most cynical observer is likely to agree that a bit more reviewing than that is going on. Most reviewers do not offer the associated tag, so their contribution goes unrecorded. In particular, Andrew Morton, who seems to review almost every patch which appears, should be at the top of the above list. 

Clearly, the task of ensuring proper credit for testers, bug reporters, and reviewers is still in its initial stages. But one has to start somewhere; this is more information than we had before. Hopefully, over time, the habit of crediting those who help with the development process will become more widespread. And that, with luck, will encourage more testing and bug reporting and, as a result, a better kernel.
