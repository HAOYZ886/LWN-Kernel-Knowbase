---
title: Fishy business
url: https://lwn.net/Articles/377103/
date: "March 3, 2010"
category: Android
author: "By Jonathan Corbet March 3, 2010"
---

By **Jonathan Corbet**  
March 3, 2010

Many pixels have been expended about the presence of the Android code in the mainline kernel, or, more precisely, the lack thereof. There are many reasons for Android's absence, including the Android team's prioritization of upcoming handset releases over upstreaming the code and some strong technical disagreements over some of the Android code. For a while, it seemed that there might be yet another obstacle: source files named after fish. 

Like most products, Android-based handsets go through a series of code names before they end up in the stores. Daniel Walker [cited an example](<https://lwn.net/Articles/377104/>): an HTC handset which was named "Passion" by the manufacturer. When it got to Google for the Android work, they concluded that "Mahimahi" would be a good name for it. It's only when this device got to the final stages that it gained the "Nexus One" name. Apple's "dirty infringer" label came even later than that. 

Daniel asked: which name should be used when bringing this code into the mainline kernel? The Google developers who wrote the code used the "mahimahi" name, so the source tree is full of files with names like `board-mahimahi-audio.c`; they sit alongside files named after trout, halibut, and swordfish. Daniel feels these names might be confusing; for this reason, `board-trout.c` became `board-dream.c` when it moved into the mainline. After all, very few G1/ADP1 users think that they are carrying trout in their pockets. 

The problem, of course, is that this kind of renaming only makes life harder for people who are trying to move code between the mainline and Google's trees. Given the amount of impedance which already exists on this path, it doesn't seem like making things harder is called for. ARM maintainer Russell King [came to that conclusion](<https://lwn.net/Articles/377106/>), decreeing: 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

There's still precious little to show in terms of progress on moving this code towards the mainline tree - let's not put additional barriers in the way. 

Let's keep the current naming and arrange for informative comments in files about the other names, and use the common name in the Kconfig - that way it's obvious from the kernel configuration point of view what is needed to be selected for a given platform, and it avoids the problem of having effectively two code bases. 

That would appear to close the discussion; the board-level Android code can keep its fishy names. Of course, that doesn't help if the code doesn't head toward the mainline anyway. The good news is that people have not given up, and work is being done to help make that happen. With luck, installing a mainline kernel on a swordfish will eventually be a straightforward task for anybody.
