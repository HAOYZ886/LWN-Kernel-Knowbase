---
title: "Maintainers don't scale"
url: https://lwn.net/Articles/896918/
date: "June 6, 2022"
category: "Development model-Maintainers"
author: "By Jake Edge June 6, 2022 LSFMM"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jake Edge**  
June 6, 2022

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2022/>)

In something of a grab-bag session, Josef Bacik led a discussion about various challenges that Linux kernel maintainers face, some of which lead to burnout. The session was originally going to be led by Darrick Wong, but he was [unable to come to LSFMM](<https://lwn.net/ml/linux-xfs/20220426024331.GR17025@magnolia/>), so Bacik gathered some of Wong's concerns and combined them with his own in a joint storage and filesystem session at the [2022 Linux Storage, Filesystem, Memory-management and BPF Summit](<https://events.linuxfoundation.org/lsfmm/>) (LSFMM). As part of the discussion, Bacik presented his view on what the role of a kernel maintainer should be, which seemed to resonate with those present. 

#### Fuzzing and CVEs

He started by noting that some of the areas that Wong wanted to discuss had already come up in other sessions, including the [difficulties in setting up and running fstests](<https://lwn.net/Articles/896670/>) and the need for [backports of fixes to multiple stable releases](<https://lwn.net/Articles/896523/>). One topic that had not come up, though, was the increasing prevalence of people running fuzzers against filesystems, then filing for CVEs on the resulting problems. The CVE then triggers the kernel security process, which limits the amount of time available to make a fix and also limits which people can be involved in the investigation and bug-fixing process. 

[ ![\[Josef Bacik\]](https://static.lwn.net/images/2022/lsfmm-bacik2-sm.png) ](<https://lwn.net/Articles/897060/>)

In Bacik's opinion, a fuzzed filesystem does not constitute a security bug; "I know I'm probably a heretic for saying that". Filesystems can only be mounted by the root user, he said, but that is often countered with the example of a USB drive; "turn off automount" is his answer for that. In any case, the problem Wong described is not one that he has personally experienced, but he asked Ted Ts'o what he knew about it. 

Ts'o said that companies that want to sell their products into certain markets, such as to the US government, have special requirements with regard to fixing security problems. There are various time frames on how quickly the bugs must be fixed on production systems based on their severity score. If those are not met, then there is an auditing process where arguments like "we're not crazy enough to let random container people mount untrusted filesystems" can be made to explain why the CVE does not apply. But that involves describing the situation to a government bureaucrat, which, unsurprisingly, security teams do not enjoy, he said. 

The good news is that generally those types of bugs do not have a high severity score because they are not remotely exploitable. Chris Mason said that he did not think the security@kernel.org team was opening CVEs for the reports it receives; Ts'o agreed, but noted that the research labs that find these bugs often have a financial incentive to open CVEs. 

Luis Chamberlain said that the process followed depends on the subsystem maintainer to a large extent. Some subsystems, such as networking, do not fix bugs behind closed doors, while others do. The problem, of course, is that public fixes can lead to zero-day exploits until the fix is made and rolled out to distributions and into production. Ts'o said that it may also depend on the employer; there may be pressure applied to a particular employee who happens to also be the maintainer of a subsystem. That is not directly "maintainer stress" but is instead targeting an employee, he said. 

#### Expectations

Bacik said that he wanted to discuss "what we expect from maintainers". Traditionally, the expectation has been that they merge, write, test, review, and backport patches; that is a lot of work for one person. Many maintainers have come up with ad hoc solutions to try to scale that back, such as making sure that the developers for the subsystem are also reviewing patches. Ensuring that there are good testing setups is important to that effort as well. 

He would like to ensure that maintainers are also actively working on maintaining the community itself. Linux developers are passionate about the work that they do, but that passion sometimes leads to conflicts. "We get a little short with each other", which is not a good thing to maintain good working relationships, he said. 

Getting together in the same room, for example at LSFMM, is helpful, but he would like to see more done to get out ahead of these kinds of problems. It would be great to resolve these difficulties before they blow up and before they require getting together at a conference to fix them. A maintainer cannot follow all of the email threads, Bacik said, but it is fairly easy to spot "the big stuff that may be contentious". In those cases, he would like to encourage maintainers to either be a mediator, or find someone in the community who is good at mediating, to try to help developers from getting "overly invested in the code; in the end, this is just code". 

The intent is to find a way to ensure that those who are butting heads do it respectfully and in a way that will allow them to continue working together. If the maintainers could be "a little bit more proactive" about keeping an eye on contentious discussions, it would help head off these kinds of problems. 

Ts'o said that he has been having weekly ext4 development meetings that have been really helpful in reducing friction for filesystem development. Wong attends those meetings, which allows them to informally discuss things, maintainer to maintainer, that will ultimately need to be resolved on the linux-fsdevel mailing list. Jan Kara and other filesystem developers, who bring other useful perspectives, attend as well. Though Ts'o is sure that no one needs more meetings, he does wonder if some kind of monthly or quarterly gathering of developers could make a difference. 

Christian Brauner agreed with the idea of periodic meetings; he thought it would be "very useful". He has run into mailing-list conflicts along the way and said that it is only "in very very rare circumstances" when a third party comes along to calm things down on the list. Bacik said that it is important to ensure "that a random person can pop into a conversation" to ask the participants to calm down. That is not necessarily the case currently. 

It is tempting to suggest that Linus Torvalds (or some other individual) should be the one to step in and calm things down, but depending on a single person is not likely to work. Sometimes people are not paying attention at the right time; Bacik said he did not see the problems Brauner was referring to until well after the fact. Bacik wondered if people who are involved in a conflict of that sort should put out a call for a maintainer or other third party to step in and help calm things down. 

There are two areas that he would like to see maintainers take more of a lead in: conflict resolution and technical direction. Sometimes there are two developers who are disagreeing about the best way forward; "that's where I really want maintainers to step up more" and point out the direction that the project should take. That will help prevent the developers from continuing to butt heads and allow the project to progress. 

Ts'o said that he agrees that the main goal should be conflict resolution, but that is in an ideal world. In his experience, a maintainer generally picks up all of the tasks that the rest of the community is unwilling to do. Since no one steps up to do test engineering for ext4, for example, he takes care of it because it needs to be done. 

He is also spending quite a bit of time in recruiting new people to the ext4 community and in encouraging companies to staff positions for ext4 developers. That includes "working with an unnamed cloud company, not my own" to try to get some of its developers working on ext4 and, hopefully, to join that weekly development call. That is something he suggested other maintainers consider doing; other subsystem developers can also help the maintainer with that effort, which ultimately will help reduce burnout. A big problem is that "we simply do not have enough people on some of the filesystem teams", he said. 

Bacik said that it is his goal to reduce the number of things that maintainers are doing so that they can focus on the areas where their experience and knowledge are needed. Automation, especially in testing, is a good way to reduce the burden for maintainers. The important jobs for maintainers, conflict resolution and technical direction, cannot be automated; that is the piece that requires a human. 

#### Moving faster

He concluded the session by saying that he would like to see the development of Btrfs (and, by extension, other parts of the kernel) move more quickly than they do today. In order for that to happen, though, there need to be more and better tests that are being continuously run to detect when bugs are introduced. Testing and test engineering are the kinds of tasks that can be handed off to more junior engineers, he said, though the bar for test quality needs to be maintained. 

In general, he said that he wants to foster an environment where things move quickly, but mistakes will be made "and that's OK". Developers need to understand that they can make mistakes, those mistakes will be detected early on, and the change will simply be reverted. Then they can try again, That philosophy can be applied to the tests and testing infrastructure, as well as to new features for the kernel.
