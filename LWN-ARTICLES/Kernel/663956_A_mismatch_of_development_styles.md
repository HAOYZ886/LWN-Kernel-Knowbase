---
title: A mismatch of development styles
url: https://lwn.net/Articles/663956/
date: "November 11, 2015"
category: kdbus
author: "By Jake Edge November 11, 2015"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jake Edge**  
November 11, 2015

The [kdbus](<https://github.com/systemd/kdbus>) interprocess communication (IPC) mechanism has been somewhat controversial since well before it was [first posted](<https://lwn.net/Articles/619068/>) to the kernel mailing list a little over a year ago. It is targeted at bringing the capabilities of the [D-Bus message bus](<http://www.freedesktop.org/wiki/Software/dbus/>) to the kernel for performance, security, and reliability purposes. But, for a variety of reasons, it has not been merged into the mainline. It now turns out that the whole mechanism is being rethought and redesigned before another attempt to get it into the kernel will be made. 

A recent discussion on the kernel mailing list didn't really illuminate much about the future direction of kdbus, but it did, perhaps, highlight some of the reasons that the project has not had a smooth path into the kernel. The thread was spawned by the [announcement](<https://lwn.net/Articles/664014/>) on the fedora-devel mailing list that kdbus would be removed from the Rawhide kernel—where it has been available for the last six months or so—while the developers ""rethink some of the approach they are taking with kdbus"". That led some to wonder what kind of rethinking was being done and how it might change kdbus. 

Richard Weinberger [started](<https://lwn.net/Articles/664049/>) the thread by noting the Rawhide announcement and wondering about the plans. That was greeted with a terse [reply](<https://lwn.net/Articles/664051/>) from Greg Kroah-Hartman—who has been coordinating the kdbus effort—that the changes would be ""posted for review 'when it's ready'"". He seemed to [bristle a bit](<https://lwn.net/Articles/664054/>) at the follow-up request for more information: 

Since when do people post "plans" or "design documents" on lkml without real code? Again, code will be posted when it's ready, like any other kernel submission. 

Part of Kroah-Hartman's irritation is likely due to the [preemptive attempts](<https://lwn.net/Articles/649111/>) to head off kdbus back in June. No information was posted to the list, but some of the details of the kdbus rethinking were revealed in Lennart Poettering's [systemd status report [YouTube]](<https://www.youtube.com/watch?v=I4AAjEaTehk>) at the inaugural [systemd conference](<https://systemd.events/systemdconf-2015>) held in early November. Around the 24-minute mark, Poettering noted that kdbus had been "less of a success story" than many of the other systemd components he had already discussed. 

He mentioned the removal of kdbus from the Rawhide kernel, but stressed that the project was not dead. The developers were simply taking a step back and "figuring out how we want to do this stuff". The existing kdbus put the compatibility with "classic D-Bus" into the kernel, but the team is now looking into doing less D-Bus compatibility in the kernel and moving that to user space. He is not sure how it will look in the end and is not willing to make any predictions about when it will end up in the kernel. It was needed in the kernel two years or so ago when the project got started and that is still true today, he said. 

Beyond that, some took exception to Kroah-Hartman's refusal to discuss plans on the mailing list. Josh Triplett [quoted](<https://lwn.net/Articles/664065/>) the development-process documentation, which suggests involving the kernel community early in the process. But kdbus developer David Herrmann [thought](<https://lwn.net/Articles/664071/>) that was not really applicable in this case: 

I think comparing kdbus to "behind closed doors" development models is unfair. We chose to center our development around DBus, not the kernel. Anybody who is interested in kdbus discussions could have easily joined the DBus and systemd communication channels (and *many* people did). I see little reason in cross-posting everything to LKML, especially given that our communication is rarely mail-based. 

The last line in that quote points out a fundamental mismatch between how kernel development and systemd/kdbus development are done. It turns out that most of the discussion is not going on behind closed doors, but it is happening on IRC or in person, which makes it hard for outsiders to follow or monitor. In his talk, Poettering noted that systemd had moved to GitHub, which has been quite a boon for the project. In particular, though the code review facilities of GitHub are nowhere near perfect, they enabled systemd to do much more code review than the previous mailing-list-centric workflow. 

So systemd is using different tools than the kernel is, which makes it hard for kernel developers to get a view into how kdbus is evolving. The expectation of at least some kernel developers is that the patch set would continue to be updated and posted in full periodically. Those postings would show how the review comments had been incorporated into the code. But there have only been patches to kdbus posted since the last posting of the full patch set back in March. Peter Zijlstra was [not happy with that approach](<https://lwn.net/Articles/664078/>): 

I've seen many random kdbus patches, but not a single comprehensive posting one could attempt to review after that. 

The normal way is to regularly post a complete set of patches for review -- which very much includes folding patches you get back into the series, not post a 100+ patch series. 

But Kroah-Hartman doesn't see anything out of the ordinary with the development of kdbus. It is currently being rethought and will be [posted for review](<https://lwn.net/Articles/664081/>) when that process has completed: 

And that will happen, when the code is ready, nothing different is happening here from any other code submission. I don't know why people somehow think we don't know how to do this whole thing, it's as if they don't trust us, which is sad. 

Andy Lutomirski, who has been one of the strongest critics of kdbus, was [unhappy](<https://lwn.net/Articles/664087/>) with how his feedback had been handled. Lutomirski has extensively reviewed the code and had [several technical disagreements](<https://lwn.net/Articles/641275/>) with the kdbus approach. To a large extent, the kdbus developers disagreed with his criticisms at the time, but it is unclear whether that has changed: ""Now the submission has been withdrawn (because of feedback received so far? from me?) and there will apparently be a new submission out of the blue, allegedly based on feedback."". He called that style of receiving feedback "off-putting", which made him far less interested in doing further reviews. 

While the thread was, at least partly, just a squabble between developers, it does point to a looming potential problem. Doing development on a mailing list is increasingly seen as unworkable for many projects these days. There are other tools (like GitHub, IRC, or even Google Hangouts) that provide a different experience that some projects are finding better suited to their needs. But it does tend to make those projects more insular—it is harder for outsiders to come up to speed at their own pace, as the pace is set by the realtime nature of some of the tools. 

In any case, it would seem that we are a ways off from seeing another version of kdbus. It does sound like some of the criticisms, especially those about putting D-Bus semantics into the kernel, have been acted upon. That may result in a more generalized IPC mechanism, which is likely to be well-received. Those who are most interested can join the [D-Bus/systemd communication channels](<http://www.freedesktop.org/wiki/Software/systemd/#mailinglists>), as Herrmann suggested, to find out what is being planned, others will just have to wait and see what gets posted.
