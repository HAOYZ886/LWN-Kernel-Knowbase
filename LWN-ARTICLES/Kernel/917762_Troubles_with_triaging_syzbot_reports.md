---
title: Troubles with triaging syzbot reports
url: https://lwn.net/Articles/917762/
date: "December 14, 2022"
category: "Development model-Bug reporting; Filesystems-Fuzzing"
author: "By Jake Edge December 14, 2022"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
December 14, 2022

A report from the [syzbot](<https://github.com/google/syzkaller/blob/master/docs/syzbot.md>) kernel fuzz-testing robot does not usually spawn a vitriolic mailing-list thread, but that is just what happened recently. While the invective is regrettable, the underlying issue is important. The dispute revolves around how best to report bugs to affected subsystems and, ultimately, how not to waste maintainers' time. 

Al Viro was [apparently fed up](<https://lore.kernel.org/all/Y5VGCefLZmrOyd0Z@ZenIV/>) with syzbot reports that involved the ntfs3 filesystem but that were not copied (CCed) to the maintainers of ntfs3. The syzbot message was sent to the kernel mailing list, but Viro shouted his reply that ""ANY BUG REPORTS INVOLVING NTFS3 IN REPRODUCER NEED TO BE CCED TO MAINTAINERS OF NTFS3"". That complaint had been relayed several times in the past, he indicated, without the problem getting fixed, so he was planning to stop looking at the reports. In fact, they will be ""getting triaged straight to /dev/null here"". 

After an ... impenetrable [reply](<https://lore.kernel.org/all/20221211075612.2486-1-hdanton@sina.com/>) from Hillf Danton, Viro [followed up](<https://lore.kernel.org/all/Y5WXNhO9UNogf8lq@ZenIV/>) with more details of the problems he sees. He pointed to a [post from September](<https://lore.kernel.org/all/YzEJ2D8kga+ZRDZx@ZenIV/>) where he made a similar request and said that others had also reported these kinds of problems to the maintainers of syzbot. The issue is that the mail sent by syzbot does not contain enough useful information for someone to quickly determine if it pertains to their area of interest: 

> It's really a matter of triage; as it is, syzkaller folks are expecting that any mail from the bot will be looked into by everyone on fsdevel, on the off-chance that it's relevant for them. What's more, it's not just "read the mail" - information in the mail body is next to useless in such situations. [...] 
> 
> What really pisses me off is that on the sending side the required check is trivial - if you are going to fuzz a filesystem, put a note into report, preferably in subject. Sure, it's your code, you get to decide what to spend your time upon (you == syzkaller maintainers). But please keep in mind that for [recipients] it's a lot of recurring work, worthless for the majority of those who end up bothering with it. Every time they receive a mail from that source. 
> 
> Ignore polite suggestions enough times, earn a mix of impolite ones and .procmailrc recipes, it's that simple... 

Danton [misunderstood](<https://lore.kernel.org/all/20221211102208.2600-1-hdanton@sina.com/>) what Viro was complaining about, but Matthew Wilcox tried to [explain](<https://lore.kernel.org/all/Y5X7RBznTD6rPfro@casper.infradead.org/>). The complaint is not that the linux-fsdevel list is being copied on the mail, but that the ntfs3 maintainers are not. Wilcox said: ""So this is just noise. And enough noise means that signal is lost."" 

Viro [agreed](<https://lore.kernel.org/all/Y5ZDjuSNuSLJd8Mn@ZenIV/>) and painstakingly described exactly how he (and any other interested recipient of a syzbot report) would triage it, which eventually ends up at the [syzkaller dashboard entry](<https://syzkaller.appspot.com/bug?id=e8da337c58f1a620dce15898ae21af03e0a5ddb3>) for the bug and its [syzkaller reproducer](<https://syzkaller.appspot.com/text?tag=ReproSyz&x=10aaf2b7880000>). That file, which resembles ""line noise"", as Viro noted, does contain enough information to see that it was an ntfs3 filesystem that was being fuzzed. But that information is not in the email (or, better still, email subject), nor is it used to direct the report to the right people to look at it. The underlying problem is that the syzkaller/syzbot maintainers are not providing the relevant data, which should be easily obtained: 

> From what I've seen in various discussions, the assumption of syzkaller folks seems to be that most of the relevant information is in stack trace and that's sufficient for practical purposes - anything beyond that is seen as unwarranted special-casing. [...] 
> 
> Face it, the underlying assumption is broken - for a large class of reports the stack trace does not contain the relevant information. It needs to be augmented by the data that should be very easy to get for the bot. Sure, your code, your priorities, but reports are only useful when they are not ignored and training people to ignore those is a bad idea... 

Ted Ts'o [agreed](<https://lore.kernel.org/all/Y5d565XVsinbNNL2@mit.edu/>), noting that he has been asking for improvements of this sort for several years. Syzbot ""is not doing things that really could be done automatically --- and cloud VM time is cheap, and upstream maintainer time is expensive"". In effect, the syzbot developers are not being respectful of upstream maintainers' time, he said. Things have been improving, but not in this particular area: 

> Now, to be fair to the Syzbot team, the Syzbot console has gotten much better. You can now download the syzbot trace, and download the mounted file system, when before, you had to do a lot more work to extract the file system (which is stored in separate constant C array's as compressed data) from the C reproducer. So have things have gotten better. 

Marco Elver [reported](<https://lore.kernel.org/all/CANpmjNNCQEXpJt1PQptyr8mrBbhWpToCRfvUT+RXmw5EA5EwVw@mail.gmail.com/>) that the problem is being worked on by the syzbot project. He pointed to a [bug report comment](<https://github.com/google/syzkaller/issues/3393#issuecomment-1330305227>) from syzkaller (and syzbot) creator Dmitry Vyukov that was posted at the end of November. It linked to [yet another message](<https://lore.kernel.org/all/Y4WE08+n1sZvSt4M@ZenIV/>) from Viro complaining about the problem. Looking further at the bug comment thread makes it clear that progress is being made on identifying what to search for and on adding tags to email subject lines to identify which filesystem is being fuzzed. 

The thread eventually went completely off the rails, including a message that seems likely to draw a response from the kernel [code of conduct](<https://docs.kernel.org/process/code-of-conduct.html>) committee. The overall tone of the thread was unfortunate, at least in spots, but both Ts'o and Viro (especially the latter) spent a fair amount of time patiently reiterating the problems that have been raised multiple times along the way, albeit at a lower volume. Those requests did not go far, so, as Ts'o put it, ""maybe something a bit more.... assertive by Al [Viro] is something that will inspire them to prioritize this feature request"". 

Fuzz testing generates a huge number of reports; in order for the testing to be effective—useful—those reports have to be acted upon. Since that is the goal, it obviously makes sense to create reports that can be quickly routed to the right people. This not the first time we have seen [complaints about fuzzing reports](<https://lwn.net/Articles/904293/>), and in a filesystem context, but hopefully we are on track to see improvements soon.
