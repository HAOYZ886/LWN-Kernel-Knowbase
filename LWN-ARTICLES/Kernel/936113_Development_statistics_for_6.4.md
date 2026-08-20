---
title: Development statistics for 6.4
url: https://lwn.net/Articles/936113/
date: "June 26, 2023"
category: "Releases-6.4"
author: "By Jonathan Corbet June 26, 2023"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
June 26, 2023

The 6.4 kernel was [released](<https://lwn.net/ml/linux-kernel/CAHk-=wi7fwNWfqj-QQqEfZTUOB4bbKT8QiEUDHoPk0ecuYA7cA@mail.gmail.com/>) on June 25 after a nine-week development cycle. By that point, 14,835 non-merge changesets had been pulled into the mainline kernel, a slight increase from 6.3 (14,424 changesets) but still lower than many other development cycles. As usual, LWN has taken a look at those changesets, who contributed them, and what the most active developers were up to. 

The work in 6.4 was contributed by 1,980 developers, 282 of whom made their first kernel contribution during this development cycle. The most active 6.4 developers were: 

> Most active 6.4 developers  
> ---  
> | By changesets  
> ---  
> Uwe Kleine-König | 781| 5.3%  
> Krzysztof Kozlowski | 499| 3.4%  
> Rob Herring | 200| 1.3%  
> Ian Rogers | 200| 1.3%  
> Konrad Dybcio | 146| 1.0%  
> Thomas Zimmermann | 132| 0.9%  
> AngeloGioacchino Del Regno | 126| 0.8%  
> Hans de Goede | 121| 0.8%  
> Christoph Hellwig | 118| 0.8%  
> Ville Syrjälä | 116| 0.8%  
> Tom Rix | 115| 0.8%  
> Nick Alcock | 112| 0.8%  
> Johannes Berg | 111| 0.7%  
> Darrick J. Wong | 111| 0.7%  
> Philipp Hortmann | 101| 0.7%  
> Geert Uytterhoeven | 88| 0.6%  
> Greg Kroah-Hartman | 87| 0.6%  
> Manivannan Sadhasivam | 85| 0.6%  
> Eric Dumazet | 84| 0.6%  
> Bart Van Assche | 83| 0.6%  
> | By changed lines  
> ---  
> Ian Rogers | 167443| 17.4%  
> Hawking Zhang | 123915| 12.8%  
> Eduard Zingerman | 25322| 2.6%  
> Laurent Pinchart | 17210| 1.8%  
> Ping-Ke Shih | 16062| 1.7%  
> Darrick J. Wong | 11027| 1.1%  
> Uwe Kleine-König | 10159| 1.1%  
> Benjamin Tissoires | 8623| 0.9%  
> Konrad Dybcio | 8421| 0.9%  
> Jani Nikula | 7982| 0.8%  
> Jiri Slaby | 7645| 0.8%  
> AngeloGioacchino Del Regno | 7353| 0.8%  
> Krzysztof Kozlowski | 7285| 0.8%  
> Hans de Goede | 7068| 0.7%  
> Paul Gortmaker | 7011| 0.7%  
> Tony Nguyen | 6834| 0.7%  
> Jeffrey Hugo | 6718| 0.7%  
> Wolfram Sang | 6665| 0.7%  
> Devi Priya | 6036| 0.6%  
> Qu Wenruo | 5617| 0.6%  
  
The 6.3 merge window included [a patch from Uwe Kleine-König](<https://git.kernel.org/linus/5c5a7680e67b>) adding a new function pointer to [`struct platform_driver`](<https://elixir.bootlin.com/linux/v6.3.9/source/include/linux/platform_device.h#L208>). Noting that the driver core ignores the return value from the `remove()` function, he decided to make that function return `void` instead. There are, however, many drivers defining that function — more than could be changed at that time. So, rather than changing `remove()`, he added `remove_new()`, which behaves in the same way with the exception that it returns `void`; that made it possible to convert drivers at leisure. 

"At leisure" may not describe what happened in 6.4, though, where Kleine-König contributed 781 changesets, almost all of them converting drivers to `remove_new()`. It's worth noting that we are likely to see a repeat of this performance; the plan calls for renaming `remove_new()` back to `remove()` (and updating all the drivers again) once the conversion is complete. Krzysztof Kozlowski, whose work (mostly in the devicetree subtree) would have normally put him easily into the top position, came in a distant second this time around. Ian Rogers made a number of enhancements to the `perf` tool, Rob Herring worked mostly on devicetree improvements, and Konrad Dybcio hacked on various system-on-chip drivers and devicetree files. 

Rogers also made it to the top of the "lines changed" column by contributing updated event definitions for `perf`. Hawking Zhang added the obligatory set of amdgpu register definitions. Eduard Zingerman reworked many of the BPF self-tests, adding a lot of inline assembly code; [this patch](<https://git.kernel.org/linus/4db10a8243df>) is a typical example. Laurent Pinchart deleted a number of unused camera-sensor drivers while adding the i.MX8 ISI driver, and Ping-Ke Shih added a set of static tables to the rtw89 WiFi driver. 

The top testers and reviewers this time around were: 

> Test and review credits in 6.4  
> ---  
> | Tested-by  
> ---  
> Daniel Wheeler | 159| 12.9%  
> Chen-Yu Tsai | 61| 5.0%  
> Marek Szyprowski | 36| 2.9%  
> Abhinav Kumar | 27| 2.2%  
> Sachin Sant | 21| 1.7%  
> Joel Fernandes | 20| 1.6%  
> Zqiang | 18| 1.5%  
> Tommaso Merciai | 17| 1.4%  
> Philipp Hortmann | 16| 1.3%  
> Tony Zhu | 16| 1.3%  
> Arnaldo Carvalho de Melo | 15| 1.2%  
> Marek Szlosek | 15| 1.2%  
> | Reviewed-by  
> ---  
> Simon Horman | 327| 3.9%  
> Konrad Dybcio | 208| 2.5%  
> Krzysztof Kozlowski | 197| 2.4%  
> AngeloGioacchino Del Regno | 151| 1.8%  
> David Sterba | 134| 1.6%  
> Rob Herring | 127| 1.5%  
> Chen-Yu Tsai | 118| 1.4%  
> Dmitry Baryshkov | 116| 1.4%  
> Dave Chinner | 115| 1.4%  
> Hans de Goede | 113| 1.4%  
> Christoph Hellwig | 112| 1.3%  
> Geert Uytterhoeven | 104| 1.3%  
  
For 6.4, 1,064 commits (7% of the total) carried Tested-by tags, while 6,392 (43%) had Reviewed-by tags. That is a significant drop since 6.3 for both types of tags. The Tested-by tags, in particular, clearly do not reflect the actual testing activity that is taking place in the kernel community. 

A total of 230 employers (that could be identified) supported work on 6.4, a slight increase from 6.3. The most active employers were: 

> Most active 6.4 employers  
> ---  
> | By changesets  
> ---  
> Intel| 1542| 10.4%  
> Linaro| 1505| 10.1%  
> Google| 1137| 7.7%  
> (Unknown)| 1086| 7.3%  
> Red Hat| 881| 5.9%  
> Pengutronix| 826| 5.6%  
> AMD| 624| 4.2%  
> (None)| 582| 3.9%  
> Meta| 467| 3.1%  
> Oracle| 384| 2.6%  
> NVIDIA| 381| 2.6%  
> SUSE| 355| 2.4%  
> IBM| 349| 2.4%  
> Qualcomm| 300| 2.0%  
> (Consultant)| 262| 1.8%  
> Collabora| 236| 1.6%  
> Renesas Electronics| 224| 1.5%  
> Huawei Technologies| 221| 1.5%  
> NXP Semiconductors| 195| 1.3%  
> Microsoft| 183| 1.2%  
> | By lines changed  
> ---  
> Google| 191963| 19.9%  
> AMD| 156235| 16.2%  
> Intel| 73179| 7.6%  
> (Unknown)| 67648| 7.0%  
> Linaro| 42260| 4.4%  
> Red Hat| 39293| 4.1%  
> Qualcomm| 28096| 2.9%  
> (None)| 23696| 2.5%  
> (Consultant)| 23488| 2.4%  
> SUSE| 23393| 2.4%  
> Meta| 20978| 2.2%  
> Realtek| 20830| 2.2%  
> NVIDIA| 19050| 2.0%  
> Oracle| 16942| 1.8%  
> IBM| 15768| 1.6%  
> Renesas Electronics| 14519| 1.5%  
> Pengutronix| 11598| 1.2%  
> MediaTek| 10956| 1.1%  
> Collabora| 10066| 1.0%  
> Microsoft| 9824| 1.0%  
  
As usual, there are not a lot of surprises to be found in these results. 

"Lines changed" is, like commit counts, a poor proxy for software productivity, but it's hard to find a better one. So, your editor has concluded, one might as well just go nuts with the "lines changed" metric over the long term. With the use of `git blame` and a certain amount of CPU time, it is possible to look at who is "blamed" for every line in the kernel source — who is the last developer to have touched it, in other words. 

Running this analysis on the 6.4-rc7 kernel turns up 22,612 developers who have touched at least one line — 2,135 of whom have touched exactly one line. The developers who have left the biggest imprint on the 6.4 kernel are: 

> Developer| Lines| Pct  
> ---|---|---  
> Linus Torvalds| 2159025| 5.9%  
> Alex Deucher| 1177105| 3.2%  
> Hawking Zhang| 840838| 2.3%  
> Huang Rui| 479002| 1.3%  
> Mauro Carvalho Chehab| 417086| 1.1%  
> Aurabindo Pillai| 383629| 1.1%  
> Oded Gabbay| 292611| 0.8%  
> Ian Rogers| 271905| 0.7%  
> Leo Li| 228680| 0.6%  
> Bhawanpreet Lakha| 206275| 0.6%  
> Qingqing Zhuo| 198516| 0.5%  
> Aaron Liu| 193174| 0.5%  
> Ping-Ke Shih| 184453| 0.5%  
> Larry Finger| 172346| 0.5%  
> Ben Skeggs| 170190| 0.5%  
> Roman Li| 164743| 0.5%  
> Mark Brown| 158765| 0.4%  
> David Howells| 158530| 0.4%  
> Hans de Goede| 157719| 0.4%  
> Laurent Pinchart| 145900| 0.4%  
> James Smart| 145427| 0.4%  
> Kalle Valo| 144362| 0.4%  
> Hans Verkuil| 142093| 0.4%  
> Johannes Berg| 139245| 0.4%  
> Takashi Iwai| 131543| 0.4%  
> Feifei Xu| 127510| 0.4%  
> Christoph Hellwig| 127129| 0.3%  
> Thierry Reding| 118462| 0.3%  
> Linus Walleij| 115289| 0.3%  
> David S. Miller| 97207| 0.3%  
  
Torvalds does not write a lot of kernel code these days, and hasn't for some time; his position at the top of the list is the enduring legacy of the initial Git commit of the 2.6.12 kernel in 2005. Many of the other developers on that list — Alex Deucher, Hawking Zhang, Huang Rui, Aurabindo Pillai, Leo Li, Qingqing ZHuo, Aaron Liu, and Roman Li — are there primarily as the result of having contributed amdgpu header files, though Deucher's work is quite a bit broader than that. Mauro Carvalho Chehab has left such a big footprint after many years of intensive work in the media subsystem and the conversion of much kernel documentation to the RST format. 

A few of the developers on the above list are there as the result of consistent kernel work over a period of decades, but that clearly is not the main variable being measured here. Those wanting to see more can view [the top 2,000 results](<https://lwn.net/Articles/936125>) separately. Note that no attempt has been made to join multiple entries resulting from name changes, typos, or mailing-list mangling. 

As of this writing, there are about 11,330 changesets waiting in linux-next for the 6.5 merge window. LWN will, of course, be watching as that work pours into the mainline and yet another development cycle runs its course. The kernel development community continues to run at full speed.
