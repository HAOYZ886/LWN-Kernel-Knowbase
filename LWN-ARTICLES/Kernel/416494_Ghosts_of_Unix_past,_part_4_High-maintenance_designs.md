---
title: "Ghosts of Unix past, part 4: High-maintenance designs"
url: https://lwn.net/Articles/416494/
date: "November 23, 2010"
category: "Development model-Patterns"
author: "November 23, 2010 This article was contributed by Neil Brown"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

November 23, 2010

This article was contributed by Neil Brown

The bible [portrays](<http://www.biblegateway.com/passage/?search=Matthew+7:13-14&version=NIV>) the road to destruction as wide, while the road to life is narrow and hard to find. This illustration has many applications in the more temporal sphere in which we make many of our decisions. It is often the case that there are many ways to approach a problem that are unproductive and comparatively few which lead to success. So it should be no surprise that, as we have been looking for patterns in the design of Unix and their development in both Unix and Linux, we find [fewer](<http://lwn.net/Articles/411845/>) patterns of success than we [do](<http://lwn.net/Articles/412131/>) of [failure](<http://lwn.net/Articles/414618/>). 

Our final pattern in this series continues the theme of different ways to go wrong, and turns out to have a lot in common with the previous pattern of trying to "fix the unfixable". However it has a crucial difference which very much changes the way the pattern might be recognized and, so, the ways we must be on the look-out for it. This pattern we will refer to as a "high maintenance" design. Alternatively: "It seemed like a good idea at the time, but was it worth the cost?". 

While "unfixable" designs were soon discovered to be insufficient and attempts were made (arguably wrongly) to fix them, "high maintenance" designs work perfectly well and do exactly what is required. However they do not fit seamlessly into their surroundings and, while they may not actually leave disaster in their wake, they do impose a high cost on other parts of the system as a whole. The effort of fixing things is expended not on the center-piece of the problem, but on all that surrounds it. 

#### Setuid

The first of two examples we will use to illuminate this pattern is the "setuid" and "setgid" permission bits and the related functionality. In itself, the setuid bit works quite well, allowing non-privileged users to perform privileged operations in a very controlled way. In fact this is such a clever and original idea that the inventor, Dennis Ritchie, was granted a patent for the invention. This patent was since placed in the public domain. Though ultimately pointless, it is amusing to speculate what might have happened had the patent rights been asserted, leading to that aspect of Unix being invented around. Could a whole host of setuid vulnerabilities have been avoided? 

The problem with this design is that programs which are running setuid exist in two realms at once and must attempt to be both a privileged service provider, and a tool available to users - much like the [confused deputy](<http://www.cis.upenn.edu/~KeyKOS/ConfusedDeputy.html>) recently [pointed out](<https://lwn.net/Articles/416744/>) by LWN reader "cmccabe." This creates a number of conflicts which requires special handling in various different places. 

The most obvious problem comes from the inherited environment. Like any tool, the programs inherit an environment of `name=value` assignments which are often used by library routines to allow fine control of certain behaviors. This is great for tools but potentially quite dangerous for privileged service providers as there is a risk that the environment will change the behavior of the library and so give away some sort of access that was not intended. All libraries and all setuid programs need to be particularly suspicious of anything in the environment, and often need to explicitly ignore the environment when running setuid. The recent [glibc vulnerabilities](<http://lwn.net/Articles/412048/>) are a perfect example of the difficulty of guarding against this sort of problem. 

An example of a more general conflict comes from the combination of setuid with executable shell scripts. This did not apply at the time that setuid was first invented, but once Unix gained the `#!/bin/interpreter` (or "shebang") method of running scripts it became possible for scripts to run setuid. This is almost always insecure, though various different interpreters have made various attempts to make it secure, such as the "`-b`" option to `csh` and the "taint mode" in `perl`. Whether they succeed or not, it is clear that the setuid mechanism has imposed a real burden on these interpreters. 

Permission checking for signal delivery is normally a fairly straightforward matching of the UID of the sending process with the UID of the receiving process, with special exceptions for UID==0 (root) as the sender. However, the existence of setuid adds a further complication. As a setuid program runs just like a regular tool, it must respond to job-control signals and, in particular, must stop when the controlling terminal sends it a `SIGTSTP`. This requires that the owner of the controlling terminal must be able to request that the process continues by sending `SIGCONT`. So the signal delivery mechanism needs special handling for `SIGCONT`, simply because of the existence of setuid. 

When writing to a file, Linux (like various flavors of Unix) checks if the file is setuid and, if so, clears the setuid flag. This is not absolutely essential for security, but has been found to be a valuable extra barrier to prevent exploits and is a good example of the wide ranging intrusion of setuid. 

Each of these issues can be addressed and largely have been. However they are issues that must be fixed not in the setuid mechanism itself, but in surrounding code. Because of that it is quite possible for new problems to arise as new code is developed, and only eternal vigilance can protect us from these new problems. Either that, or removing setuid functionality and replacing it with something different and less intrusive. 

It was recently [announced](<http://lwn.net/Articles/412237/>) that Fedora 15 would be released with a substantially reduced set of setuid programs. Superficially this seems like it might be "removing setuid functionality" as suggested, but a closer look shows that this isn't the case. The plan for Fedora is to use filesystem capabilities instead of full setuid. This isn't really a different mechanism, just a slightly reworked form of the original. Setuid stores just one bit per file which (together with the UID) determines the capabilities that the program will have. In the case of setuid to root, this is an all or nothing approach. Filesystem capabilities store more bits per file and allow different capabilities to be individually selected, so a program that does not need all of the capabilities of root will not be given them. 

This certainly goes some way to increasing security by decreasing the attack surface. However it doesn't address the main problem that the setuid programs exist in an uncertain world between being tools and being service providers. It is unclear if libraries which make use of environment variables after checking that setuid is not in force, will also correctly check if capabilities are not in force. Only a comprehensive audit would be able to tell for sure. 

Meanwhile, by placing extra capabilities in the filesystem we impose extra requirements on filesystem implementations, on copy and backup tools, and on tools for examining and manipulating filesystems. Thus we achieve an uncertain increase in security at the price of imposing a further maintenance burden on surrounding subsystems. It is not clear to this author that forward progress is being achieved. 

#### Filesystem links

Our second example, completing the story of high maintenance designs, is the idea of "hard links", known simply as links before symbolic links were invented. In the design of the Unix filesystem, the name of a file is an entity separate from the file itself. Each name is treated as a link to the file, and a file can have multiple links, or even none - though of course when the last link is removed the file will soon be deleted. 

This separation does have a certain elegance and there are certainly uses that it can be put to with real value. However the vast majority of files still only have one link, and there are plenty of cases where the use of links is a tempting but ultimately sub-optimal option, and where symbolic links or other mechanisms turn out to be much more effective. In some ways this is reminiscent of the Unix permission model where most of the time the subtlety it provides isn't needed, and much of the rest of the time it isn't sufficient. 

Against this uncertain value, we find that: 

  * Archiving programs such as `tar` need extra complexity to look out for hard links, and to archive the file the first time it is seen, but not any subsequent time. 

  * Similar care is needed in `du`, which calculates disk usage, and in other programs which walk the filesystem hierarchy. 

  * Anyone who can read a file can create a link to that file which the owner of the file may not be able to remove. This can lead to users having charges against their storage quota that they cannot do anything about. 

  * Editors need to take special care of linked files. It is generally safer to create a new file and rename it over the original rather than to update the file in place. When a file has multiple hard links it is not possible to do this without breaking that linkage, which may not always be desired. 

  * The Linux kernel's internals have an awkward distinction between the "dentry" which refers to the name of a file, and the "inode", which refers to the file itself. In many [cases](<http://lwn.net/Articles/403070/>) we find that a dentry is needed even when you would think that only the file is being accessed. This distinction would be irrelevant if hard links were not possible, and may well relate to the choice made by the developers of Plan 9 to not support hard links at all. 

  * Hard links would also make it awkward to reason about any name-based access control approach (as discussed in [part 3](<https://lwn.net/Articles/414618/>)) as a given file can have many names and so multiple access permissions. 
While hard links are certainly a lesser evil than setuid, and there is little motivation to rid ourselves of them, they do serve to illustrate how a seemingly clever and useful design can have a range of side effects which can weigh heavily against the value that the design tries to bring. 

#### Avoiding high maintenance designs

The concept described here as "high maintenance" is certainly not unique to software engineering. It is simply a specific manifestation of the so-called [law of unintended consequences](<http://en.wikipedia.org/wiki/Unintended_consequences>) which can appear in many disciplines. 

As with any consequences, determining the root cause can be a real challenge, and finding an alternate approach which does not result in worse consequences is even harder. There are no magical solutions on offer by which we can avoid high maintenance designs and their associated unintended consequences. Rather, here are three thoughts that might go some small way to reining in the worst such designs. 

  1. Studying history is the best way to avoid repeating it, and so taking a broad and critical look at our past has some hope of directing is well for the future. It is partly for this reason that "patterns" were devised, to help encapsulate history. 

  2. Building on known successes is likely to have fewer unintended consequences than devising new ideas. So following the pattern that started this series of "full exploitation" is, where possible, most likely to yield valuable results. 

  3. An effective way to understand the consequences of a design is to document it thoroughly, particularly explaining how it should be used to someone with little background knowledge. Often writing such documentation will highlight irregularities which make it easier to fix the design than to document all the corner cases of it. This is certainly the [experience](<http://lwn.net/Articles/247788/>) of Michael Kerrisk who maintains the man pages for Linux, and, apparently, of our Grumpy Editor who found that [fixing the cdev interface](<https://lwn.net/Articles/75585/>) made him less grumpy than trying to document it, unchanged, for LDD3. 

When documenting the behavior of the Unix filesystem, it is desirable to describe it as a hierarchical structure, as that was the overall intent. However, honesty requires us to call it as directed acyclic graph (DAG) because that is what the presence of hard links turns it into. It is possible that having to write DAG instead of hierarchy several times might have been enough to raise the question of whether hard links are such a good idea after all. 

#### Harken to the ghosts

In his classic novella "A Christmas Carol", Charles Dickens uses three "ghosts" to challenge Ebenezer Scrooge about his ideology and ethics. They reminded him of his past, presented him with a clear picture of the present, warned him about future consequences, but ultimately left the decision of how to respond to him. We, as designers and engineers, can similarly be challenged as we reflect on these "Ghosts of Unix Past" that we have been exploring. And again, the response is up to us. 

It can be tempting to throw our hands up in disgust and build something new and better. Unfortunately, mere technical excellence is no guarantee of success. As Paul McKenney [astutely observed](<http://lwn.net/Articles/412687/>), at the 2010 Kernel Summit, economic opportunity is at least an equal reason for success, and is much harder to come by. Plan 9 from Bell Labs attempted to learn from the mistakes of Unix and build something better; many of the mistakes explored in this series are addressed quite effectively in Plan 9\. However while Plan 9 is an important research operating system, it does not come close to the user or developer base that Linux has, despite all the faults of the latter. So, while starting from scratch can be tempting, it is rare that it has a long-term successful outcome. 

The alternative is to live with our mistakes and attempt to minimize their ongoing impact, deprecating that which cannot be discarded. The x86 CPU architecture seems to be a good example of this. Modern 64-bit processors still support the original 8086 16-bit instruction set and addressing modes. They do this with minimal optimization and using only a small fraction of the total transistor count. But they continue to support it as there has been no economic opportunity to break with the past. Similarly Linux must live with its past mistakes. 

Our hope for the future is to avoid making the same sort of mistakes again, and to create such compelling new designs that the mistakes, while still being supported, can go largely unnoticed. It is to this end that it is important to study our past mistakes, collect them into patterns, and be always alert against the repetition of these patterns, or at least to learn how best to respond when the patterns inevitably recur. 

So, to conclude, we have a succinct restatement of the patterns discovered on this journey, certainly not a complete set of patterns to be alert for, but a useful collection nonetheless. 

Firstly there was "Full exploitation": a pattern hinted at in that early paper on Unix and which continues to provide strength today. It involves taking one idea and applying it again and again to diverse aspects of a system to bring unity and cohesiveness. As we saw with signal handlers, not all designs benefit from full exploitation, but those that do can bring significant value. It is usually best to try to further exploit an existing design before creating something new and untried. 

"Conflated" designs happen when two related but distinct ideas are combined in a way that they cannot easily be separated. It can often be appropriate to combine related functionality, whether for convenience or efficiency, but it is rarely appropriate to tie aspects of functionality together in such a way that they cannot be separated. This is an error which can be recognized as the design is being created, though a bit of perspective often makes it a lot clearer. 

"Unfixable" designs are particularly hard to recognize until the investment of time in them makes replacing them unpalatable. They are not clearly seen until repeated attempts to fix the original have resulted in repeated failures to produce something good. Their inertia can further be exacerbated by a stubbornness to "fix it if it kills me", or an aversion to replacement because "it is better the devil you know". It can take substantial maturity to know when it is time to learn from past mistakes, give up on failure, and build something new and better. The earlier we can make that determination, the easier it will be in the long run. 

Finally "high maintenance" designs can be the hardest for early detection as the costs are usually someone else's problem. To some extent these are the antithesis of "fully exploitable" designs as, rather than serving as a unifying force to bring multiple aspects of a system together, they serve as an irritant which keeps other parts unsettled yet doesn't even produce a pearl. Possibly the best way to avoid high maintenance designs is to place more emphasis on full exploitation and to be very wary of including anything new and different. 

If identifying, describing, and naming these patterns makes it easier to detect defective designs early and serves to guide and encourage effective design then they will certainly have filled their purpose. 

#### Exercises for the interested reader

  1. Identify a design element in the IP protocol suite which could be described as "high maintenance" or as having "unintended consequences". 

  2. Choose a recent extension to Linux and write some comprehensive documentation, complete with justification and examples. See if that suggests any possible improvements in the design which would simplify the documentation. 

  3. Research and enumerate uses of "hard links" which are not adequately served by using symbolic links instead. Suggest technologies that might effectively replace these other uses. 

  4. Describe your "favorite" failings in Unix or Linux and describe a pattern which would help with early detection and correction of similar failings.
