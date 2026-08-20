---
title: "A change in direction for security-module stacking?"
url: https://lwn.net/Articles/970070/
date: "April 23, 2024"
category: "Security-Security modules"
author: "By Jake Edge April 23, 2024"
---

By **Jake Edge**  
April 23, 2024

The long-running effort to complete the work on stacking (or composing) the Linux security modules (LSMs) recently encountered a barrier—in the form of a "suggestion" to discontinue it from Linus Torvalds. His complaint revolved around the indirect function calls that are used to implement LSMs, but he also did not think much of the effort to switch away from those calls. While it does not appear that a major course-change is in store for LSMs, it is clear that Torvalds is not happy with the direction of that subsystem. 

In an April 9 [post](<https://lwn.net/ml/linux-security-module/CAHk-=wh+_xXpnHfUv=FwGWcce4XwqwKvtq7LcxK6WKmbf4eGGA@mail.gmail.com/>) to the linux-security-module mailing list, Torvalds decried the stacking plans in part because ""we just had *another* hardware security issue with speculated indirect branches go public"". He was referring to the [branch history injection flaw](<https://lwn.net/Articles/969210/>) that was the most recent in a long line of speculative-hardware vulnerabilities. Torvalds said that he recognized that stacking LSMs was a ""design decision and a target"" for over a decade, but it needs a rethink: 

> So I say "suggestion" in the subject line, but really think it needs to be more than that: this whole "nested LSM" stuff as a design goal just needs to be all rolled back, and the new design target is "one LSM, enabled statically at build time, without the need for indirect calls". 

He also said that he was aware of KP Singh's [work](<https://lwn.net/ml/linux-security-module/20240207124918.3498756-1-kpsingh%40kernel.org/>) to use [static calls](<https://lwn.net/Articles/815908/>) to avoid the indirect function calls in LSMs, but seemed to suggest that the patches were ""random hacks"". There are some seeming misunderstandings in Torvalds's complaints, however. For one thing, there is no "nesting" of LSMs—"stacking" either really—the security solutions are composed, instead. A given hook function in the core kernel will effectively traverse the list of active LSMs, calling the corresponding hook function if present for an active LSM, until it gets a denial, which short-circuits the rest of the calls. If no LSM denies the access, it is allowed. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

There are a number of real use cases for having multiple LSMs active in the kernel. Our [2022 article](<https://lwn.net/Articles/912775/>) on the feature describes the history of how we have gotten to this point and why it is important to be able to enable multiple LSMs on current systems. It has been possible to compose any number of "minor" LSMs for years now, but the final push is on to allow more than one "major" LSM (e.g. SELinux, Smack, AppArmor) to be enabled. The main reason behind the need for that is containers, so that a Fedora container that uses SELinux can run on an Ubuntu host that uses AppArmor, for example. 

In a response to Torvalds, Kees Cook [patiently pointed out](<https://lwn.net/ml/linux-security-module/202404091041.63A1CFC1A@keescook/>) some of that history, including the reasons behind the LSM-stacking work. In the end, he said, it has simplified things to the point where subsystems that logically should be LSMs could be switched: 

> The general "LSM stacking" work, while it does add a certain kind of complexity, has actually made the many pre-existing manual layering of LSMs much more sane and simpler to reason about. Now the security hooks are no longer a random sprinkling of calls tossed over the core kernel, and are now all singularly well defined. This started years ago with pulling the "capabilities" checking into a well-defined LSM, and continued from there for things like Yama, and has now finally reached the last, and perhaps most historically invasive, LSM: IMA/EVM [[Integrity Measurement Architecture/Extended Verification Module](<https://sourceforge.net/p/linux-ima/wiki/Home/>)], which is finally now a well defined LSM too. 
> 
> I don't think it's sane to demand that LSM stacking be removed. That's just not the world we live in -- we have specific and large scale needs for the infrastructure that is in place. 

Cook also disagreed with the characterization of static calls, noting that they have been needed by the LSM subsystem for over a year just for the performance benefits. But Torvalds [strongly disagreed](<https://lwn.net/ml/linux-security-module/CAHk-=whXwcBzFJd65gQ09EGt1yuH6RyuErhOgvJK8pjkAVw07w@mail.gmail.com/>); he said that the reason for stacking is: ""Just because you people cannot agree"". He also explained that it was not static calls themselves that were random hacks, but that the use of them for LSMs is, in part because of the random-seeming limit of 11 levels of "nesting". His parting shot was to further paint the LSMs as an attack vector against the kernel. 

As might be guessed, Cook [saw things differently](<https://lwn.net/ml/linux-security-module/202404091430.A00477BC9@keescook/>). He noted, again, that stacking has been around for quite some time now; his current system has five separate LSMs activated, not to mention the capabilities LSM that is always present. "Stacking" is not removable at this point, but, beyond that, the most recent vulnerability is not in the LSM subsystem: ""the attack vector is broken CPUs"". In addition, the array to hold the static calls needs to have a limit and there are 11 LSMs available for the kernel, which is why that number was chosen. 

LSM maintainer Paul Moore [was rather unhappy](<https://lwn.net/ml/linux-security-module/CAHC9VhSMWr9OEsHQ6y=3fw+Qk_1mWg2GcCfERHD4vn9Y_XOJsQ@mail.gmail.com/>) with another part of Torvalds's message. For whatever reason, Torvalds was unable to resist taking a shot at the LSM subsystem and its developers in his initial message: 

> Yes, I realize that the "security" in the LSM name is a bad joke, and that to a first level approximation the LSM people don't actually care about real security, and that the goal is just "policy". 

Moore wondered if the insult was really just rooted in stress from yet another hardware flaw affecting the kernel, but even so, the effects will be borne by the LSM developers. Because of who he is, Torvalds's words have much greater weight, Moore said. It is thus rather ironic that Torvalds is asking—""(demanding? it's hard to say at this point)""—those he just insulted to rework their subsystem. Moore pointed out that insults are not likely to be particularly motivating. 

Beyond that, as Cook had pointed out, it is far too late to remove stacking entirely. The LSM developers will act on Torvalds's email, Moore said, but the first step is to reduce the performance penalty of the indirect calls—and, in the process, mitigate the hardware security flaws they expose—by getting the LSM static calls patches merged. ""The rest will need more discussion, preferably after things have cooled down and we call all look at things with a more objective lens."" 

Casey Schaufler, who has been pushing the full LSM-stacking work upstream for 12 years or more at this point, [replied](<https://lwn.net/ml/linux-security-module/37ee8a6c-ecd1-4669-8e60-df6adeb0eab7@schaufler-ca.com/>) to Torvalds's complaints by agreeing with some of his points. As with other developers, he is completely in favor of replacing the indirect calls, but is unsure what they should be replaced with if static calls are not the right approach. ""While I can't change the brain dead behavior of 21st century hardware I am perfectly willing to re-write the entire $%^&*( LSM layer if it can be done in a way that makes you happy."" But Moore [said](<https://lwn.net/ml/linux-security-module/CAHC9VhQfTRCOCO4hQtvRBd-UdwVKkG-5g=Ng=oo3vTy8C-u6SA@mail.gmail.com/>) that Schaufler should not head down that path; Moore has no plans to move to ""a single-LSM approach to satisfy a spur of the moment comment triggered by the latest hardware flaw"". He repeated his plan to convert the LSMs to use static calls ""and go from there"". 

Meanwhile, Greg Wettstein [thought](<https://lwn.net/ml/linux-security-module/20240411103351.GA11734@wind.enjellic.com/>) that more sweeping changes are needed for LSMs in order to support ""an environment where there are going to be multiple and potentially industry specific mandated security controls"". While he agrees that the performance and attack-vector characteristics of indirect branches need to be mitigated, he does not see static calls as the right path, at least given the current LSM architecture. 

> There needs to be an 'LSM' architecture that allows a security policy to be implemented for a process hierarchy, just like we do for all other resource namespaces. Platform owners are then free to select whether they want to implement multiple orthogonal security controls or lock the platform into a single control of their choosing. 

While that may sound like a situation tailor-made for a BPF solution, he cautioned against that approach, as well, citing the discussion about a [recent patch](<https://lwn.net/ml/linux-security-module/20240324223231.6249-1-greg@enjellic.com/>). There have been no replies to his post, however, which may be an indication that radical changes along those lines are fairly unlikely. 

In truth, Torvalds's post seems to have been made in haste—coupled with serious unhappiness about the latest hardware flaw. Backing out all of the LSM-stacking work seems well-nigh impossible at this point, especially considering the user-space compatibility guarantees that Torvalds himself regularly enforces. Beyond that, the container use case for multiple major LSMs is not going away either, so some sort of solution will be needed there. The LSM development community seems willing to engage on alternate solutions, but one suspects that what has come out of more than a decade of effort will eventually be adopted.
