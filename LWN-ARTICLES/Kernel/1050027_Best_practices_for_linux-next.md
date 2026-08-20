---
title: "Best practices for linux-next"
url: https://lwn.net/Articles/1050027/
date: "December 12, 2025"
category: "Development model-linux-next; linux-next"
author: "By Jonathan Corbet December 12, 2025 Maintainers Summit"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
December 12, 2025

* * *

[Maintainers Summit](<https://lwn.net/Articles/1049982/>)

One of the key components in the kernel's development process is the linux-next repository. Every day, a large number of branches, each containing commits intended for the next kernel development cycle, is pulled into linux-next and integrated. If there are conflicts between branches, the linux-next process will reveal them. In theory, many other types of problems can be found as well. Some developers feel that linux-next does not work as well as it could, though. At the 2025 Maintainers Summit, Mark Brown, who helps to keep linux-next going, led a session on how it could be made to work more effectively. 

[![\[Mark Brown\]](https://static.lwn.net/images/conf/2025/ossjp/MarkBrown-sm.png)](<https://lwn.net/Articles/1050031/>) The idea for the session, he said, began when he heard complaints about some types of ""odd fixes"" not appearing in linux-next before landing in the mainline. Maintainers may manage many branches, and some of those branches are not being pulled into linux-next. Additional problems come about when maintainers cherry-pick commits before sending them to the mainline, making it harder to track those commits as they appeared in linux-next. There is also, he said, one maintainer who refuses to put fixes into linux-next because he does not want them tested next to feature-oriented changes. 

He asked the group whether more pressure should be applied to maintainers to put their repositories into linux-next. Nobody seemed to disagree with that idea. 

Linus Torvalds had a different complaint: fixes that _do_ get into linux-next, but then are not forwarded on to him for merging into the mainline. There are cases where the kernel has a known bug, there is a fix that is ready, but he does not have it. 

Another problem for Torvalds is buggy patches that break linux-next for everybody, interfering with its primary purpose. He has repeatedly asked for problematic repositories to be simply removed from linux-next, and to be notified when that happens. 

Steve Rostedt said that he will occasionally rebase his repositories that appear in linux-next, mostly to add tags to commits. That can have an effect similar to cherry-picking, where commits will change their IDs. Torvalds said that habit, too, can cause trouble in linux-next, but Brown said that rebasing is really only a problem if others have built on the repository that has been rebased. The most problematic tree when it comes to cherry-picking, he said, is the DRM (graphics) subsystem, which does extensive cherry-picking of patches. DRM maintainer Dave Airlie answered that the scale of that subsystem requires cherry-picking. All other subsystems have the same problems, he said, but they haven't yet gotten big enough to make that clear. He expressed willingness to have the assembled maintainers develop a better process for DRM, but did not believe that they could do it. 

Torvalds repeated that he would like to see consequences when buggy commits break linux-next, that the guilty repository needs to be kicked out — at least temporarily. Once the relevant maintainer has acknowledged and fixed the problem, the repository would be allowed back in. If the problem is _not_ fixed, though, then the repository should be kept out and not pulled during the next merge window. 

#### Avoiding linux-next bugs

Ted Ts'o said that the filesystem developers had often run into problems testing linux-next, caused by bugs introduced by other subsystems. In response, linux-next maintainer Stephen Rothwell set up a process where the filesystem trees are pulled in first, and an `fs-next` tag is set once that process is is complete, before other repositories are pulled. That creates a sort of limited linux-next containing only filesystem trees and a few others that filesystem subsystems depend on, but without most of the rest of the work being done in the kernel community. Linux-next as a whole is too flaky, Ts'o said, but `fs-next` is useful for testing. 

Arnd Bergmann said that all subsystems are not equal, and that handling the filesystem repositories first makes sense. The cost of mistakes in that area is especially high. Miguel Ojeda suggested a similar process for other higher-level subsystems, merging them into a side branch before merging the result into linux-next. That would create a sort of two-level process, integrating broad areas of the kernel before throwing everything together. 

Airlie said that breaking linux-next (and mainline) around the rc1 release is expected, but breaking it every day is expensive. That leads to the wrong people finding problems, he said. Torvalds said that real development teams should be running continuous-integration testing on their own branches and only test linux-next occasionally. It is not reasonable, he said, to expect developers to debug problems introduced by other subsystems. The real problem, he added, is that some repositories clearly are not seeing any testing at all. That often happens with the smaller subsystems, he said, and linux-next is not really helping in this case. 

Brown agreed that maintainers should care most about their own repositories, but said that it is still worthwhile to keep an eye on linux-next as a whole; that improves the chances that the culprit behind a problem will fix it before the buggy commit makes it into the mainline. Ojeda says that he has to test linux-next every day to find problems that affect the Rust build. 

Brown asked about what tests could be run by linux-next itself to help with Rust problems, and got some ideas, but wondered about which specific configurations should be tested. Torvalds worried that, if some specific configurations are tested, others will only test with those, and he will still get ""the weird bugs"" that show up with other configurations. Ojeda said that he did not want people testing with canned configurations; rather, they should use their own and see which problems, in particular, appear. 

Part of the problem, Airlie said, is that a lot of testing is done in continuous-integration systems, but nobody actually boots their laptop with their new code. So the parts of the system that actually show text on the screen aren't checked, and inevitably break. 

#### Build problems

Jiri Kosina asked how the build problems that Torvalds encounters when pulling repositories happen, given that those repositories should already be pulled into linux-next and build-tested there. Torvalds answered that ordering issues are one source of problems; one repository will implicitly depend on commits in another, and the order in which linux-next pulls the repositories hides the problem. More often, though, Torvalds discovers that basic configurations just haven't been tested. He added that "allmodconfig" builds (which build the entire kernel as loadable modules) cause some code to be disabled, and can thus hide problems. As a result, there are some problems that he only encounters when he builds a kernel with his own personal configuration. 

Ts'o expressed a concern that dropping repositories from linux-next would cause a loss of test coverage; it would be necessary, he said, to take that action in a public way. Torvalds said he would like to get an automatic email just before the merge window telling him which repositories have been disabled. 

During the merge window, Torvalds said, he typically does about 30 merges each day. Build failures are relatively rare during this time; they will happen maybe two or three times during the merge window, but they are still annoying and unnecessary. 

Jakub Kicinski said that he often finds merge conflicts before linux-next does, and wondered if he should send resolutions. Torvalds said that he doesn't look at conflict resolutions done by maintainers, at least not before he does the resolution himself. But having the resolution to check his work afterward can be helpful. He does depend on resolutions from others for conflicts in Rust code, where he is less confident of his own abilities. 

At the end of the session, Rostedt asked how long maintainers should keep a bug fix in linux-next before sending it to Torvalds. The answer was that, if it is a bug that affects others, the fix should be sent before the next rc release — for obvious fixes, at least. If a fix is not obvious, though, more time should be taken to be sure that the fix is correct.
