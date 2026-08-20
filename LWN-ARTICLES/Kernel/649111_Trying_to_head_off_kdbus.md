---
title: Trying to head off kdbus
url: https://lwn.net/Articles/649111/
date: "June 24, 2015"
category: kdbus
author: "By Jake Edge June 24, 2015"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
June 24, 2015

There has been a fair amount of [opposition to kdbus](<https://lwn.net/Articles/640357/>)—an effort to move much of the D-Bus interprocess communication (IPC) mechanism into the kernel—that has [come up](<https://lwn.net/Articles/641275/>) since it was first [posted](<https://lwn.net/Articles/619068/>) back in November 2014\. Much of that criticism has been of the more or less unhinged variety, but there have also been a number of technical complaints about the IPC scheme. Two of the most persistent critics of the technical aspects of kdbus recently made preemptive attacks of a sort against kdbus—neither of which was particularly well-received by Greg Kroah-Hartman, who has been coordinating the kdbus effort. 

In a post to the linux-kernel mailing list on June 22, Andy Lutomirski [asked for](<https://lwn.net/Articles/649112/>) Linus Torvalds's opinion on whether kdbus should be merged. He made it clear that he is not asking about timing (i.e. whether he will accept a—so far hypothetical—pull request from Kroah-Hartman in this merge window), but: ""I mean whether you think that kdbus should be merged if it had appropriate review and people were okay with the implementation"". He goes on to say that the uncertainty about the future of kdbus makes it harder for those who might be inclined to review the code (or to review it again, in his case) to decide whether to do that review: 

The kernel people who would review it (myself included) probably don't want to review new versions at a line-by-line level, because we (myself included) either don't know whether there's any point or don't think that it should be merged *even if the implementation were flawless*. 

He continued by listing three main reasons that he felt kdbus should not be merged. First off, he doesn't believe that it is needed to solve the problems that it claims to solve and that most of what needs to be done could be done in user space. He is also concerned that the buffering model used by kdbus does not take control groups using a memory controller into account. Lastly: ""The sandbox model is, in my opinion, an experiment that isn't going to succeed."" 

As might be guessed, Kroah-Hartman was [less than pleased](<https://lwn.net/Articles/649130/>) by Lutomirski's message, calling it a ""preemptive pull request denial"". He was clearly irritated that Lutomirski had not had the courtesy to wait for his pull request—if he decided to make one—before attacking kdbus. But the technical complaints in the message went unread, as Kroah-Hartman seized on an incorrect, but offhand, comment early in Lutomirski's message to stop reading at that point. 

Things took a different turn on June 23 with a [request](<https://lwn.net/Articles/649133/>) from Eric Biederman—another persistent kdbus critic—to remove the kdbus tree from linux-next. Biederman cited ""significant work that was identified last merge window that has not yet been done, and who knows when it will be done"" as the reason to remove it. Furthermore: 

Having the kdbus tree in linux-next with the implicit suggestion that a pull request will be sent to Linus this merge window before the problems are addressed and we will have to repeat the mess from last merge window keeps me up at night. 

Once again, Kroah-Hartman was [unamused](<https://lwn.net/Articles/649134/>): ""No, that's not how this works. That's not how any of this works."" While Biederman claimed that the project was ignoring feedback, Kroah-Hartman disagreed: ""We are not ignoring _constructive_ feedback"". There is ""no valid reason"" to remove kdbus from linux-next, he concluded. 

But Biederman was [not to be deterred](<https://lwn.net/Articles/649152/>). He maintained that linux-next is for code that is destined for the next merge window, which implies that kdbus will be offered up for merging soon (as the 4.2 merge window is open), but that the code is not even remotely ready for merging. He suggested that lots of constructive feedback has simply been ignored. In the final analysis, he is trying to stop another pull request for kdbus by having the code removed from linux-next—not a common tactic, or even one that has been used at all before. 

Part of the trigger for both Lutomirski and Biederman's posts seem to have been the [announcement](<http://0pointer.net/blog/the-new-sd-bus-api-of-systemd.html>) of the sd-bus API that relies on the existing kdbus implementation (though it can fall back to using D-Bus), coupled with the most recent [systemd release](<http://lists.freedesktop.org/archives/systemd-devel/2015-June/033170.html>) that depends on sd-bus. In addition, one of the kdbus maintainers, David Herrmann, recently put out a [blog post](<https://dvdhrm.wordpress.com/2015/06/20/from-af_unix-to-kdbus/>) that describes kdbus in terms of the `AF_UNIX` sockets that are familiar to many developers. That may have been seen as a prelude to another pull request for kdbus. Beyond that, the systemd release announcement encourages distributions to add kdbus to their kernels and to use it with systemd/sd-bus. _That_ will simply [lead to problems](<https://lwn.net/Articles/649154/>) down the road, Lutomirski said: 

FWIW, once there are real distros with kdbus userspace enabled, reviewing kdbus gets more complicated -- we'll be in the position where merging kdbus in a different form from that which was proposed will break existing users. 

But it is not uncommon for user-space code to be written to try out new kernel features, sometimes long before they get merged. It is a way to shake out problems with the new feature and to prove that it is useful. On the flip side, though, getting a feature into a distribution is a subtle (or not-so-subtle) way to apply pressure for the feature to be merged. Over the years, there have been a number of examples of features that were released in distributions and were merged into the kernel over fairly strenuous objections because of their widespread availability. Some Android features (e.g. binder) likely fall into that category, but things like AppArmor also probably qualify. 

It is against that backdrop that Lutomirski and Biederman are making their moves. But, of course, Torvalds will have the final say and it turns out that he is [inclined to merge kdbus](<https://lwn.net/Articles/649157/>): 

So I am still expecting to merge it, mainly for a rather simple reason: I trust my submaintainers, and Greg in particular. So when a major submaintainer wants to merge something, that pulls a *lot* of weight with me. 

Though it is clear that one of the arguments made by kdbus proponents doesn't hold any water with him: performance. The [pull request](<https://lwn.net/Articles/640360/>) Kroah-Hartman made for the 4.1 kernel ran into some serious questions about the performance gains claimed, with Torvalds himself [expressing](<http://thread.gmane.org/gmane.linux.kernel/1930358/focus=1939166>) a fair amount of skepticism that speeding up D-Bus required a move into the kernel: 

If somebody wants to speed up dbus, they should likely look at the user-space code, not the kernel side. 

My guess is that pretty much the entirely of the quoted kdbus "speedup" isn't because it speeds up any kernel side thing, it's because it avoids the user-space crap in the dbus server. 

IOW, all the people who say that it's about avoiding context switches are probably just full of shit. It's not about context switches, it's about bad user-level code. 

He hasn't changed his mind at all (""I am not AT ALL impressed by the performance argument""), but still is likely to merge kdbus at some point. Given that there have been no new postings of kdbus since the pull request from the last merge window, it would seem a little late to be asking for a new version to be pulled in for 4.2. And Kroah-Hartman has not done so; there may be work going on to address some of the outstanding comments and questions (perhaps including the performance question) so that kdbus won't be offered up for merging again until the next development cycle. 

One of the key aspects of the dispute concerns the addressing of review comments. Lutomirski, Biederman, and others believe that there have been substantive constructive comments that have simply been ignored—though, with no new code posted, that may well be a premature conclusion. But "constructive", it seems, is in the eye of the beholder, so the kdbus developers may also feel that they have addressed those comments by disagreeing with them—and explaining why. It would seem to be a classic standoff situation that requires a final arbiter to make a determination. Torvalds is that arbiter, of course, and he seems to have decided—now it appears to just be a question of when, not if, kdbus is merged.
