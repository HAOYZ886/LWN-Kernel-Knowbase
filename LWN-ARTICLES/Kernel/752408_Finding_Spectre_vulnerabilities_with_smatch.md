---
title: Finding Spectre vulnerabilities with smatch
url: https://lwn.net/Articles/752408/
date: "April 20, 2018"
category: "Security-Meltdown and Spectre"
author: "By Jonathan Corbet April 20, 2018"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
April 20, 2018

The furor over the Meltdown and Spectre vulnerabilities has calmed a bit — for now, at least — but that does not mean that developers have stopped worrying about them. Spectre variant 1 (the bounds-check bypass vulnerability) has been of particular concern because, while the kernel is thought to contain numerous vulnerable spots, nobody really knows how to find them all. As a result, the defenses that have been developed for variant 1 have only been deployed in a few places. Recently, though, Dan Carpenter has [enhanced](<https://lwn.net/Articles/752409/>) the [smatch](<https://lwn.net/Articles/691882/>) tool to enable it to find possibly vulnerable code in the kernel. 

Spectre variant 1 is the result of the processor incorrectly predicting the results of a bounds check; it then speculatively executes code with a parameter (such as an array index) that falls outside of its allowed range. This problem can be [mitigated](<https://lwn.net/Articles/746551/>) by disabling speculative execution in situations where an array index is under the control of a potential attacker. In the kernel, that is done by replacing code like: 

```
value = array[index];
```

with: 

```
index = array_index_nospec(index, ARRAY_SIZE);
        value = array[index];
```

That's the easy part; the hard part is finding the places in the kernel where the `array_index_nospec()` macro should be used. Until now, the only tool available has been the proprietary Coverity checker, which is not accessible to everybody and produces a fair number of false positives. As a result, there are only a handful of `array_index_nospec()` calls in current kernels. 

Carpenter's addition to smatch changes that situation by providing a free tool that can search for potential Spectre variant-1 vulnerabilities. The algorithm is simple enough in concept: 

What the test does is it looks at array accesses where the user controls the offset. It asks "is this a read?" and have we used the array_index_nospec() macro? If the answers are yes, and no respectively then print a warning. 

This test returns a list of about 800 places where `array_index_nospec()` should be used. Carpenter assumes that a large percentage of these are false positives, and has asked for suggestions on how the test could be made more accurate. Instead of offering suggestions, though, both [Thomas Gleixner](<https://lwn.net/Articles/752410/>) and [Peter Zijlstra](<https://lwn.net/Articles/752411/>) confirmed that a number of the reports were accurate; Zijlstra said ""I fear that many are actually things we want to fix"". He followed up with [a patch series](<https://lwn.net/Articles/752412/>) fixing seven of them — nearly doubling the number of `array_index_nospec()` calls in the kernel. 

Once the low-hanging fruit has been tackled, there probably will be a focus on improving the tests in smatch to filter out the inevitable false positives and to be sure that vulnerable sites are not slipping through. But, now that there is a free tool to do this checking, progress in this area can be expected to accelerate. Perhaps it will be possible to find — and fix — many of the existing Spectre vulnerabilities before the attackers get there.
