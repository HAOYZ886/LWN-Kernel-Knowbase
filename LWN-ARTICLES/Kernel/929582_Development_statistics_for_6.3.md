---
title: Development statistics for 6.3
url: https://lwn.net/Articles/929582/
date: "April 24, 2023"
category: "Releases-6.3"
author: "By Jonathan Corbet April 24, 2023"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
April 24, 2023

The 6.3 kernel was [released](<https://lwn.net/ml/linux-kernel/CAHk-=wg02PoScxDO0wwD5EkFpx50DF1c2TxXqyAnzGjdFf71jw@mail.gmail.com/>) on April 24 after a nine-week development cycle. As is the case with all mainline releases, this is a major kernel release with a lot of changes and a big pile of new features. The time has come, yet again, for a look at where that work came from and who supported it. 

The 6.3 development cycle saw the merging of 14,424 non-merge changesets from 1,971 developers, which is a bit of a slowdown from 6.2. Of those developers, 250 made their first kernel contribution for this release. The work merged for 6.2 deleted over 513,000 lines of code — far more than the usual — but the kernel still grew by over 131,000 lines. 

The most active developers in this cycle were: 

> Most active 6.3 developers  
> ---  
> | By changesets  
> ---  
> Krzysztof Kozlowski | 387| 2.7%  
> Dmitry Baryshkov | 317| 2.2%  
> Arnd Bergmann | 185| 1.3%  
> Andy Shevchenko | 175| 1.2%  
> Christoph Hellwig | 167| 1.2%  
> Uwe Kleine-König | 163| 1.1%  
> Konrad Dybcio | 118| 0.8%  
> Sean Christopherson | 113| 0.8%  
> Martin Kaiser | 113| 0.8%  
> Chuck Lever | 109| 0.8%  
> Hans de Goede | 104| 0.7%  
> Johan Hovold | 99| 0.7%  
> Thomas Zimmermann | 99| 0.7%  
> Ville Syrjälä | 98| 0.7%  
> Mark Brown | 97| 0.7%  
> Vladimir Oltean | 96| 0.7%  
> Greg Kroah-Hartman | 96| 0.7%  
> Randy Dunlap | 95| 0.7%  
> Jakub Kicinski | 93| 0.6%  
> Jonathan Cameron | 92| 0.6%  
> | By changed lines  
> ---  
> Arnd Bergmann | 160437| 16.4%  
> Kalle Valo | 53435| 5.5%  
> Greg Kroah‑Hartman | 52609| 5.4%  
> Hans Verkuil | 28249| 2.9%  
> Cai Huoqing | 19975| 2.0%  
> Wenjing Liu | 18159| 1.9%  
> Thierry Reding | 13698| 1.4%  
> Dmitry Baryshkov | 12724| 1.3%  
> Trevor Wu | 12633| 1.3%  
> Abel Vesa | 11843| 1.2%  
> Jakub Kicinski | 11591| 1.2%  
> Krzysztof Kozlowski | 9418| 1.0%  
> Steen Hegelund | 9124| 0.9%  
> Jacek Lawrynowicz | 8802| 0.9%  
> Herbert Xu | 7601| 0.8%  
> Ondrej Zary | 7584| 0.8%  
> Shazad Hussain | 7438| 0.8%  
> Herve Codina | 7032| 0.7%  
> Bjorn Andersson | 6943| 0.7%  
> Neil Armstrong | 6769| 0.7%  
  
This is the fourth release in a row where Krzysztof Kozlowski appears in the top two changeset contributors; he continues his work with devicetree files. Dmitry Baryshkov worked extensively on a number of Qualcomm device drivers. Among other things, Arnd Bergmann removed a lot of old architecture and device-support code. Andy Shevchenko contributed cleanups across large parts of the driver tree, and Christoph Hellwig continues to refactor code in the block and filesystem areas. 

In the changed-lines column, Bergmann's removal work got rid of just over 158,000 lines of code. Kalle Valo added a new Qualcomm WiFi driver. Greg Kroah-Hartman worked throughout the device-driver tree and removed the unneeded r8188eu driver from the staging tree. Hans Verkuil removed a number of old media drivers, and Cai Huoqing removed a set of obsolete graphics drivers. 

The top testers and reviewers this time around were: 

> Test and review credits in 6.3  
> ---  
> | Tested-by  
> ---  
> Daniel Wheeler | 134| 8.2%  
> Philipp Hortmann | 112| 6.9%  
> Ulf Hansson | 44| 2.7%  
> Tony Lindgren | 44| 2.7%  
> Scott Mayhew | 41| 2.5%  
> Niklas Schnelle | 34| 2.1%  
> Gurucharan G | 34| 2.1%  
> Andrew Halaney | 33| 2.0%  
> Florian Fainelli | 23| 1.4%  
> Mingming Su | 23| 1.4%  
> | Reviewed-by  
> ---  
> Konrad Dybcio | 352| 4.0%  
> Krzysztof Kozlowski | 225| 2.5%  
> Rob Herring | 146| 1.6%  
> Simon Horman | 142| 1.6%  
> Christoph Hellwig | 133| 1.5%  
> Laurent Pinchart | 126| 1.4%  
> AngeloGioacchino Del Regno | 124| 1.4%  
> Linus Walleij | 118| 1.3%  
> Dmitry Baryshkov | 108| 1.2%  
> Hans de Goede | 103| 1.2%  
  
Daniel Wheeler and Philipp Hortmann are reliably the top testers, regularly adding their tags to Realtek and AMD graphics driver patches, respectively. Ulf Hansson and Tony Lindgren, instead, both tested many of the same patches to the cpuidle subsystem. On the review side, Konrad Dybcio reviewed 352 patches to Qualcomm drivers — at a rate of nearly six patches for every day of the development cycle, weekends and holidays included. Kozlowski and Rob Herring both focused mainly on devicetree patches. 

This time around, 1,358 patches (9.4% of the total) had Tested-by tags, while 6,902 (47.9%) had Reviewed-by tags. The increase in the number of patches with Reviewed-by tags noted [in the 6.2 development-statistics article](<https://lwn.net/Articles/923410/>) continues with 6.3. 

A total of 220 employers (that could be identified) supported work on 6.3, a slight drop from 6.2. The most active employers were: 

> Most active 6.3 employers  
> ---  
> | By changesets  
> ---  
> Linaro| 1752| 12.1%  
> Intel| 1416| 9.8%  
> Red Hat| 1013| 7.0%  
> (Unknown)| 957| 6.6%  
> Google| 840| 5.8%  
> (None)| 686| 4.8%  
> AMD| 601| 4.2%  
> IBM| 460| 3.2%  
> NVIDIA| 455| 3.2%  
> Huawei Technologies| 413| 2.9%  
> Oracle| 393| 2.7%  
> Meta| 363| 2.5%  
> SUSE| 320| 2.2%  
> (Consultant)| 300| 2.1%  
> Pengutronix| 265| 1.8%  
> Renesas Electronics| 224| 1.6%  
> Qualcomm| 210| 1.5%  
> NXP Semiconductors| 201| 1.4%  
> Microchip Technology Inc.| 166| 1.2%  
> Linux Foundation| 165| 1.1%  
> | By lines changed  
> ---  
> Linaro| 236941| 24.2%  
> Qualcomm| 80099| 8.2%  
> (Unknown)| 61511| 6.3%  
> Intel| 57448| 5.9%  
> Linux Foundation| 53935| 5.5%  
> Red Hat| 50334| 5.1%  
> AMD| 38130| 3.9%  
> NVIDIA| 35199| 3.6%  
> Cisco| 28249| 2.9%  
> Google| 24424| 2.5%  
> IBM| 21713| 2.2%  
> Meta| 21334| 2.2%  
> (None)| 18667| 1.9%  
> Microchip Technology Inc.| 17778| 1.8%  
> MediaTek| 17113| 1.8%  
> Oracle| 12501| 1.3%  
> (Consultant)| 11013| 1.1%  
> Bootlin| 8681| 0.9%  
> SUSE| 7865| 0.8%  
> Renesas Electronics| 6893| 0.7%  
  
Linaro continues its longstanding trend of increasing its contributions over time. In general, though, this table looks about the same as it always does. 

Of course, not all companies contribute to the kernel in the same way; each has its own reasons for contributing, and those reasons will drive the work that is done. Some insight can perhaps be gained by looking at which companies dominate in which parts of the kernel. For the following analysis, contributions merged after the 5.17 release were considered, giving just over one year of history. 

During that period, 89,392 non-merge changesets landed in the mainline. Of those, 12,579 (14%) touched files in `arch/`, while 48,132 (54%) touched files in either `drivers/` or `sound/` — together reflecting work to support specific hardware. The top employers working in those areas were: 

> Most active employers, 5.18 to 6.3  
> ---  
> | Architecture subsystems  
> ---  
> Linaro| 1941| 15.4%  
> Google| 1359| 10.8%  
> IBM| 1050| 8.3%  
> (Unknown)| 789| 6.3%  
> Intel| 638| 5.1%  
> (None)| 569| 4.5%  
> Red Hat| 529| 4.2%  
> Arm| 430| 3.4%  
> Renesas Electronics| 324| 2.6%  
> CS Group| 240| 1.9%  
> | Driver subsystems  
> ---  
> Intel| 7189| 14.9%  
> AMD| 4147| 8.6%  
> (Unknown)| 3292| 6.8%  
> Linaro| 2667| 5.5%  
> (None)| 2437| 5.1%  
> Huawei Technologies| 2154| 4.5%  
> Red Hat| 2122| 4.4%  
> NVIDIA| 1831| 3.8%  
> Google| 1738| 3.6%  
> Pengutronix| 1430| 3.0%  
  
The list of companies working on architecture-specific support is mostly unsurprising. Linaro exists to support the Arm architecture, as does Arm itself. IBM works on the Power architecture, while Intel is focused on x86. Google might seem to a bit of an outlier, but remember that the company is active in both cloud computing and mobile devices. Google's most active contributor under `arch/` (Sean Christopherson) has seemingly been rewriting the KVM subsystem on his own, while many other Google developers work on Arm support. 

Intel and AMD naturally dominate on the drivers side; supporting their GPUs alone brings a lot of changes into the kernel. 

The filesystem and block layers are another area of interest; 6,037 changesets (7% of the total) touched these areas. The core kernel (somewhat arbitrarily defined as the `kernel/` and `mm/` directories), instead, saw only 4,682 changes — 5% of the total — during this time. 

> Most active employers, 5.18 to 6.3  
> ---  
> | Filesystem and block layer  
> ---  
> Red Hat| 877| 14.5%  
> SUSE| 859| 14.2%  
> Oracle| 706| 11.7%  
> Meta| 614| 10.2%  
> Huawei Technologies| 551| 9.1%  
> (Consultant)| 456| 7.6%  
> (Unknown)| 279| 4.6%  
> Google| 235| 3.9%  
> Microsoft| 218| 3.6%  
> Alibaba| 150| 2.5%  
> | Core kernel  
> ---  
> Google| 575| 12.3%  
> Oracle| 537| 11.5%  
> Huawei Technologies| 468| 10.0%  
> Red Hat| 456| 9.7%  
> Meta| 421| 9.0%  
> Intel| 293| 6.3%  
> (Unknown)| 206| 4.4%  
> (None)| 183| 3.9%  
> ByteDance| 143| 3.1%  
> Amazon.com| 136| 2.9%  
  
The filesystem and block patches came primarily from distributors and companies that run massive data centers of their own. The core-kernel list is similar, but the distributors are less active in that part of the kernel. 

Another significant part of the kernel is the networking subsystem. A huge amount of work enters the kernel through the networking tree during each merge window, but only 4,168 changesets (just under 5% of the total) touched core networking; most of the rest applied to the network-interface drivers. Finally, there is the all-important Documentation directory, with the devicetree (`Documentation/devicetree`) files excluded. 

> Most active employers, 5.18 to 6.3  
> ---  
> | Networking  
> ---  
> Intel| 509| 12.2%  
> Red Hat| 498| 11.9%  
> Google| 437| 10.5%  
> Meta| 322| 7.7%  
> (Unknown)| 317| 7.6%  
> NVIDIA| 257| 6.2%  
> Huawei Technologies| 175| 4.2%  
> NXP Semiconductors| 175| 4.2%  
> Oracle| 161| 3.9%  
> Amazon.com| 154| 3.7%  
> | Documentation  
> ---  
> (Unknown)| 263| 10.8%  
> (None)| 251| 10.3%  
> Google| 233| 9.6%  
> Intel| 214| 8.8%  
> Red Hat| 142| 5.8%  
> Meta| 126| 5.2%  
> Loongson| 123| 5.1%  
> Huawei Technologies| 88| 3.6%  
> AMD| 64| 2.6%  
> Amazon.com| 55| 2.3%  
  
The presence of companies like Red Hat, Google, and Meta in the networking list is not particularly surprising, but one might wonder about a couple of the others. Fully half of Intel's contribution to the networking subsystem comes in the form of Johannes Berg's WiFi work. NVIDIA, instead, found its way into this subsystem by way of its acquisition of Mellanox in 2020. 

The Documentation numbers, instead, show a high proportion of developers who are not affiliated with any employer at all. This might be interpreted to mean that companies are relatively reluctant to pay developers to work on documentation; it also reflects the fact that documentation is a common starting place for new developers. 

There ~~is exactly one company — Google —~~ are exactly two companies — Google and Red Hat — that spread their contributions widely enough to appear on all of the above lists. 

While the reasons driving contributions to the kernel vary; that work all adds up to an impressive body of work, with regular releases every nine or ten weeks. This work looks set to continue in the near future; as of this writing there are just over 12,000 changesets waiting in linux-next for the 6.4 development cycle. Look here for an update on that work once the 6.4 cycle completes.
