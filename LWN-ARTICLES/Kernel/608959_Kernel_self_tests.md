---
title: Kernel self tests
url: https://lwn.net/Articles/608959/
date: "August 20, 2014"
category: Regression testing
author: "By Jonathan Corbet August 20, 2014 Kernel Summit 2014"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
August 20, 2014

* * *

[Kernel Summit 2014](<https://lwn.net/Articles/KernelSummit2014/>)

Shuah Khan started off her session at the 2014 Kernel Summit by noting that she has been helping with the process of testing stable kernel releases for some time. That testing is mostly of the build-and-boot variety, but it would be nice to be able to test kernels a bit more thoroughly than that. If there were a simple set of sanity tests that any developer could run, perhaps more regressions could be caught early, before they can affect users. The result of her work toward that goal is a new `make` target called "kselftest" in the kernel build system. 

There is a minimal set of tests there now; going forward, she would like to increase the set of tests that is run. We have lots of testing code out there, she said; it would be good to make more use of it. But she would like help in deciding which tests make sense to include with kselftest. The goal, she said, is to make things run quickly; it's a basic sanity test, not a full-scale stress test. 

Ted Ts'o asked what the time budget is; what does "quickly" mean? Shuah replied that she doesn't know; the current tests run in well under ten minutes. That time could probably be increased, she said, but it should not grow to the point that developers don't want to run the tests. Mel [![\[Shuah Khan\]](https://static.lwn.net/images/conf/2014/ks/ShuahKhan-sm.jpg)](<https://lwn.net/Articles/608961/>) Gorman noted that his tests, if run fully, take about thirteen days — probably a bit more than the budget allows. Paul McKenney added that the full torture-test suite for the read-copy-update subsystem runs for more than six hours. At this point, Shuah said that her goal is something closer to 15-20 minutes. 

Josh Triplett expressed concerns about having the tests in the kernel tree itself. It could be hard to use bisection to find a test failure if the tests themselves are changing as well. Perhaps, he said, it would be better to fetch the tests from somewhere else. But Shuah said that would work against the goal of having the tests run quickly and would likely reduce the number of people running them. 

Darren Hart asked if only functional tests were wanted, or if performance tests could be a part of the mix as well. Shuah responded that, at this point, there are no rules; if a test makes sense as a quick sanity check, it can go in. What about driver testing? That can be harder to do, but it might be possible to put together an emulator that simulates devices, bugs and all. 

Grant Likely said that it would be nice to standardize the output format of the tests to make it easier to generate reports. There was a fair amount of discussion about testing frameworks and test harnesses. Rather than having the community try to choose one by consensus, it was suggested, Shuah should simply pick one. But Christoph Hellwig pointed out that the xfstests suite has no standards at all. Instead, the output of a test run is compared to a "golden" output, and the differences are called out. That makes it easy to bring in new tests and avoids the need to load a testing harness on a small system. Chris Mason agreed that this was "the only way" to do things. 

The session closed with Shuah repeating that she would like more tests for the kselftest target, along with input about how this mechanism should work in general. 

**Next** : [Two sessions on review](<https://lwn.net/Articles/608968/>).
