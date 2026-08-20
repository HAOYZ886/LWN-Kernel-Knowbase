---
title: Filesystem test suites
url: https://lwn.net/Articles/757239/
date: "June 13, 2018"
category: "Development tools-Testing"
author: "By Jake Edge June 13, 2018 LSFMM"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jake Edge**  
June 13, 2018

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2018/>)

While the 2018 Linux Storage, Filesystem, and Memory-Management Summit (LSFMM) filesystem track session was advertised as being a filesystem test suite "bakeoff", it actually focused on how to make the existing test suites more accessible. Kent Overstreet said that he has learned over the years that various filesystem developers have their own scripts for testing using QEMU and other tools. He and Ted Ts'o put the session together to try to share some of that information (and code) more widely. 

Most of the scripts and other code has not been polished or turned into a project, Overstreet continued. Bringing new people up to speed on the tests and how they are run takes time, but developers want to know how to run the tests before they send code to the maintainer. 

Ts'o said that he had a goal for his [xfstests-bld](<https://opensource.google.com/projects/xfstests-bld>) tool: give people submitting ext4 patches no excuses for not running the smoke tests. He wants to make sure that patches have been at least minimally tested before spending time reviewing them. Xfstests-bld has support for a handful of different filesystems, but just with default options for any beyond ext4; his hope is that other filesystems will also use it and provide suitable configurations. 

J. Bruce Fields said that he runs the smoke tests on all patches anyway, so he would rather have patch submitters run their own tests on the code. But Ts'o was adamant that since there are more submitters than maintainers, he wants to know that the smoke tests have been run before looking at a submission. 

Overstreet has a test framework that builds a test kernel, boots into it, and builds the tests and runs them with QEMU. It uses `debootstrap` to build a root filesystem and users do not have to mess with Kconfig; there is a stripped-down configuration that the test uses. Ts'o said that he has two configurations, one for QEMU and another for Google Compute Engine. Overstreet wondered if it was "silly for us all to maintain our own" test harnesses. 

Mimi Zohar had a different complaint: getting started with xfstests is difficult. There is minimal documentation and no default configuration, which means it takes a long time to get going. That is why some have created these test scripts, Ts'o said. 

Ts'o further described his framework. It can run tests in KVM/QEMU or push them to Android devices. The `smoke` parameter will run the default tests for the filesystem being tested; for ext4, those tests take about ten minutes. Full testing of ext4 takes about 20 hours, but an intern he worked with was able to reduce that to two hours by sharding the tests in Google Compute Engine. The Android xfstests require a rooted device and, likely, one you are not particularly attached to; the tests do a lot of writes, which may drastically reduce the lifetime of the device. 

Eric Sandeen noted that the documentation for the different tests and frameworks is scattered among various web pages; he wondered if it could be centralized somewhere. Dave Chinner suggested patches to the xfstests documentation that at least pointed those interested to all of the different pages.
