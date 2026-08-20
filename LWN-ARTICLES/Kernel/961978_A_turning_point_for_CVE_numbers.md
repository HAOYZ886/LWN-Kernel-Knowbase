---
title: A turning point for CVE numbers
url: https://lwn.net/Articles/961978/
date: "February 14, 2024"
category: "Security-CVE numbers"
author: "By Jonathan Corbet February 14, 2024"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
February 14, 2024

The [Common Vulnerabilities and Exposures](<https://cve.mitre.org/>) (CVE) system was set up in 1999 as a way to refer unambiguously to known vulnerabilities in software. That system has found itself under increasing strain over the years, and numerous projects have responded by trying to assert greater control over how CVE numbers are assigned for their code. On February 13, though, a big shoe dropped when the Linux kernel project [announced](<http://www.kroah.com/log/blog/2024/02/13/linux-is-a-cna/>) that it, too, was taking control of CVE-number assignments. As is often the case, though, the kernel developers are taking a different approach to vulnerabilities, with possible implications for the CVE system as a whole. 

CVE numbers can be useful for anybody who is interested in whether a given software release contains a specific vulnerability or not. Over time, though, they have gained other uses that have degraded the value of the system overall. Developers within software distributors, for example, have been known to file CVE numbers in order to be able to ship important patches that would, without such a number, not be accepted into a stable product release. Security researchers (and their companies) like to accumulate CVE numbers as a form of resume padding and skill signaling. CVE numbers have become a target in their own right; following [Goodhart's law](<https://en.wikipedia.org/wiki/Goodhart%27s_law>), they would appear to have lost much of their value as a result. 

Specifically, in many cases, the CVE numbers resulting from these activities do not correspond to actual vulnerabilities that users need to be worried about. The assignment of a CVE number, though, imposes obligations on the project responsible for the software; there may be pressure to include a fix, or developers may have to go through the painful and uncertain process of contesting a CVE assignment and getting it nullified. As this problem has worsened, frustration with the CVE system has grown. 

One outcome of that has been an increasing number of projects applying to become the CVE Numbering Authority (CNA) for their code. If a CNA exists for a given program, all CVE numbers for that program must be issued by that CNA, which can decline to issue a number for a report that, in its judgment, does not correspond to a real vulnerability. Thus, becoming the CNA gives projects a way to stem the flow of [bogus CVE numbers](<https://lwn.net/Articles/944209/>). In recent times, a number of projects, including [curl](<https://daniel.haxx.se/blog/2024/01/16/curl-is-a-cna/>), [PostgreSQL](<https://www.postgresql.org/about/news/postgresql-is-now-a-cve-numbering-authority-cna-2790/>), the [GNU C Library](<https://lwn.net/ml/oss-security/ec94d28c-dbf4-4506-98e3-313c38ae15d5@redhat.com/>), [OpenNMS](<https://www.opennms.com/en/blog/2022-08-30-opennms-is-now-a-cna/>), [Apache](<https://www.cve.org/PartnerInformation/ListofPartners/partner/apache>), [Docker](<https://www.docker.com/blog/docker-becomes-mitre-cna/>), the [Document Foundation](<https://blog.documentfoundation.org/blog/2019/03/15/mitre-names-the-document-foundation-as-a-cve-numbering-authority-cna/>), [Kubernetes](<https://github.com/kubernetes/committee-security-response/blob/main/cna-handbook.md>), [Python](<https://policies.python.org/python.org/CVE-Numbering-Authority/>), and many others have set up their own CNAs. The OpenSSF has provided [a guide to becoming a CNA](<https://openssf.org/blog/2023/11/27/openssf-introduces-guide-to-becoming-a-cve-numbering-authority-as-an-open-source-project/>) for other projects that might be interested in taking that path. 

Corporations, too, can become the CNA for their products. Many companies want that control for the same reasons that free-software projects do; they grow tired of responding to frivolous CVE-number assignments and want to put and end to them. Of course, control over CVE assignments could be abused by a company (or a free-software project) to try to sweep vulnerabilities under the rug. There is [an appeal process](<https://www.cve.org/ResourcesSupport/AllResources/CNARules#section_9_appeals_process>) that can be followed in such cases. 

#### The kernel CNA

The kernel project has, for the most part, declined to participate in the CVE game. Famously, the project (or, at least, some of the most influential developers within it) [has long taken the position](<https://lwn.net/Articles/290227/>) that all bugs are potentially security issues, so there is [no point](<https://lwn.net/Articles/704231/>) in making a fuss over the fixes that have been identified by somebody as having security implications. Still, the kernel has proved fertile ground for those who would pad their resumes with CVE credits, and that grates on both developers and distributors. 

The situation has now changed, and the kernel will be assigning CVE numbers for itself. If that idea brings to mind a group of grumpy, beer-drinking kernel developers reviewing and rejecting CVE-number requests, though, then a closer look is warranted. The key to how this is going to work can be found in [this patch](<https://lwn.net/ml/linux-kernel/2024021430-blanching-spotter-c7c8@gregkh/>) to the kernel's documentation: 

> As part of the normal stable release process, kernel changes that are potentially security issues are identified by the developers responsible for CVE number assignments and have CVE numbers automatically assigned to them. These assignments are published on the linux-cve-announce mailing list as announcements on a frequent basis. 
> 
> Note, due to the layer at which the Linux kernel is in a system, almost any bug might be exploitable to compromise the security of the kernel, but the possibility of exploitation is often not evident when the bug is fixed. **Because of this, the CVE assignment team is overly cautious and assign CVE numbers to any bugfix that they identify**. This explains the seemingly large number of CVEs that are issued by the Linux kernel team. 

(Emphasis added). What this text is saying is that anything that looks like a bug fix — meaning many of the changes that find their way into the stable kernel updates — will have a CVE number assigned to it. Bear in mind that, as of 6.1.74, the 6.1 kernel (which has been out for just over one year) has had 12,639 fixes applied to it. The 4.19 kernel, as of 4.19.306, has had 27,952. Not all of these patches will get CVE numbers, but many will. So there are going to be a _lot_ of CVE numbers assigned to the kernel in the coming years. 

Back in 2019, LWN [covered a talk by Greg Kroah-Hartman](<https://lwn.net/Articles/801157/>) about the CVE-number problem. From that article: 

> Kroah-Hartman put up a slide showing possible "fixes" for CVE numbers. The first, "ignore them", is more-or-less what is happening today. The next, option, "burn them down", could be brought about by requesting a CVE number for every patch applied to the kernel. 

It would appear that, nearly five years later, a form of the "burn them down" option has been chosen. The flood of CVE numbers is going to play havoc with policies requiring that shipped software contain fixes for all CVE numbers filed against it — and there are plenty of policies like that out there. Nobody who relies on backporting fixes to a non-mainline kernel will be able to keep up with this CVE stream. Any company that is using CVE numbers to select kernel patches is going to have to rethink its processes. 

A couple of possible outcomes come to mind. One is that the CVE system will be overwhelmed and eventually abandoned, at least with regard to the kernel. There was not much useful signal in kernel CVE numbers before, but there will be even less now. An alternative is that distributors will simply fall back on shipping the stable kernel updates which, almost by definition, will contain fixes for every known CVE number. That, for example, is the result that Kees Cook [seemed to hope for](<https://fosstodon.org/@kees/111925962939008804>): 

> I'm excited to see this taking shape! It's going to be quite the fire-hose of identifiers, but I think that'll more accurately represent the number of fixes landing in stable trees and how important it is for end users to stay current on a stable kernel. 

It is easy to get the sense, though, that either outcome would be acceptable to the developers in charge of mainline kernel security. 

However it plays out, it is going to be interesting to watch; popcorn is recommended. The CVE system has been under increasing stress for years, and it hasn't always seemed like there has been much interest in fixing it. The arrival of the kernel CNA will not provide that fix, but it may reduce the use of kernel CVE numbers as resume padding or ways to work around corporate rules and, perhaps, draw attention to the fact that keeping a secure kernel requires accepting a truly large number of fixes. That might just be a step in the right direction.
