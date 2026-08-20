---
title: "How many kernel test frameworks?"
url: https://lwn.net/Articles/790235/
date: "June 5, 2019"
category: "Development tools-Testing"
author: "By Jake Edge June 5, 2019"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jake Edge**  
June 5, 2019

The kernel self-test framework ([kselftest](<https://www.kernel.org/doc/html/latest/dev-tools/kselftest.html>)) has been a part of the kernel for some time now; a relatively recent proposal for a [kernel unit-testing framework](<https://lwn.net/Articles/780985/>), called [KUnit](<https://google.github.io/kunit-docs/third_party/kernel/docs/>), has left some wondering why both exist. In a lengthy discussion thread about KUnit, the justification for adding another testing framework to the kernel was debated. While there are different use cases for kselftest and KUnit, there was concern about fragmenting the kernel-testing landscape. 

In early May, Brendan Higgins posted [v2 of the KUnit patch set](<https://lwn.net/ml/linux-kernel/20190501230126.229218-1-brendanhiggins@google.com/>) with an eye toward getting it into Linux 5.2. That was [deemed](<https://lwn.net/ml/linux-kernel/20190502105053.GA12416@kroah.com/>) a bit of an overaggressive schedule by Greg Kroah-Hartman [and Shuah Khan](<https://lwn.net/ml/linux-kernel/76e84d54-6b7e-8cc1-492b-43822fc43ac4@kernel.org/>) given that the merge window would be opening a week later or so. But Khan did agree that the patches could come in via her kselftest tree. There were some technical objections to some of the patches, which is no surprise, but overall the patches were met with approval—and some Reviewed-by tags. 

There were some sticking points, however. Several, including Kroah-Hartman [and Logan Gunthorpe](<https://lwn.net/ml/linux-kernel/ea36c3d6-8c13-2186-16f3-596d834aeebe@deltatee.com/>) complained about the reliance on user-mode Linux (UML) to run the tests. Higgins [said](<https://lwn.net/ml/linux-kernel/CAFd5g46RyQ+jaV3bDejBaeca4Yv3G9ppT5JxdKqZw6PSbiSDYw@mail.gmail.com/>) that he had ""mostly fixed that"". The KUnit tests will now run on any architecture, though the Python wrapper scripts are still expecting to run the tests in UML. He said that he should probably document that, which is something that he has [subsequently done](<https://google.github.io/kunit-docs/third_party/kernel/docs/usage.html#kunit-on-non-uml-architectures>). 

A more overarching concern was [raised](<https://lwn.net/ml/linux-kernel/54940124-50df-16ec-1a32-ad794ee05da7@gmail.com/>) by Frank Rowand. From his understanding, using UML is meant to ""avoid booting a kernel on real hardware or in a virtual machine"", he said, but he does not really see that as anything other than ""a matter of semantics""; running Linux via UML is simply a different form of virtualization. Furthermore: 

It seems to me that KUnit is just another piece of infrastructure that I am going to have to be familiar with as a kernel developer. More overhead, more information to stuff into my tiny little brain. 

I would guess that some developers will focus on just one of the two test environments (and some will focus on both), splitting the development resources instead of pooling them on a common infrastructure. 

Khan [replied](<https://lwn.net/ml/linux-kernel/1b1efa91-0523-21a9-e541-fdc3612bd117@kernel.org/>) that she sees kselftest and KUnit as complementary. Kselftest is ""a collection of user-space tests with a few kernel test modules back-ending the tests in some cases"", while KUnit provides a framework for in-kernel testing. Rowand [was not particularly swayed](<https://lwn.net/ml/linux-kernel/911e44bb-9bb1-e603-a260-fac63760fff6@gmail.com/>) by that argument, however. He sees that there is (or could be) an almost complete overlap between the two. 

Unlike some other developers, Ted Ts'o actually finds the use of UML to be beneficial. He [described](<https://lwn.net/ml/linux-kernel/20190507172256.GB5900@mit.edu/>) some unit tests that are under development for ext4; they will test certain features of ext4 in isolation from any other part of the kernel, which is where he sees the value in KUnit. The framework provided with kselftest targets running tests from user space, which requires booting a real kernel, while KUnit is simpler and faster to use: 

So this is why it's largely irrelevant to me that KUinit uses UML. In fact, it's a feature. We're not testing device drivers, or the scheduler, or anything else architecture-specific. UML is not about virtualization. What it's about in this context is allowing us to start running test code as quickly as possible. Booting KVM takes about 3-4 seconds, and this includes initializing virtio_scsi and other device drivers. If by using UML we can hold the amount of unnecessary kernel subsystem initialization down to the absolute minimum, and if it means that we can [communicate] to the test framework via a userspace "printf" from UML/KUnit code, as opposed to via a virtual serial port to KVM's virtual console, it all makes for lighter weight testing. 

#### Frameworks

Part of the difference of opinion may hinge on the definition of "framework" to a certain extent. Ts'o stridently argued that kselftest is not providing an in-kernel testing framework, but Rowand just as vehemently disagreed with that. Rowand [pointed](<https://lwn.net/ml/linux-kernel/a09a7e0e-9894-8c1a-34eb-fc482b1759d0@gmail.com/>) to the use of kernel modules in kselftest and [noted](<https://lwn.net/ml/linux-kernel/6d6e91ec-33d3-830b-4895-4d7a20ba7d45@gmail.com/>) that those modules can be built into a UML kernel. Ts'o [did not think](<https://lwn.net/ml/linux-kernel/20190509015856.GB7031@mit.edu/>) that added up to a framework since ""each of the in-kernel code has to create their own in-kernel test infrastructure"". Rowand [sees that differently](<https://lwn.net/ml/linux-kernel/78e4d46e-6212-9871-51d6-dd2126f39d45@gmail.com/>): ""The kselftest in-kernel tests follow a common pattern. As such, there is a framework."" [To Ts'o](<https://lwn.net/ml/linux-kernel/20190510104338.GB6889@mit.edu/>), that doesn't really equate to a framework, though perhaps the situation could change down the road: 

So we may have different definitions of "framework". In my book, code reuse by "cut and paste" does not make a framework. Could they be rewritten to *use* a framework, whether it be KTF [Kernel Test Framework] or KUnit? Sure! But they are not using a framework *today*. 

In addition, Ts'o [said](<https://lwn.net/ml/linux-kernel/20190509214233.GA20877@mit.edu/>) that kselftest expects to have a working user-space environment: 

One major difference: kselftest requires a userspace environment; it starts systemd, requires a root file system from which you can load modules, etc. Kunit doesn't require a root file system; doesn't require that you start systemd; doesn't allow you to run arbitrary perl, python, bash, etc. scripts. 

Rowand [disagreed](<https://lwn.net/ml/linux-kernel/80c72e64-2665-bd51-f78c-97f50f9a53ba@gmail.com/>): 

Kselftest in-kernel tests (which is the context here) can be configured as built in instead of as a module, and built in a UML kernel. The UML kernel can boot, running the in-kernel tests before UML attempts to invoke the init process. 

No userspace environment needed. So exactly the same overhead as KUnit when invoked in that manner. 

Ts'o is [not convinced](<https://lwn.net/ml/linux-kernel/20190511173344.GA8507@mit.edu/>) by that. He noted that the kselftest documentation is missing any mention of this kind of test. There are tests that run before init is started, but they aren't part of the kselftest framework: 

There exists test modules in the kernel that run before the init scripts run --- but that's not strictly speaking part of kselftests, and do not have any kind of infrastructure. As noted, the kselftests_harness header file fundamentally assumes that you are running test code in userspace. 

#### Overlaps

There may be overlaps in the functionality of KUnit and kselftest, however. Knut Omang, who is part of the [Kernel Test Framework](<https://github.com/oracle/ktf>) project—another unit-testing project for the kernel that is not upstream—[pointed out](<https://lwn.net/ml/linux-kernel/7fd35df81c06f6eb319223a22e7b93f29926edb9.camel@oracle.com/>) that there are two types of tests that are being conflated a bit in the discussion. One is an isolated test of a particular subsystem that is meant to be run rapidly and repeatedly by developers of that subsystem. The other is meant to test interactions between more than one subsystem and might be run as part of a regression test suite or in a continuous-integration effort, though it would be used by developers as well. The unit tests being developed for ext4 would fall into the first category, while xfstests would fall into the latter. 

Omang said that the two could potentially be combined into a single tool, with common configuration files, test reporting, and so on. That is what KTF is trying to do, he said. But Ts'o is [skeptical](<https://lwn.net/ml/linux-kernel/20190509133551.GD29703@mit.edu/>) that a single test framework is the way forward. There are already multiple frameworks out there, he said, including xfstests, blktests, kselftest, and so on. Omang also suggested that UML was still muddying the waters in terms of single-subsystem unit tests: 

[...] the problem with using UML is that you still have to relate to the complexity of a kernel run time system, while what you really want for these types of tests is just to compile a couple of kernel source files in a normal user land context, to allow the use of Valgrind and other user space tools on the code. The challenge is to get the code compiled in such an environment as it usually relies on subtle kernel macros and definitions, which is why UML seems like such an attractive solution. 

But Ts'o sees things differently: 

"Just compiling a couple of kernel source files in a normal user land" is much harder than you think. It requires writing vast numbers of mocking functions --- for a file system I would have to simulate the block device layer, large portions of the VFS layer, the scheduler and the locking layer if I want to test locking bugs, etc., etc. In practice, UML itself is serving as [the] mocking layer, by its mere existence. So when Frank says that KUnit doesn't provide any mocking functions, I don't at all agree. Using KUnit and UML makes testing internal interfaces *far* simpler, especially if the comparison is "just compile some kernel source files as part of a userspace test program". 

Gunthorpe [saw some potential overlap](<https://lwn.net/ml/linux-kernel/b09ba170-229b-fde4-3e9a-e50d6ab4c1b5@deltatee.com/>) as well. He made a distinction in test styles that was somewhat similar to Omang's. He noted that there are not many users of the `kselftest_harness.h` interface at this point, so it might make sense to look at unifying the areas that overlap sooner rather than later. 

The second item, arguably, does have significant overlap with kselftest. Whether you are running short tests in a light weight UML environment or higher level tests in an heavier VM the two could be using the same framework for writing or defining in-kernel tests. It *may* also be valuable for some people to be able to run all the UML tests in the heavy VM environment along side other higher level tests. 

Looking at the selftests tree in the repo, we already have similar items to what Kunit is adding as I described in point (2) above. kselftest_harness.h contains macros like EXPECT_* and ASSERT_* with very similar intentions to the new KUNIT_EXECPT_* and KUNIT_ASSERT_* macros. 

Ts'o [is not opposed](<https://lwn.net/ml/linux-kernel/20190509233043.GC20877@mit.edu/>) to unifying the tests in whatever way makes sense, but said that `kselftest_harness.h` needs to be reworked before in-kernel tests can use it. Gunthorpe seemed to change his mind some when he [replied](<https://lwn.net/ml/linux-kernel/8914afef-1e66-e6e3-f891-5855768d3018@deltatee.com/>) that perhaps the amount of work to unify the two use cases was not worth it: 

Using kunit for in-kernel tests and kselftest_harness for userspace tests seems like a sensible line to draw to me. Trying to unify kernel and userspace here sounds like it could be difficult so it's probably not worth forcing the issue unless someone wants to do some really fancy work to get it done. 

Ultimately, what [Rowand seems to be after](<https://lwn.net/ml/linux-kernel/875c546d-9713-bb59-47e4-77a1d2c69a6d@gmail.com/>) is a better justification for KUnit and why it is, and needs to be, different from kselftest, in the patch series itself. ""I was looking for a fuller, better explanation than was given in patch 0 of how KUnit provides something that is different than what kselftest provides for creating unit tests for kernel code."" Higgins [asked](<https://lwn.net/ml/linux-kernel/20190514083819.GC230665@google.com/>) for specific suggestions on where the documentation of KUnit was lacking. Rowand [replied](<https://lwn.net/ml/linux-kernel/5ff098a9-9424-901c-9017-d4492e306528@gmail.com/>) that in-patch justification is what he, as a code reviewer, was looking for: 

One thing that has become very apparent in the discussion of this patch series is that some people do not understand that kselftest includes in-kernel tests, not just userspace tests. As such, KUnit is an additional implementation of "the same feature". (One can debate exactly which in-kernel test features kselftest and KUnit provide, and how much overlap exists or does not exist. So don't take "the same feature" as my final opinion of how much overlap exists.) So that is a key element to be noted and explained. 

But Gunthorpe [did not agree](<https://lwn.net/ml/linux-kernel/8c693b9f-43ec-8982-825c-cabfd61b659d@deltatee.com/>); ""in my opinion, Brendan has provided over and above the information required to justify Kunit's inclusion"". The difference of opinion about whether kselftest provides any kind of in-kernel framework appears to be the crux of the standoff. Gunthorpe believes that the in-kernel kselftest code should probably be changed to use KUnit, once it gets merged, which he was strongly in favor of. 

As the discussion was trailing off, Higgins [posted v3 of the patch set](<https://lwn.net/ml/linux-kernel/20190514054251.186196-1-brendanhiggins@google.com/>) on May 13, followed closely by an [update to v4](<https://lwn.net/ml/linux-kernel/20190514221711.248228-1-brendanhiggins@google.com/>) a day later. Both addressed the technical comments on the v2 code and also added the documentation about running on architectures other than UML. There have been relatively few comments and no major complaints about those postings. One might guess that KUnit is on its way into the mainline, probably for 5.3.
