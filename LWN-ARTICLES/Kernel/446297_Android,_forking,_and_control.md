---
title: "Android, forking, and control"
url: https://lwn.net/Articles/446297/
date: "June 6, 2011"
category: Development model
author: "By Jonathan Corbet June 6, 2011"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
June 6, 2011

Many words have been said about the relationship between the Android project and the mainline kernel development community. At LinuxCon Japan, James Bottomley took the stage to say a few more. There are, he said, some interesting lessons to be learned from that disconnect. If the development community pays attention to what has been going on, we may be better placed to deal well with such situations in the future. 

James started with the statement that Android is, hands down, the most successful Linux distribution ever produced. Its adoption dwarfs that of Linux on the desktop - and on the server too. Android's success is spectacular, but it was achieved by: 

  * Forking the kernel, 
  * Rewriting the toolchain and C library, 
  * Developing a custom Java-based application framework, and 
  * Working from an extreme dislike of the GPL 

In other words, James said, Android is a poster child for how one should not work in the open source community. They did everything we told them not to, and won big. While we would like the Android developers to change and do some things differently, their success suggests that, perhaps, Android is not the only group in need of change. Maybe the community needs to reevaluate how it weighs code quality against market success; do we, he asked, need a more commercially-oriented metric? 

One of the big assumptions brought into this debate is that forking is a bad thing. Android started by forking the kernel and writing its own user space mostly from scratch, and the community has duly condemned these moves. But it is worth understanding what the Android developers were trying to do; Android started by finding adopters first; only then did they get around to actually implementing their system. At that point, the time pressures were severe; they had to have something ready as soon as possible. There is a lot to be said for the development community's patch review and acceptance processes, but they do tend to be somewhat open-ended. Google couldn't wait for that process to run its course before it shipped Android, so there was little choice other than forking the kernel. 

Was forking the kernel wrong? In a sense, James said, it cannot be wrong: the GPL guarantees that right, after all. The right is guaranteed because forking is sometimes necessary, and rights are meaningless if they are not exercised. In this specific case, without a fork, the Android project would have had a hard time achieving its goals (with regard to power management and more) in a commercially useful time. The result would have been a delayed Android release which would have led to less success in the market or, perhaps, missing the market window entirely and failing to take off. Forks, in other words, can be good things - they can enable groups to get things done more quickly than going through the community process. 

Is forking equal to fragmentation, he asked? It is an important question; fragmentation killed the Unix market back in the 1990's. James claimed that forks which fail do not fragment the community; they simply disappear. Forks which are merged back into their parent project also do not represent fragmentation; they bring their code and their developers back to the original project. The forks which are harmful are those which achieve some success, carrying part of the community with them, and which do not return to the parent project. From that, James said, it follows that it is important for the community to help forks merge back. 

The Android developers, beyond forking the kernel, also took the position that the GPL is bad for business. The project's original goal was to avoid GPL-licensed code altogether; the plan was to write a new kernel as well. In the end, a certain amount of reason prevailed, and the (GPL-licensed) Linux kernel was adopted; there are a few other GPL-licensed components as well. So, James said, we can thank Andy Rubin - from whom the dislike of the GPL originates - for conclusively demonstrating that a handset containing GPL-licensed code can be successful in the market. It turns out that downstream vendors really don't care about the licensing of the code in their devices; they only care that it's clear and compliant. 

What about Android's special application framework? James said that the Java-based framework is one of the most innovative things about Android; it abstracts away platform details and moves the application layer as far away from the kernel as possible. The framework restricts the API available to applications, giving more control over what those applications do. Given the structure of the system, it seems that rewriting the C library was entirely unnecessary; nobody above the framework makes any sort of direct use of it anyway. 

So maybe Android didn't do everything wrong. But there were some mistakes made; the biggest, from James's point of view, was the lack of a calendar which can handle SyncML. That made Android handsets relatively useless for business users. One of the keys to the Blackberry's success was its nice calendaring. Motorola had seen this problem and implemented its own proprietary SyncML calendaring application for the Droid; that actually made things worse, as business users would get an Android handset with the idea that it would work with their calendars. If they ended up with something other than the Droid, they would be disappointed and, eventually, just buy an iPhone instead. Android had no SyncML support until 2.1, when a new, written-from-scratch implementation was added. The cost of this mistake was one year of poor corporate uptake. 

The other problem with Android, of course, is its "walled garden" approach to development. Android may be an open-source project, but Google maintains total control over the base release; nobody else even sees the code until Google throws it over the wall. No changes from partners get in, so there is no community around the code, no shared innovation. As an example, Android could have solved its calendar problem much sooner had it been willing to accept help from outside. Google's total control over Android was needed to give the project its market focus. It was a necessary precondition for market dominance, but it is bad for community and has forced Google to reinvent a lot of wheels. 

Another big mistake was being sued by Oracle. That suit is based on Android's rewrite of Java which, in turn, was entirely motivated by fear of the GPL. Had Android been built on Oracle's GPL-licensed Java code base, there would have been no suit; Google would have been protected by the GPL's implied patent license. If Oracle wins, rewriting Java will turn out to be a hugely expensive exercise in license avoidance. And the sad fact is that the license is entirely irrelevant: the Java runtime's API constitutes a "bright line" isolating applications from the GPL. 

#### Lessons learned

So what can be learned from all of this? James reiterated that forking can be a good thing, but only if the results are merged back. The Android fork has not been merged back despite a great deal of effort; it's also not clear that the Android developers have bought into the solutions that the kernel community has come up with. Maybe, he said, we need to come up with a way to make merging easier. The community should have a better way of handling this process, which currently tends to get bogged down in review, especially if the fork is large. 

Projects which create forks also need to think about their processes. Forks tend to create not-invented-here mentalities which, in turn, lead to a reluctance to show the resulting code. It's no fun to post code that you know is going to be panned by the community. The longer a fork goes, the worse the situation gets; fixing of fundamental design mistakes (which is what [wakelocks](<https://lwn.net/Articles/318611/>) are in the community's view) gets harder. Preventing this problem requires forks to be more inclusive, post their code more often, and ask the community's advice - even if they do not plan to take that advice. It's important to open the wall and let ideas pass through in both directions. 

James talked a bit about "licensing fears," stating that the GPL is our particular version of FUD. The discussions we have in the community about licensing tend to look like scary problems to people in industry; less heat from the community on this subject would do a lot of good. The fear of the GPL is driven by outside interests, but we tend to make it easy for them. The community should be more proactive on this front to allay fears; pointing to Android as an example of how GPL-licensed code can work is one possibility. The Linux Foundation does some of this work, but James thinks that the community needs to help. The GPL, he said, is far easier to comply with than most commercial licensing arrangements; that's a point we need to be making much more clearly. 

We should also design more "bright line" systems which make the question of GPL compliance clear. The kernel's user-space ABI is one such system; developers know that user-space code is not considered to be derived from the kernel. Making the boundary easy to understand helps to make the GPL less scary. 

The community should do better at fostering and embracing diversity, encouraging forks (which can create significant progress) and helping them to merge back. Currently, James said, the kernel gets a "C - must do better" grade at best here. We only take code from people who look like us; as a result, the Android merge attempt was more painful than it needed to be. 

Companies, in turn, should aim for "control by acclamation" rather than control by total ownership. Linus Torvalds was given as an example; he has a lot of control, but only because the community trusts him to do the right thing. In general, if the community trusts you, it will happily hand over a lot of control; that's why the benevolent dictator model is as common as it is. On the other hand, companies which try to assert control through walled garden development or by demanding copyright assignment from contributors have a much harder time with the community. 

In summary, James said, Android was a fiasco for everybody involved; we all need to figure out how to do better. We need to find better ways of encouraging and managing forks and allaying licensing fears. Projects which create forks should be thinking about merging back from the outset. Then projects which (like Android) are a commercial success can also be a community success. 

[Your editor would like to thank the Linux Foundation for funding his travel to Japan to attend this event.]
