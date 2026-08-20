---
title: Toward better kernel releases
url: https://lwn.net/Articles/114626/
date: "December 7, 2004"
category: "Development model-Kernel quality; Regression testing; Releases"
author: 
---

It was asked recently: is the 2.6.10 release coming sometime soon? Andrew Morton [replied](<http://lwn.net/Articles/114218/>) that the latter part of December looked like when it might happen. He also noted that he is trying to produce a higher-quality release this time around: 

We need to be be achieving higher-quality major releases than we did in 2.6.8 and 2.6.9. Really the only tool we have to ensure this is longer stabilisation periods. 

Andrew also noted that getting people to test anything other than the final releases is hard, with the result that many bugs are only reported after a new "stable" kernel is out. If things don't get better, says Andrew, it may be necessary to start doing point releases (e.g. 2.6.10.1) for the final stabilization steps. Alternatively, the kernel developers could switch to a new sort of even/odd scheme, so that 2.6.11 would be a new features release, and 2.6.12 would be bug fixes only. 

Much of the discussion, however, centered around regression testing. If only there were more automated testing, the reasoning goes, fewer bugs would make it into final kernel releases. This wish may eventually come true, but, for now, it appears that regression testing is not as helpful as many would like. 

OSDL has [pointed out](<http://lwn.net/Articles/114408/>) that it runs a whole set of tests every day. The problem, they say, is getting people to actually look at the results. It may be that not enough people know about OSDL's work, and, for that reason, the output is not being used. But it also may be that the testing results are simply not that useful. 

Consider [this posting from Andrew Morton](<https://lwn.net/Articles/114627/>) on regression testing: 

However I have my doubts about how useful it will end up being. These test suites don't seem to pick up many regressions.... We simply get far better coverage testing by releasing code, because of all the wild, whacky and weird things which people do with their computers. Bless them. 

The test suites, it seems, are not testing for the right things. One could argue that the test suites simply have not, yet, been developed to the point where they are performing comprehensive testing of the kernel. This gap could be slowly filled in by having kernel bug fixes be accompanied by new tests which verify that the bug remains fixed. Much of the code in the kernel, however, is hardware-specific, and that code is where a lot of bugs tend to be found. Hardware-specific code can only be tested in the presence of the hardware in question. Outfitting a testing lab with even a fraction of the hardware supported by Linux would be a massively expensive undertaking. 

So the wider Linux community is likely to remain the testing lab of last resort for the kernel; the community as a whole, after all, does have all that hardware. And the truth of the matter is that helping with testing is part of the cost of free software (and of the proprietary variety as well). So the best results might be had by trying to get more widespread testing earlier in the process. Getting Linus to distinguish between intermediate and release candidate kernels might help in that regard. If that can't be done, then, perhaps, going with point releases may be required.
