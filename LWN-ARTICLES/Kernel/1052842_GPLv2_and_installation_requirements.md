---
title: GPLv2 and installation requirements
url: https://lwn.net/Articles/1052842/
date: "January 8, 2026"
category: Copyright issues
author: "By Jonathan Corbet January 8, 2026"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
January 8, 2026

On December 24 2025, Linus Torvalds posted [a strongly worded message](<https://social.kernel.org/notice/B1aR6QFuzksLVSyBZQ>) celebrating a [ruling](<https://lwn.net/Articles/1051984/>) in the ongoing [GPL-compliance lawsuit](<https://lwn.net/Articles/1052734/>) filed against VIZIO by the Software Freedom Conservancy (SFC). This case and Torvalds's response have put a spotlight on an old debate over the extent to which the source-code requirements of the [GNU General Public License (version 2)](<https://www.gnu.org/licenses/old-licenses/gpl-2.0.en.html>) extend to keys and other data needed to successfully install modified software on a device. It is worth looking at whether this requirement exists, the subtleties in interpretation that cloud the issue, and the extent to which, if any, the SFC is demanding that information. 

#### Tivoization

It is increasingly common for computing systems to refuse to boot and run software that is lacking an authorized signature. There are many legitimate reasons to want this feature; a company may want to ensure that its internal systems have not been tampered with, or a laptop owner may want defense against [evil maid attacks](<https://en.wikipedia.org/wiki/Evil_maid_attack>). There are also numerous cases of companies using this feature to protect their own business models. The practice of locking down devices running otherwise free software so that their owners cannot change that software has become known as "tivoization", after Tivo used it to control access to its digital video recorders. 

Tivoization sits poorly with many free-software developers, who see it as a way of using copyleft software without providing all of the freedoms that should come with it. The practice enables devices built around antifeatures that, in a more free world, users would simply remove. But, while there is no universal agreement on whether this practice violates version 2 of the GPL, the larger contingent would appear to be on the side of "no". It is noteworthy that the GNU project, while [criticizing tivoization](<https://www.gnu.org/philosophy/tivoization.en.html>), does not claim that it is a GPLv2 violation. When version 3 of the GPL was drafted, a special effort was made to add language disallowing tivoization, but many projects, including the kernel, have never adopted that version of the license. 

#### Keys in the VIZIO case

At the end of 2025, the judge in the VIZIO case ruled that provision of signing keys is not required by the GPL. Specifically, the ruling said that the GPL's source-code requirements do not extend to materials required to install a modified version of the software on the original device and have it ""continue to function properly"". The SFC was quick to put out [a press release](<https://sfconservancy.org/news/2025/dec/24/vizio-msa-irrelevant-ruling/>) saying that the ruling addressed an issue that was not actually before the court — that the SFC had not been asking for that ability. The truth of the matter would appear to be a bit more nuanced, though. 

[The SFC's complaint](<https://usethesource.sfconservancy.org/tmp_vizio_docs/software-freedom-conservancy-v-vizio-first_amended_complaint-2024-01-10.pdf>) driving this lawsuit goes into a fair amount of detail about why access to the source code is important. Examples include claim 111: 

> With the source code for the SmartCast Works at Issue as used on the Subject TVs, developers could continue to develop and improve an operating system for smart televisions, which would benefit the public and further the goals of software freedom. 

And claim 118: 

> Access to the Source Code of the Linux kernel, the other SmartCast Works at Issue, and for the Library Linking Programs, as used on VIZIO smart TVs, would enable software developers to preserve useful but obsolete features. It would also allow software developers to maintain and update the operating system should VIZIO or its successor ever decide to abandon it or go out of business. It would also allow for the maintenance of older models that are no longer supported by VIZIO. In these ways, purchasers of VIZIO smart TVs can be confident that their devices would not suffer from software-induced obsolescence, planned or otherwise. 

The complaint also goes into some detail about how VIZIO makes far more money from sales of advertising and data about its customers than it does from selling televisions. Supporting the company's real business model requires monitoring what owners of the devices are doing and selling that information to others. With access to the source, developers could remove the surveillance features built into VIZIO devices, improving their privacy and security. The complaint notes that VIZIO seems unlikely to make it possible to disable those features on its own. 

All of these assertions are fairly obviously true, and there is no doubt that the owners of VIZIO devices would benefit from the ability to make the described changes. That is what the freedom of free software is all about. But there is a catch: access to the source will provide none of those freedoms without the ability to install modified software on the device — and have the device actually run that software. The SFC's complaint does not mention signing keys specifically, but it describes freedoms that cannot be exercised without those keys. 

#### Reinstallation requirements

So why does the SFC claim that the court's ruling is irrelevant? There are, as it turns out, multiple ways to look at what it means to install software on a device. Specifically, should a modified software load still be able to provide all of the functionality that the original did? VIZIO's software, for example, surely contains proprietary components that allow the device to play DRM-protected content. Allowing a user-modified kernel to access those components would enable the creation of a channel to export unprotected content from the device, thus accelerating the escape of that content onto the Internet by several milliseconds. For some reason, VIZIO feels that this outcome should be avoided. 

The SFC states that it never asked for this capability, that, specifically, nobody is claiming that the device must continue to provide all of its functionality if modified software is installed. At the other extreme, though, while a device that refuses to boot because the kernel lacks the requisite signature surely fits within the description of "not continuing to function properly", the SFC appears to find that result insufficient. The level of functionality that must be available to user-modified software has not been clearly defined, but it seems to involve functionality beyond that needed for a doorstop. 

What the SFC may be asking for is something akin to the Chromebook developer mode. A Chromebook can either be locked down, in which case all supported functionality is available, or it can be in developer mode, which allows user-installed software but loses access to some features. Similarly, some Android devices allow their software to be replaced, but they lose the ability to attest to the purity of their software and some apps may refuse to run. If VIZIO televisions had a developer mode that allowed users to install software with reduced functionality, that would seemingly satisfy the SFC's requests. VIZIO, though, did not see fit to design that feature into its products. 

That said, the SFC _did_ argue that the GPL at least _might_ have a ""reinstallation requirement"" that preserves the functionality of the device; it is worth understanding why. As noted above, the SFC's complaint assumes that it would be possible to install modified software on the system. As a way of heading off the threat to its business model, VIZIO [asked for a summary judgment](<https://sfconservancy.org/static/docs/2025-05-02_SFC-vs-Vizio_second-Vizio-Motion-for-Summary-Adjudication.pdf>) that no such requirement exists in the GPL: 

> VIZIO moves on the grounds that the plain language of GPLv2 and LGPLv2.1 or, in the alternative, the undisputed extrinsic evidence, compels the conclusion that neither license imposes a duty on licensees to provide all information necessary to permit reinstallation of modified software back on the same device such that the device continues to function properly. 

Note that this motion was brought forward as a way of requesting that a significant portion of the SFC's case be thrown out of court. A failure to respond to the motion in the strongest possible way would have been an act of malpractice on the part of the SFC's lawyers; they had to challenge it. So, even though the SFC claims to have never brought in the "the device continues to function properly" language, it found itself having to defend that language anyway. That defense came in [this filing](<https://sfconservancy.org/static/docs/2025-07-03_SFC-vs-Vizio_SFC-response-to-second-Vizio-MSA.pdf>), which included quotes from some important developers that the installation requirement does indeed exist: 

> As one member of the FOSS community [Bdale Garbee] explains, under the GPLv2, the distributor of a device that includes a computer program licensed under the GPLv2 "must provide scripts that allow a recompiled binary of the source code to be installed back onto the device in question." 

The purpose here was not to establish the existence of a reinstallation requirement; it was to show the existence of doubt on the question so that a summary judgment would be inappropriate. The judge, however, [sided with VIZIO](<https://sfconservancy.org/static/docs/2025-12-23_SFC-vs-Vizio_30-2021-01226723-CU-BC-CJC_Leal-minute-order.pdf>) and granted the motion. While the judge did not address the issue of whether installation _without_ the need for proper functioning might be required by the license, it seems unlikely that the outcome would be different. 

The approach taken by Torvalds and other (but certainly not all) kernel developers toward tivoization reflects an important tradeoff. Had the kernel moved to GPLv3 and required the ability to install modified versions, much of the industry built around Linux may well have moved on to something else. That, in turn, would lead to a significant reduction in corporate support for Linux development. 

There are certainly developers who would happily make that trade in exchange for a license that guarantees more freedom. But a kernel that is not used provides no freedom at all, and Linux without companies behind it would not be in anything close to the condition it is now. Torvalds chose, many years ago, a policy that freedom extends to the software, but not beyond; the result is a kernel that is now nearly ubiquitous. So we find ourselves in a situation where the software is capable and free, but that is not always the case for the hardware it runs on. That problem is well worth solving, but GPLv2 does not appear to be the right tool for the job.
