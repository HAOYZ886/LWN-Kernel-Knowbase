---
title: Why kernel development still uses email
url: https://lwn.net/Articles/702177/
date: "October 1, 2016"
category: "Development model-Patch management"
author: "By Jonathan Corbet October 1, 2016 Kernel Recipes"
---

By **Jonathan Corbet**  
October 1, 2016

* * *

[Kernel Recipes](<https://lwn.net/Archives/ConferenceByYear/#2016-Kernel_Recipes>)

In a world full of fancy development tools and sites, the kernel project's dependence on email and mailing lists can seem quaintly dated, if not positively prehistoric. But, as Greg Kroah-Hartman pointed out in a [Kernel Recipes](<http://kernel-recipes.org/en/2016/>) talk titled "Patches carved into stone tablets", there are some good reasons for the kernel community's choices. Rather than being a holdover from an older era, email remains the best way to manage a project as large as the kernel. 

In short, Greg said, kernel developers still use email because it is faster than any of the alternatives. Over the course of the last year, the project accepted about eight changes per hour — every hour — from over 4,000 developers sponsored by over 400 companies. It must be doing something right. The list of maintainers who accepted at least one patch per day contains 75 entries; at the top of the list, Greg himself accepted 9,781 patches over the year. Given that he accepts maybe one third of the patches sent his way, it is clear that the patch posting rate is much higher than that. 

Finding tools that can manage that sort of patch rate is hard. A poor craftsman famously complains about his tools, Greg said, but a good craftsman knows how to choose excellent tools. 

So which tools are available for development work? Greg started by looking at GitHub, which, he said, has a number of advantages. It is "very very pretty" and is easy to use for small projects thanks to its simple interface. GitHub offers free hosting and unlimited bandwidth, and can (for a fee) be run on a company's own infrastructure. It makes life easy for the authors of drive-by patches; Greg uses it for the [usbutils](<https://github.com/gregkh/usbutils>) project and gets an occasional patch that way. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

On the other hand, GitHub does not scale to larger projects. He pointed at the [Kubernetes](<https://github.com/kubernetes/kubernetes>) project, which has over 4,000 open issues and 511 open pull requests. The system, he said, does not work well for large numbers of reviewers. It has [![\[Greg Kroah-Hartman\]](https://static.lwn.net/images/conf/2016/recipes/GregKroah-Hartman-sm.jpg)](<https://lwn.net/Articles/702283/>) a reasonable mechanism for discussion threads attached to pull requests — GitHub has duplicated email for that feature, he said — but only the people who are actually assigned to a pull request can see that thread. GitHub also requires online access, but there are a lot of kernel developers who, for whatever reason, do not have good access to the net while they are working. In general, it is getting better, but projects like Kubernetes are realizing that they need to find something better suited to their scale; it would never work for the kernel. 

Moving on to [Gerrit](<https://www.gerritcodereview.com/>), Greg started to list its good points, but stopped short, saying he didn't know any. Actually, there was one: project managers love it, since it gives them the feeling that they know what is going on within the project. He noted that Google, which promotes Gerrit for use with the Android project, does not use it for any of its internal projects. Even with Android, Gerrit is not really needed; Greg pointed out that, in [the complicated flow chart](<https://source.android.com/source/life-of-a-patch.html>) showing how to get a patch into Android, Gerrit has a small and replaceable role. 

Gerrit, he said, makes patch submission quite hard; [Repo](<https://code.google.com/p/git-repo/>) helps a bit in that regard, but not many projects use it. Gerrit can be scripted, but few people do that. An audience member jumped in to say that using Gerrit was like doing one's taxes every time one submits a patch. The review interface makes it clear that the Gerrit developers do not actually spend time reviewing code; he pointed in particular at the need to separately click through to view every file that a patch touches. It is hard to do local testing of patches in Gerrit, and tracking a patch series is impossible. All discussions are done through a web interface. Nobody, Greg said, will do reviews in Gerrit unless it's part of their job. 

What about plain-text email? Email has been around forever, and everybody has access to it in one form or another. There are plenty of free email providers and a vast number of clients. Email works well for non-native speakers, who can use automatic translation systems if need be. Email is also friendly from an accessibility standpoint; that has helped the kernel to gain a number of very good blind developers. Email is fast, it makes local testing easy, and remote testing is possible. Writing scripts to deal with emailed patches is easily done. And there is no need to learn a new interface to work with it. 

On the other hand, the quality of email clients is not uniformly good. Some systems, like Outlook, will uniformly corrupt patches; as a result, companies doing kernel development tend to keep a Linux machine that they can use to send patches in a corner somewhere. Gmail is painful for sending patches, but it works very well as an IMAP server. Project managers, he noted, tend not to like email. He seemed to think of that as an advantage, or, at worst, an irrelevance, since the kernel's workflow doesn't really have any project-manager involvement anyway. 

Email integrates easily with other systems; it functions well with the kernel's [0-day build and boot testing system](<https://01.org/lkp/documentation/0-day-test-service>) for example. It also is nicely supported by the [patchwork](<http://jk.ozlabs.org/projects/patchwork/>) system, which is used by a number of kernel subsystems to track the status of patches. Patchwork will watch a mailing list, collect the patches seen there, and track acks and such. It provides a nice status listing that project managers love. 

In summary, Greg said, email matters because it is simple, supports the widest group, and is scalable. But the most important thing is that it grows the community. When new developers come in, the first thing they have to do is to learn how the project works. That includes reading the reviews that developers are doing; that is how one learns what developers care about and how to avoid mistakes. With the kernel, those reviews are right there on the mailing list for all to see; with a system like Gerrit, one has to work to seek them out. 

As Rusty Russell once said, if you want to get smarter, the thing to do is to hang out with smart people. An email-based workflow lets developers hang out with a project's smart people, making them all smarter. Greg wants Linux to last a long time, so wants to see the kernel project use tools that help to bring in new developers. Email, for all its flaws, is still better than anything else in that regard. 

[Your editor thanks Kernel Recipes for supporting his travel to this event.]
