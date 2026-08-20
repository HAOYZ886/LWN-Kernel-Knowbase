---
title: "What happens to kernel staging-tree code"
url: https://lwn.net/Articles/798258/
date: "September 5, 2019"
category: Staging tree
author: "By Jonathan Corbet September 5, 2019"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
September 5, 2019

The staging tree was [added to the kernel](<https://lwn.net/Articles/285594/>) in 2008 for the 2.6.28 development cycle as a way to ease the process of getting substandard device drivers into shape and merged into the mainline. It has been followed by controversy for just about as long. The recent disagreements over the [EROFS](<https://lwn.net/Articles/796687/>) and [exFAT](<https://lwn.net/Articles/797963/>) filesystems have reignited many of the arguments over whether the staging tree is beneficial to the kernel community or not. LWN cannot answer that question, but we can look into what has transpired in the staging tree in its first eleven years to see if there are any conclusions to be drawn there. 

The core idea behind the staging tree is that it is open to code that does not live up to the normal standards for inclusion into the kernel. Once a driver is added there, it is available to anybody who is brave enough to try to make use of it, but the real purpose is to allow developers to improve the code to the point that it is ready to go into the kernel proper. It serves as an easy place for new developers to try out simple changes and, when it works well, it helps the kernel to gain hardware support that might otherwise languish out-of-tree indefinitely. 

Not everybody agrees that putting code into the staging tree is the best way to improve it; many of the developers who are concerned about the proposal to add exFAT to the staging tree argue that it will improve more quickly outside of staging. To quote [Dave Chinner](<https://lwn.net/ml/linux-kernel/20190901010721.GG7777@dread.disaster.area/>): 

That's stuff that is much more easily done and reviewed by patches on a mailing list. You don't need the code in the staging tree to get this sort of thing done and, really, having it already merged gets in the way of doing major structural change as it cannot be rapidly iterated independently of the kernel dev cycle. 

Greg Kroah-Hartman, the maintainer of the staging tree, [replied](<https://lwn.net/ml/linux-kernel/20190902152524.GA4964@kroah.com/>) that putting the code there gives it a common place for developers to cooperate on it. He acknowledged that it may slow the process somewhat, but argued that this cost was worth it for groups that wish to cooperate in that way. 

It occurred to your editor that nobody has apparently looked at what fate has befallen the modules that have entered into staging over the last eleven years. A simple bit of scripting and brute-force Git work yielded a list of when each module entered and left the staging tree. All that was left was a huge amount of tedious manual work to figure out _why_ each module was removed from the staging tree. It was one of those projects that, about halfway through, doesn't seem like such a good idea anymore. But now it is done; results follow. 

#### Graduations

In the end, 247 modules were tracked from their entry into the staging tree. Of those, 52 "graduated" from the staging tree into the mainline kernel — 21% of the total. Those modules were: 

> Module| Entry| Exit| Releases | Fate  
> ---|---|---|---|---  
> altera-stapl | v3.0| v3.2| 3| Graduated  
> ath6kl | v2.6.37| v3.2| 5| Graduated   
> batman-adv | v2.6.33| v2.6.38| 6| Graduated   
> brcm80211 | v2.6.37| v3.2| 5| Graduated  
> ccree | v4.12| v4.17| 6| Graduated  
> cx25821 | v2.6.32| v3.2| 10| Graduated   
> dwc2 | v3.10| v3.14| 5| Graduated  
> echo | v2.6.28| v3.15| 27| Graduated  
> et131x | v2.6.28| v3.18| 30| Graduated   
> fsl-mc | v4.1| v4.19| 19| Graduated   
> gma500 | v3.0| v3.3| 4| Graduated   
> hv | v2.6.32| v3.4| 12| Graduated   
> iio/dac | v2.6.38| v3.6| 8| Graduated   
> iio/imu | v2.6.35| v3.9| 14| Graduated   
> iio/light | v2.6.32| v4.18| 46| Graduated  
> iio/magnetometer | v2.6.35| v4.6| 31| Graduated   
> imx-drm | v3.7| v3.19| 13| Graduated  
> ipack | v3.5| v3.8| 4| Graduated  
> line6 | v2.6.30| v4.0| 30| Graduated   
> media/as102 | v3.2| v3.18| 17| Graduated  
> media/cec | v4.8| v4.10| 3| Graduated   
> media/cxd2099 | v3.2| v4.17| 36| Graduated  
> media/dt3155v4l | v2.6.35| v4.2| 27| Graduated  
> media/go7007 | v2.6.28| v3.17| 29| Graduated  
> media/mn88472 | v3.19| v4.8| 10| Graduated   
> media/mn88473 | v3.19| v4.6| 8| Graduated  
> media/msi3101 | v3.12| v3.17| 6| Graduated  
> media/pulse8-cec | v4.8| v4.10| 3| Graduated  
> media/rtl2832u_sdr | v3.15| v3.17| 3| Graduated  
> media/s5p-cec | v4.8| v4.12| 5| Graduated   
> media/solo6x10 | v2.6.36| v3.17| 21| Graduated  
> media/st-cec | v4.9| v4.12| 4| Graduated  
> mei | v3.0| v3.5| 6| Graduated  
> mrst-touchscreen | v2.6.35| v2.6.37| 3| Graduated   
> mt7621-gpio | v4.17| v4.19| 3| Graduated  
> mt7621-spi | v4.17| v5.2| 7| Graduated  
> omapdrm | v3.3| v3.9| 7| Graduated  
> panel | v2.6.29| v4.6| 37| Graduated  
> rar_register | v2.6.32| v2.6.36| 5| Graduated  
> rdma/hfi1 | v4.3| v4.7| 5| Graduated   
> samsung-laptop | v2.6.33| v3.0| 7|  Graduated  
> sm7xxfb | v2.6.33| v4.3| 31| Graduated   
> ti-soc-thermal | v3.6| v3.11| 6| Graduated   
> ti-st | v2.6.35| v3.0| 5| Graduated   
> tm6000 | v2.6.35| v3.2| 7| Graduated   
> typec | v4.12| v4.19| 8| Graduated   
> udlfb | v2.6.31| v2.6.38| 8| Graduated   
> usbip | v2.6.28| v3.17| 29| Graduated   
> vboxvideo | v4.13| v5.2| 11| Graduated   
> xillybus | v3.12| v3.18| 7| Graduated   
> zram | v2.6.33| v3.14| 21| Graduated   
> zsmalloc | v3.4| v3.14| 11| Graduated   
  
(Lest somebody complain that your editor's math is off, the "releases" column includes the release in which the module was removed from the staging tree, since it was a part of that development cycle). 

Some modules clearly pass through staging more quickly than others do. The `altera-stapl` driver zipped through in three release cycles. On the other hand, the industrial I/O `light` module sat there for 46 development cycles — nearly nine years. Regardless of how long it took, though, each of those modules was improved to the point that the appropriate subsystem maintainer was willing to accept it. 

A 21% success rate may not seem all that high, but that is not the full story. A surprising number of staging drivers were deleted because their functionality was provided by another driver in the mainline. These "superseded" modules were: 

> Module| Entry| Exit| Releases | Fate  
> ---|---|---|---|---  
> adis16255 | v2.6.35| v2.6.38| 4| Superseded  
> android/switch | v3.3| v3.5| 3| Superseded   
> at76_usb | v2.6.28| v2.6.32| 5| Superseded   
> cpc-usb | v2.6.31| v2.6.32| 2| Superseded  
> cptm1217 | v2.6.38| v4.0| 22| Superseded   
> cs5535_gpio | v2.6.38| v3.1| 3| Superseded   
> dt3155 | v2.6.34| v2.6.36| 3| Superseded   
> me4000 | v2.6.28| v2.6.32| 5| Superseded  
> media/easycap | v2.6.36| v3.7| 11| Superseded  
> media/tw686x-kh | v4.7| v4.9| 3| Superseded  
> meilhaus | v2.6.29| v2.6.32| 4| Superseded  
> mimio | v2.6.29| v2.6.34| 6| Superseded  
> msm | v2.6.35| v3.1| 6| Superseded  
> mt29f_spinand | v3.13| v5.0| 29| Superseded  
> mt7621-eth | v4.17| v5.1| 6| Superseded  
> otus | v2.6.29| v2.6.37| 9| Superseded   
> pata_rdc | v2.6.31| v2.6.32| 2| Superseded  
> quatech_usb2 | v2.6.32| v3.5| 13| Superseded  
> rt2860 | v2.6.29| v3.0| 11| Superseded   
> rt2870 | v2.6.29| v3.0| 11| Superseded   
> rt3070 | v2.6.30| v2.6.36| 7| Superseded  
> rt3090 | v2.6.32| v2.6.34| 3| Superseded   
> rtl8187se | v2.6.29| v3.15| 26| Superseded   
> rtl8192ee | v3.16| v3.18| 3| Superseded   
> rtl8192su | v2.6.31| v2.6.37| 7| Superseded   
> rtl8723au | v3.15| v4.9| 15| Superseded   
> rtl8821ae | v3.14| v3.18| 5| Superseded   
> rtlwifi | v4.14| v5.2| 10| Superseded   
> rts5139 | v3.2| v3.16| 15| Superseded   
> rts_pstor | v3.0| v3.8| 9| Superseded   
> serqt_usb | v2.6.30| v2.6.31| 2| Superseded   
> slicoss | v2.6.28| v4.10| 42| Superseded   
> ste_rmi4 | v2.6.38| v4.6| 28| Superseded   
> stlc45xx | v2.6.30| v2.6.32| 3| Superseded   
> uc2322 | v2.6.30| v2.6.31| 2| Superseded   
> usbvideo | v2.6.38| v3.0| 2| Superseded   
> zcache | v3.0| v3.12| 13| Superseded   
  
That is 37 superseded drivers overall, or 15% of the total. Whether these modules should be considered success stories will vary on a case-by-case basis. Some staging drivers were sufficiently ugly that they motivated desperate developers to write something better from scratch; some may say that the merging of such a driver into staging has achieved its intended goal. In other cases, the companies behind the hardware finally started providing proper drivers; many of the Realtek drivers in the above list fall into that category. In yet others, the staging driver may have provided needed information that enabled a proper driver to be written — also a useful role. 

In each case, the support provided by these staging drivers did eventually find its way into the mainline. How much help was given by putting any given driver into staging is hard to determine, but the addition of this support seems like a success for the community overall. 

#### Failures

Then, there is a long list of drivers that were simply deleted from the staging tree with no equivalent functionality being provided elsewhere: 

> Module| Entry| Exit| Releases | Fate  
> ---|---|---|---|---  
> agnx | v2.6.29| v2.6.32| 4| Failed  
> altpciechdma | v2.6.29| v2.6.34| 6| Failed  
> arlan | v2.6.33| v2.6.35| 3|  Failed  
> asus_oled | v2.6.29| v3.12| 23| Failed  
> b3dfg | v2.6.30| v2.6.34| 5| Failed  
> bcm | v2.6.37| v3.19| 22| Failed  
> btmtk_usb | v3.11| v3.14| 4| Failed   
> ccg | v3.5| v3.10| 6| Failed   
> ced1401 | v3.7| v3.17| 11| Failed   
> crystalhd | v2.6.34| v3.17| 23| Failed   
> csr | v3.6| v3.11| 6| Failed   
> cxt1e1 | v2.6.35| v3.17| 22| Failed   
> dabusb | v2.6.38| v3.0| 2| Failed   
> dgap | v3.12| v4.6| 15| Failed   
> dgnc | v3.12| v4.20| 29| Failed   
> dgrp | v3.7| v3.17| 11| Failed   
> dream | v2.6.32| v2.6.37| 6| Failed   
> dst | v2.6.30| v2.6.33| 4| Failed   
> epl | v2.6.29| v2.6.32| 4| Failed   
> frontier | v2.6.29| v3.17| 28| Failed   
> ft1000 | v2.6.37| v4.4| 27| Failed  
> gdm72xx | v3.5| v4.6| 22| Failed   
> heci | v2.6.30| v2.6.32| 3| Failed   
> i2o | v4.0| v4.2| 3| Failed   
> i4l | v4.6| v4.11| 6| Failed   
> iio/gyro | v2.6.35| v4.19| 44| Failed   
> iio/trigger | v2.6.32| v4.17| 45| Failed  
> intel_sst | v2.6.37| v3.3| 6| Failed  
> keucr | v2.6.37| v3.17| 20| Failed  
> lttng | v2.6.33| v2.6.33| 1| [Failed](<https://lwn.net/Articles/471973/>)  
> lustre | v3.11| v4.18| 28| [Failed](<https://lwn.net/Articles/756565/>)  
> media/atomisp | v4.12| v4.18| 7| Failed  
> memrar | v2.6.35| v3.0| 5| Failed   
> mt7621-mmc | v4.17| v5.2| 7| Failed   
> net | v3.5| v3.10| 6| Failed   
> netwave | v2.6.33| v2.6.35| 3| Failed  
> nokia_h4p | v3.15| v3.18| 4| Failed  
> ozwpan | v3.4| v4.3| 20| Failed  
> p9auth | v2.6.30| v2.6.34| 5| Failed   
> phison | v2.6.30| v3.17| 27| Failed   
> poch | v2.6.28| v2.6.35| 8| Failed   
> pohmelfs | v2.6.30| v3.3| 13|  Failed  
> quickstart | v2.6.36| v3.17| 21| Failed   
> rspiusb | v2.6.29| v2.6.32| 4| Failed   
> sb105x | v3.8| v3.15| 8| Failed   
> sbe-2t3e3 | v2.6.37| v3.16| 19| Failed   
> sep | v2.6.38| v3.17| 19| Failed   
> serqt_usb2 | v2.6.31| v3.17| 26| Failed   
> silicom | v3.7| v3.17| 11| Failed   
> skein | v3.16| v4.19| 24| Failed   
> spectra | v2.6.36| v3.3| 7| Failed   
> strip | v2.6.33| v2.6.35| 3| Failed   
> sxg | v2.6.28| v2.6.32| 5| Failed   
> tidspbridge | v2.6.36| v3.17| 21| Failed   
> wavelan | v2.6.33| v2.6.35| 3| Failed   
> westbridge | v2.6.37| v3.1| 4| Failed   
> winbond | v2.6.28| v3.17| 29| Failed   
> wlags49_h2 | v2.6.33| v3.17| 24| Failed   
> wlags49_h25 | v2.6.33| v3.17| 24| Failed   
> xgifb | v2.6.35| v5.1| 47| Failed   
  
That is 60 modules, or 24% of the total. For most of these modules, the final commit reads something like "nobody has worked on this for years, it's time to give up". A few of them were for hardware that never actually made it into production, or which was so old and obsolete that nobody could obtain it to make the driver work properly. In some cases, as with the LTTng tracing subsystem, the code was pushed out almost immediately due to developer opposition to its presence. At least one was removed due to licensing issues. 

Regardless of the reason, each of the above modules reflects an effort that did not eventually bear fruit. By its nature, the staging tree was never going to be 100% successful at graduating modules into the mainline; whether a 24% failure rate is too high will certainly be a matter of personal opinion. 

#### Staging out

One role of the staging tree that was not anticipated back in 2008 is to be a final resting place for code that developers want to get rid of. If it appears that nobody is using a particular driver, it can be moved to the staging tree for a few releases; if nobody complains, it will eventually be deleted. These drivers have been "staged out" in that way: 

> Module| Entry| Exit| Releases | Fate  
> ---|---|---|---|---  
> autofs | v2.6.37| v3.0| 3| Staged out   
> cpia | v2.6.37| v2.6.38| 2| Staged out  
> generic_serial | v3.0| v3.1| 2| Staged out   
> ipx | v4.16| v4.18| 3| Staged out  
> irda | v4.14| v4.17| 4| Staged out  
> media/lirc | v2.6.36| v4.16| 40| Staged out  
> media/mx2 | v4.6| v4.8| 3| Staged out   
> media/mx3 | v4.6| v4.8| 3| Staged out  
> media/omap1 | v4.6| v4.8| 3| Staged out  
> media/omap24xx | v3.14| v3.19| 6| Staged out   
> media/parport | v3.19| v4.0| 2| Staged out  
> media/sn9c102 | v3.14| v3.17| 4| Staged out  
> media/soc_camera | v5.1| \--| **3**|  Staging out  
> media/timb | v4.6| v4.8| 3| Staged out  
> media/tlg2300 | v3.19| v4.0| 2| Staged out   
> media/vino | v3.19| v4.0| 2| Staged out  
> media/zoran | v4.18| v5.2| 6| Staged out  
> ncpfs | v4.16| v4.18| 3| Staged out  
> rdma/amso1100 | v4.3| v4.5| 3| Staged out   
> rdma/ehca | v4.3| v4.5| 3| Staged out   
> rdma/ipath | v4.3| v4.5| 3| Staged out   
> se401 | v2.6.38| v3.0| 2| Staged out   
> serial | v3.2| v3.5| 4| Staged out   
> smbfs | v2.6.37| v3.0| 3| Staged out   
> stradis | v2.6.37| v2.6.38| 2| Staged out  
> telephony | v3.4| v3.8| 5| Staged out   
> tty | v3.0| v3.1| 2| Staged out   
  
Thus, 27 modules — 11% of those to enter the staging tree — came initially from the rest of the kernel and were eventually deleted. The removal of unwanted code is clearly a good thing, so these cases should be seen as successes for the staging tree. The unfortunate part, perhaps, is that the amount of code staged out in this manner is so small; there are doubtless many other drivers in the kernel that are not actually in use anymore. 

#### Hangers-on

There remains one last group: the modules that are still in the staging tree as of the 5.3-rc7 kernel: 

> Module| Entry| Exit| Releases | Fate  
> ---|---|---|---|---  
> android/ion | v3.14| \--| 31|   
> android/uapi | v3.14| \--| 31|   
> axis-fifo | v4.19| \--| 6|   
> bcm2835-audio | v4.11| \--| 11| (now in vc04_services)  
> board | v3.17| \--| 28|   
> clocking-wizard | v3.19| \--| 26|   
> comedi | v2.6.29| \--| 55|   
> emxx_udc | v3.17| \--| 28|   
> erofs | v4.19| \--| 6|   
> fbtft | v4.0| \--| 25|   
> fieldbus | v5.2| \--| 2|   
> fsl-dpaa2 | v4.12| \--| 13|   
> fwserial | v3.8| \--| 37|   
> gasket | v4.19| \--| 6|   
> gdm724x | v3.12| \--| 33|   
> goldfish | v3.9| \--| 36|   
> greybus | v4.9| \--| 16|   
> gs_fpgaboot | v3.15| \--| 30|   
> iio | v2.6.32| \--| 52|   
> iio/accel | v2.6.32| \--| 52|   
> iio/adc | v2.6.32| \--| 52|   
> iio/addac | v2.6.38| \--| 46|   
> iio/cdc | v3.2| \--| 43|   
> iio/frequency | v2.6.38| \--| 46|   
> iio/impedance-analyzer | v3.2| \--| 43|   
> iio/meter | v2.6.38| \--| 46|   
> iio/resolver | v2.6.38| \--| 46|   
> isdn | v5.3| \--| 1|   
> kpc2000 | v5.2| \--| 2|   
> ks7010 | v4.8| \--| 17|   
> media/allegro-dvt | v5.3| \--| 1|   
> media/bcm2048 | v3.14| \--| 31|   
> media/davinci_vpfe | v3.9| \--| 36|   
> media/hantro | v5.0| \--| 4|   
> media/imx | v4.13| \--| 12|   
> media/imx074 | v4.17| \--| 8|   
> media/ipu3 | v5.0| \--| 4|   
> media/meson | v5.3| \--| 1|   
> media/mt9t031 | v4.17| \--| 8|   
> media/omap4iss | v3.14| \--| 31|   
> media/platform | v4.11| \--| 15| (now in vc04_services)  
> media/sunxi | v4.20| \--| 5|   
> media/tegra-vde | v4.16| \--| 9|   
> most | v4.3| \--| 22|   
> mt7621-dma | v4.17| \--| 8|   
> mt7621-dts | v4.17| \--| 8|   
> mt7621-pci | v4.17| \--| 8|   
> mt7621-pci-phy | v5.1| \--| 3|   
> mt7621-pinctrl | v4.17| \--| 8|   
> netlogic | v3.10| \--| 35|   
> nvec | v3.0| \--| 45|   
> octeon | v2.6.31| \--| 53|   
> octeon-usb | v3.11| \--| 34|   
> olpc_dcon | v2.6.37| \--| 47|   
> pi433 | v4.14| \--| 11|   
> ralink-gdma | v5.1| \--| 3|   
> rtl8188eu | v3.12| \--| 33|   
> rtl8192e | v2.6.32| \--| 52|   
> rtl8192u | v2.6.33| \--| 51|   
> rtl8712 | v2.6.37| \--| 47|   
> rtl8723bs | v4.12| \--| 13|   
> rts5208 | v3.14| \--| 31|   
> sm750fb | v4.1| \--| 24|   
> speakup | v2.6.37| \--| 47|   
> unisys | v3.15| \--| 30|   
> vc04_services | v4.9| \--| 16|   
> vme | v2.6.32| \--| 52|   
> vt6655 | v2.6.31| \--| 53|   
> vt6656 | v2.6.32| \--| 52|   
> wilc1000 | v4.2| \--| 23|   
> wlan-ng | v2.6.28| \--| 56|   
  
Thus, of the 247 modules to enter staging, 71 (29%) have yet to leave. The undisputed king of the long-term staging residents is the `comedi` subsystem. Since it was added in November 2008, this module has been the target of 8,673 of the 53,512 commits to the staging tree over its history — 16% of all activity in `drivers/staging`. There are quite a few other modules that have lived in the staging tree for a long time, though. If there was ever intended to be a time limit for residency in staging, it does not appear to have been enforced. 

Some of those are close to being ready for graduation; in the above-linked message, Kroah-Hartman named `comedi`, `greybus`, and `speakup` as needing ""just a bit more work"". The Greybus drivers, in fact, are currently queued to move in 5.4. Others are certainly in need of deletion: ""Yes, there is code in there that probably should be dropped now as I haven't done a sweep in a few years"". And others will probably hang on for a while yet. 

#### Exit stage right

To summarize, here are the conclusions on what happened to modules that were added to staging: 

> Fate| Count| Percent  
> ---|---|---  
> Graduated| 52| 21%  
> Superseded| 37| 15%  
> Failed| 60| 24%  
> Staged out| 27| 11%  
> Still present| 71| 29%  
  
None of this is likely to answer the question of whether the staging tree is a good thing for kernel development or not; as is so often the case, it has almost certainly been helpful for some modules and less so for others. What is clear is that a lot of code has passed through this part of the kernel tree and has helped to shape the mainline kernel as a whole. Like it or not, the staging tree is a part of the kernel development process; its role may evolve in response to pressure from developers, but it seems likely to be a significant part of our process for some time to come.
