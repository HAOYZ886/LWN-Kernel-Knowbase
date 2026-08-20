---
title: On moving on from being a maintainer
url: https://lwn.net/Articles/670087/
date: "January 6, 2016"
category: "Development model-Maintainers"
author: "By Jake Edge January 6, 2016"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jake Edge**  
January 6, 2016

The maintainer of a Linux subsystem has a large, and largely thankless, job to do. While reviewing patches is clearly technical in nature, much of the rest of the work is almost clerical—and it takes enough time that there may be little or no time for programming or other actively technical tasks. Thus, it is not a surprise to see that maintainers burn out over time and start looking for other work (in the kernel or elsewhere) to do. In fact, it _is_ surprising that it doesn't happen more often. However, there is no clear path for relinquishing the maintainer role—and generally no succession plan—which can make the transition kind of tricky. 

That scenario is currently playing out for the md (software RAID) subsystem, where maintainer Neil Brown has [announced](<https://lwn.net/Articles/670284/>) that he intends to step down on February 1\. Brown got ""sucked in"" to being the md maintainer in late 2001 because there was no one else doing it. Since there is no ""obvious candidate for replacement maintainer - no one who has already been doing significant parts of the maintainer role"", he intends to create a maintainership vacuum in the hopes that one or more folks step up to fill the role. 

He laments that he has not been able to attract additional maintainers, though he noted that there are some folks in the md community who are certainly capable of doing the job. The question, in Brown's eyes, is whether or not they ""care about the code and the subsystem"", which is something that only individuals can determine for themselves. That means he doesn't feel in a position to appoint anyone to the role and would like to see folks volunteer. By stepping down, he hopes that might create a little pressure to step up. 

As he noted, Linus Torvalds has expressed a preference for small maintainer teams, which might make sense for md. Another alternative might be for the device mapper (dm) maintainer team to add md maintainer duties. Beyond just md, though, he is also relinquishing the maintainer role for the [mdadm](<http://neil.brown.name/blog/mdadm>) administrative tool. That could be handled by the new md maintainer or team, though he would prefer to see different people maintain md and mdadm. According to Brown (in response to an email query), there are two main reasons he favors that separation: it worked well when he handed off nfsd to Bruce Fields and nfs-utils to Steve Dickson, but also that ""it encourages public accountability - it is too easy for me to make an API change to md, start using it in mdadm, and not have anyone review it"". 

Brown's announcement describes the responsibilities of a maintainer as he sees them: 

So I'm hoping to get one or more volunteers to be maintainer: 

  * to gather and manage patches and outstanding issues, 
  * to review patches or get them reviewed 
  * to follow up bug reports and get them resolved 
  * to feed patches upstream, maybe directly to Linus, maybe through some other maintainer, depending on what relationships already exist or can be formed, 
  * to guide the longer term direction (saying "no" is important sometimes), 
  * to care, 
but also to be aware that maintainership takes real effort and time, as does anything that is really worthwhile. 

As can be seen, there is a great deal to do. He also noted that another job he had previously spent a lot of time on, following the linux-raid mailing list to provide support on md-related issues, has fallen by the wayside for him. But, in what might be a preview of what will happen with the maintainer role, others in the md community have stepped up. He is ""absolutely thrilled that the gap has been more than filled by other very competent community members"". 

Though he soon won't be doing the maintainer's job, Brown is not disappearing from the md world. He has committed to continue work on the [raid5-journal](<https://lwn.net/Articles/665299/>) and [raid1-cluster](<https://www.kernel.org/doc/Documentation/md-cluster.txt>) projects. He would also be willing to mentor any volunteers and will still review some patches as well as comment on designs. He concluded with a call to action: 

Take the bull by the horns and start *being* a maintainer(team). I won't get in your way and I'll help where I can. 

Certainly Brown is not the only maintainer to find that they have tired of doing that job. Back at the end of 2014, John Linville [stepped down](<https://lwn.net/Articles/626689/>) as the wireless network maintainer by "promoting" some of the subsystem maintainers and handing off the wireless driver patch handling to Kalle Valo. The mac80211, bluetooth, and nfc maintainers were asked to start pushing their patches directly to network maintainer David Miller, rather than going through Linville's tree. It seems that Linville had been more successful in finding maintainers along the way—or in them finding him—which simplified his decisions when he decided to work on other things. The wireless subsystem is rather larger than md, however, which tends to foster a bigger pool of potential maintainers. 

As with other parts of the kernel development process, the maintainership role is a bit haphazard. Maintainers handle their duties as they see fit and focus their efforts in different ways. The main job is to get the right patches in a—hopefully—timely manner to Torvalds and into the mainline. Determining which patches are "right" is part of the job, too, of course, but some maintainers (including Torvalds) largely leave that job to their sub-maintainers, while others do not. Some of that can be seen in our [article](<https://lwn.net/Articles/670209/>) on how patches get to the mainline. 

In most cases, the maintainer's style has likely come about organically over time—certain things seemed to work for them. But that style may impact how a transition out of the role will need to be handled. For md, there may be some folks interested in the maintainer job (or, more likely, team), who spoke up in the short thread. While it may seem a little crazy to those outside the kernel development community, creating a vacuum as an exit strategy may actually work better than other mechanisms—at least for some subsystems and maintainers.
