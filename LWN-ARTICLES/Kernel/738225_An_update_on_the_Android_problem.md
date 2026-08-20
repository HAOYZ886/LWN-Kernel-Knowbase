---
title: An update on the Android problem
url: https://lwn.net/Articles/738225/
date: "November 7, 2017"
category: Android
author: "By Jonathan Corbet November 7, 2017 2017 Maintainers Summit"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
November 7, 2017

* * *

[2017 Maintainers Summit](<https://lwn.net/Articles/KernelSummit2017/>)

Android has been a great boon to the kernel community, having brought a great deal of growth in both the user and the development communities. But Android has also been a problem in that devices running it ship with kernels containing large amounts (often millions of lines) of out-of-tree code. That fragments the development community and makes it impossible to run mainline kernels on this hardware. The problematic side of Android was discussed at the 2017 Maintainer Summit; the picture that resulted is surprisingly optimistic. 

Greg Kroah-Hartman started by saying that he has been working for some time with the system-on-chip (SoC) vendors to try to resolve this problem, which he blames primarily on Qualcomm for having decided not to work upstream. Qualcomm has since concluded that this decision was a mistake and is determined to fix it, but the process of doing so will take years. The other SoC vendors are also committed to closing the gap between the kernels they provide and the mainline but, again, getting there will take a while. 

Google's new rules requiring the use of long-term support kernels with Android and keeping up with updates should also help. If vendors do not follow those rules, he said, he will eventually stop maintaining the LTS releases. For now, though, he is running an experiment where he will support the 4.4.x kernels for a period of six years. Vendors are coming around to using those [![\[Greg
Kroah-Hartman\]](https://static.lwn.net/images/conf/2017/osse/GregKroah-Hartman-sm.jpg)](<https://lwn.net/Articles/738226/>) updates, he said, but there is a new problem in the form of carriers who are proving unwilling to ship those updates. He is trying to get carriers to put one out every six months for now. 

Rom Lemarchand, Google's Android kernel manager, said that newer devices are shipping with 4.4 kernels now. The SoC market cycle is such that these chips will always run a two-year-old kernel. The two-year support lifetime for LTS kernels thus didn't work well for SoC vendors; just about the time that they ship something, the support goes away. Hopefully the six-year support period will work better. Updates are still a problem, though; vendors still are working under the mentality that they only need to take patches that have CVE numbers attached to them, which is not the case. Kroah-Hartman added that they weren't even taking all of the patches with CVE numbers. Kees Cook said that none of the vendors have decent testing for their kernels and don't want to merge any changes at all. They don't, he said, want to admit that they are bringing in LTS patches. 

Along the lines of testing, there was some discussion of the [Linux Test Project](<https://linux-test-project.github.io/>) (LTP). This project has tended to be viewed dismissively by kernel developers, but it is evidently the recipient of more resources and has been getting better. There may eventually be value in integrating LTP into the kernel self tests. Linus Torvalds said that even an improved LTP is not that interesting compared to real workloads, though, so he would much rather see Android running on mainline kernels. This is evidently being worked on, but is not there yet. Lemarchand said that the HiKey boards are staying as close to mainline and can boot a 4.9 kernel, but Arnd Bergmann pointed out that the HiKey boards are no longer being produced. 

Somebody asked: has any Android phone ever done a major kernel upgrade after it has been shipped? That is evidently a difficult proposition, since there a number of regulatory certifications that must be redone. But the Galaxy Nexus and Galaxy S phones both saw major kernel upgrades, so it is possible. Torvalds noted that there are a lot of Android devices that are not phones, tablets for example, that might prove to be better development devices. It would be nice if mainline developers could run their own kernels on real devices. Bergmann said that the gap is shrinking on some devices, and Kroah-Hartman repeated that he is working toward this goal with the SoC vendors, but the process should be expected to take about six years. 

Cook said that applying the larger updates involved in following the LTS kernels completely should eventually make vendors more comfortable with larger kernel changes in general. Sean Paul said that running mainline kernels on Android devices may well become possible soon, but phones still probably will not jump to new major releases. Even that would be good, though, Bergmann said; the current out-of-tree code problem defeats the goal of building a single ARM kernel for all devices. Fixing that would enable third-party distributors to ship systems for multiple phones. Torvalds said that, even if vendors don't upgrade their devices, the ability to do so would enable some useful regression testing. James Bottomley said that the whole situation is a repeat of the enterprise Linux problem from many years ago. 

Ted Ts'o asked if there were any ARM Chromebooks that could be used as development machines; Paul answered that the ones based on Rockchip SoCs were close. Torvalds asked about the status of the Mali GPU driver; Bergmann responded that there had been one person working on reverse-engineering that device, but he didn't work well with other developers. Now somebody else is making progress with the older GPUs, but nobody is working on current-generation devices. It was said that everybody within ARM is in favor of solving the problem by open-sourcing ARM's driver — except for one recalcitrant high-level manager. 

Torvalds said that, if the Mali problem could be solved, the community as a whole would be in good shape. Bergmann said that there are currently four ARM GPUs with good free-software support, but they are all older. Going forward, Mali seems to be the GPU of choice for Android devices, so that is the problem that needs to be solved. Lemarchand said that pressure is being applied from the Android side as well. 

The final conclusion of this session was that, while the Android problem has not gone away, the situation is far better than it was one year ago. 

[Your editor would like to thank the Linux Foundation, LWN's travel sponsor, for supporting his travel to this event].
