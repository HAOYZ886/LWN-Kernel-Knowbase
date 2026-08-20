---
title: What to do about CVE numbers
url: https://lwn.net/Articles/801157/
date: "October 4, 2019"
category: "Security-CVE numbers"
author: "By Jonathan Corbet October 4, 2019 Kernel Recipes"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
October 4, 2019

* * *

[Kernel Recipes](<https://lwn.net/Archives/ConferenceByYear/#2019-Kernel_Recipes>)

Common Vulnerability and Exposure (CVE) numbers have been used for many years as a way of uniquely identifying software vulnerabilities. It has become increasingly clear in recent years that there are problems with CVE numbers, though, and [increasing numbers of vulnerabilities](<https://lwn.net/Articles/791855/>) are not being assigned CVE numbers at all. At the [2019 Kernel Recipes](<https://kernel-recipes.org/en/2019/>) event, Greg Kroah-Hartman delivered a "40-minute rant with an unsatisfactory conclusion" on CVE numbers and how the situation might be improved. The conclusion may be "unsatisfactory", but it seems destined to stir up some discussion regardless. 

CVE numbers, Kroah-Hartman began, were meant to be a single identifier for vulnerabilities. They are a string that one can "throw into a security bulletin and feel happy". CVE numbers were an improvement over what came before; it used to be impossible to effectively track bugs. This was especially true for the "embedded library in our product has an issue" situation. In other words, he said, CVE numbers are good for zlib, which is embedded in almost every product and has been a source of security bugs for the last fifteen years. 

Since CVE numbers are unique, somebody has to hand them out; there are now about 110 organizations that can do so. These include both companies and countries, he said, but not the kernel community, which has nobody handling that task. There also needs to be a unifying database behind these numbers; that is the [National Vulnerability Database (NVD)](<https://nvd.nist.gov/>). The NVD provides a searchable database of vulnerabilities and assigns a score to each; it is updated slowly, when it is updated at all. The word "national" is interesting, he said; it really means "United States". Naturally, there is now a [CNNVD](<http://www.cnnvd.org.cn/>) maintained in China as well; it has more stuff and responds more quickly, but once an entry lands there it is never updated. 

#### CVE problems

There are a number of problems with CVE numbers, Kroah-Hartman said; he didn't have time to go through the full set listed in [his slides [SlideShare]](<https://www.slideshare.net/ennael/kernel-recipes-2019-cves-are-dead-long-live-the-cve>). To begin with, the database is incomplete, with many vulnerabilities missing altogether or rejected for a variety of reasons. Even when CVE numbers are assigned for a vulnerability, the process tends to take a long time and updating the NVD takes even longer. 

A big problem, he said, is that the system is run by the US government. People tend not to trust governments in general, and other governments are increasingly distrustful of the US government in particular. The system is erratically funded by the Department of Homeland Security, and is significantly underfunded overall. People need to trust that this sort [![\[Greg
Kroah-Hartman\]](https://static.lwn.net/images/conf/2019/kr/GregKroah-Hartman-sm.jpg)](<https://lwn.net/Articles/801171/>) of vulnerability database will not leak information, but government-run systems are subject to a number of pressures. During a Senate hearing on Meltdown and Spectre, Senators pressed the NVD representatives on why the Senate had not been notified about the vulnerabilities ahead of time, for example. Kroah-Hartman said that he trusts MITRE to run the NVD, but that the number of governmental representatives wanting early access to data is worrisome. 

Another problem is complexity. There is a single CVE entry ([CVE-2017-5753](<https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-5753>)) for Spectre version 1, but there are over 100 patches addressing it, and more are still coming. A CVE number doesn't point to patches, reducing its usefulness for helping people be sure they have closed a given vulnerability. It is really not possible to handle such complex things with a single ID number, he said. 

CVE numbers are abused by security developers looking to pad their resumes. As a result, a lot of "stupid things" are submitted for CVE numbers, and getting the invalid ones revoked is difficult. As an example, he gave [CVE-2019-12379](<https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2019-12379>), which was published on May 27\. It refers to an alleged memory leak in the console driver, one that, Kroah-Hartman said, poses no security threat at all. In fact, it wasn't even a leak, in the end. Even so, the NVD gave the report a security score of "medium" the day after it was submitted. Shortly thereafter the report was disputed, and it turned out that the "fix" introduced a real memory leak of its own. On June 4, Ben Hutchings [reverted](<https://git.kernel.org/linus/15b3cd8ef46>) the patch. 

One might think that the story was over at that point, but the CVE entry was only marked "disputed" in July. Distributions like Fedora have policies that require them to ship fixes for all CVE numbers, so they shipped the buggy patch in the meantime. Cleaning everything up took rather longer. This issue was eventually dealt with, but similar things happen every month — or even every week. 

Then, he said, CVE numbers are also abused by engineers to bypass internal procedures — in particular, to get their company to ship a particular patch in a product update. Getting a CVE number is a good way to force a patch into an enterprise kernel, for example. Between 2006 and 2018, he said, there were 1005 kernel CVE numbers assigned. Of those, 414 (40%) had a negative "fix date", with the average fix happening 100 days prior to the CVE-number request. Many of these are just worthwhile fixes that couldn't be merged into a shipping kernel without a CVE number behind them. He summarized by saying that this shows that CVE numbers don't really matter; they no longer carry any useful information. 

#### Bug fixes

The kernel community is currently pulling about 22 bug fixes per day into the stable trees; that is about 5% of the volume going into the mainline kernel, he said, and it should be higher. There are one or two stable-kernel releases each week. Each stable kernel is tested as a unified release and given away for free. The kernel developers are fixing about one known security problem per week, along with a vast number of other bugs that are not known to be security issues when they are fixed. All of these fixes are handled in the same way; "a bug is a bug", he said. 

He mentioned a TTY fix that was understood, after three years, to close a serious vulnerability. He was the author of both the original code and the fix, and he hadn't realized that there was a security problem in the code. Users of enterprise kernels were vulnerable to this issue for three years, he said; those who were running the stable kernels were not. Only a small portion of kernel security fixes are assigned CVE numbers; anybody who is only cherry-picking CVE-labeled fixes is thus running an insecure system. Even fixes with CVE numbers often have followup fixes that are not documented. 

He has audited a number of kernels for phones, he said. One popular handset was running 4.14.85, with three-million added lines of out-of-tree code ("what could possibly go wrong?"). If you compare that with the 4.14.108 stable release that was current in May when this analysis was done, the phone was 1,759 patches behind. The handset vendor had cherry-picked 36 patches from later kernels, but had missed twelve fixes with CVE numbers, and crucial bug fixes across the kernel tree. As a result, this phone can be crashed (or worse) by a remote attacker. 

The Google security team, he said, has a "huge tool" that scours the net for security reports. In 2018, every reported problem was already fixed in the long-term stable kernels before they found it; the only exceptions were for problems in out-of-tree code. There was no need for cherry-picking at all; anybody using those kernels was already secure against known issues. As a result, Google is now requiring Android vendors to use the long-term stable kernels. He called out Sony and Essential as being especially good at picking up new kernel releases; the Pixel devices are lagging a bit, he said, but are "basically there". 

There are, he said, 2.5 billion instances of Linux running on Android phones; that is where Linux runs now. All other users are a drop in the bucket in comparison. So this is where security matters the most; if these devices keep up with the stable-kernel releases, they will be secure, he said. 

#### How to fix CVE numbers

Kroah-Hartman put up a slide showing possible "fixes" for CVE numbers. The first, "ignore them", is more-or-less what is happening today. The next, option, "burn them down", could be brought about by requesting a CVE number for every patch applied to the kernel. It would be "a horrible intern job for six months", he said, and somebody has even offered to fund such a position. But we know that the system is broken; abusing it will not make things better. Thus, the third option: "make something new". 

The requirements for a replacement are fairly well understood. It would need to provide a unique identifier for vulnerabilities, just like CVE numbers are meant to. The system should be distributed, though; asking for identifiers from others doesn't work. It needs to be updatable over time, searchable, and public. 

Consider, he said, [commit 7caac62ed](<https://git.kernel.org/linus/7caac62ed>), which was applied in August. The changelog for this commit cites no less than three CVE numbers. The kernel community insists that developers break down their changes into simple patches, but this fix for three CVE numbers was still acceptable as a single patch. It really is a single issue, he said, that is better identified by the ID of the patch that fixed it than any of the three CVE numbers attached to it. He ran through a number of other patches, many of which included commit IDs as a way of identifying what was being fixed, usually in a "Fixes" tag. The use of those IDs in this way, he said, has become nearly universal in the kernel community. 

Thus, he said, fixes already contain a unique ID: the "Fixes" tag showing where the problem was introduced. That ID could be used as the unique ID for a vulnerability; there is no need to introduce another one. We have, in fact, been using commit IDs this way for 14 years, and nobody has noticed. All that remains to be done is to get some marketing for this scheme. After all, CVE numbers are essentially marketing, telling a story about a particular vulnerability; this new scheme needs something similar. 

The first thing that is needed to start the marketing effort, he said, is a catchy name. He ran through some possibilities, including Linux Git Kernel ID (LGKI), Kernel Git ID (KGI), or Git Kernel Hash (GKH). He paused for laughter at that last acronym (which is also his initials) before moving on. In the end, he said, the best name to use is "change ID" — the name we've been using for the last 14 years. A change ID is a world-wide, unique ID that works today, so let's use it. The format would look something like CID-0123456789ab. 

Kroah-Hartman concluded by returning to his list of things to do about CVE numbers. We should indeed "ignore CVEs", but he supplemented the list with a fourth entry: rebrand what we have been doing all along. 

#### Questions

Dmitry Vyukov led off the questions by asking about the claim that stable kernel releases are fully tested. Subsequent stable releases fix a lot more stuff, he said, so how, exactly, is that testing happening? Kroah-Hartman answered that the kernel certainly has problems with too many bugs. The stable releases in particular, though, benefit from a lot of effort to avoid regressions; he claimed that only 0.01% of the patches going into stable kernels cause regressions now. 

Vyukov answered that he is not seeing any tests being added for bugs found by his syzkaller testing. So how can the community actually prevent regressions? The answer was that we certainly need more tests. 

Your editor had to question the 0.01% figure, since [some analysis](<https://lwn.net/Articles/692866/>) done a few years ago showed a rate closer to 2%. Kroah-Hartman said that the number came from the Chrome OS team, which was counting "noticeable regressions". 

The final question was about users who are stuck with vendor kernels that will not be upgraded; what are they to do? Kroah-Hartman responded that this is a real problem. Those vendors typically add about three-million lines of code to their kernels, so they are shipping a "Linux-like system". The answer is to force vendors to get their code upstream; to do that, customers have to push back. Sony, in particular, has been insisting that its vendors have their code in the mainline kernel. That is how we solved the problem for servers years ago; it is still the approach to use today. 

[Your editor thanks the Linux Foundation, LWN's travel sponsor, for supporting his travel to this event.]
