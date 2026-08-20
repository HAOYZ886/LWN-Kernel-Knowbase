---
title: "Snowpatch: continuous-integration testing for the kernel"
url: https://lwn.net/Articles/777421/
date: "January 26, 2019"
category: "Development tools-Testing; Patchwork"
author: "By Jonathan Corbet January 26, 2019 LCA"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
January 26, 2019

* * *

[LCA](<https://lwn.net/Archives/ConferenceByYear/#2019-linux.conf.au>)

Many projects use continuous-integration (CI) testing to improve the quality of the software they produce. By running a set of tests after every commit, CI systems can identify problems quickly, before they find their way into a release and bite unsuspecting users. The Linux kernel project lags many others in its use of CI testing for a number of reasons, including a fundamental mismatch with how kernel developers tend to manage their workflows. At [linux.conf.au 2019](<https://2019.linux.conf.au/>), Russell Currey described a CI system called [Snowpatch](<https://github.com/ruscur/snowpatch>) that, he hopes, will bridge the gap and bring better testing to the kernel development process. 

There are a number of advantages to CI, Currey said. It provides immediate feedback to developers; with luck, they can fix their problems before other people have to spend any time reporting them. It can save a lot of time for reviewers. As a result, the whole code submission process speeds up, and the project is able to move more quickly as a whole. 

The core idea behind a kernel CI implementation is not complicated: one just needs to merge patches from the mailing lists, then run a set of tests on the result. These tests can be as simple as `checkpatch.pl`, but can also [![\[Russell Currey\]](https://static.lwn.net/images/conf/2019/lca/RussellCurrey-sm.jpg)](<https://lwn.net/Articles/777513/>) include building and booting, running the kernel's self-testing code, and more. Once the tests are done, the results can be reported back to the developer. 

Doing this in the kernel context proves to be harder than in projects that are hosted on sites like GitHub, though. A pull request contains all of the information needed to merge a group of changes; an email containing, say, patch 7/10 lacks that context. It is nearly impossible to tell from an email message whether a patch series has been merged, rejected, or superseded. In general, mailing lists simply do not carry the same level of metadata as contemporary project-hosting sites, and that makes the CI problem harder. 

Even so, there are groups doing CI testing on the kernel now. The "big boy" of kernel CI is the 0day robot, which picks up patches from the mailing lists and runs a number of tests. It does some static-analysis testing on the x86 architecture, build testing with over 100 kernel configurations, and a runs set of tests looking for performance regressions. When tests fail, email is sent to the developer. 0day is useful, but it is proprietary to Intel, so nobody else has the ability to change it to do what they want. In the absence of failures, there is also no way for developers to tell whether the tests have been run on a given patch posting or not. 

Providing better CI for the kernel requires obtaining better metadata for patches, but any proposal that requires kernel developers to change their workflow is clearly not going to get far, he said. The solution is to use [Patchwork](<http://jk.ozlabs.org/projects/patchwork/>), which is already in use by a number of kernel subsystems and is designed to supplement mailing lists rather than replacing them. Patchwork is able to track the state of patches, keep a patch series together, and host test results. And, perhaps best of all for those who would like to extend its functionality, it has a JSON API that can be used to build scripts around it. 

Patchwork fills the bill nicely because it is already in use and accepted by many developers; adopting it will not require any workflow changes. Patchwork can host test results without having to run the tests itself; they can come from anywhere. There is also value in having the results posted on a web site; developers can learn when tests have been run (and their outcome) without the need to send out email for every patch set. 

Snowpatch, thus, is built on top of patchwork. It is written in Rust in, Currey said, an attempt to be cool. The effort began at linux.conf.au 2016 in Geelong, and is maintained in collaboration with Andrew Donnellan. The code is GPL-licensed. There is an instance running now for the linuxppc-dev mailing list. 

At its core, Snowpatch grabs a patch from Patchwork, applies it to one or more repository branches, then sends the result to a remote system for testing. When the results come back, they are added to the Patchwork entry. Actually running the tests requires [Jenkins](<https://jenkins.io/>) for now — a limitation that Currey apologized for. But, he said, Jenkins does everything that the project needs it to do. 

Should anybody else want to set up a Snowpatch instance, he said, there are a few basic requirements. First of all, it needs a local repository to which patches can be applied. Access to a patchwork instance is needed to be able to publish the results. A Jenkins server is needed to run the tests, and there needs to be a remote Git repository that is visible to the Jenkins system. Currey ended his talk with an expression of hope that more kernel subsystems will set up Snowpatch and start making use of it to improve their CI testing. 

A member of the audience asked about the risk of malicious patches taking over the test machines. Currey answered that "something" needs to be in place to deal with that problem, but it hasn't been addressed yet. That something might involve having a maintainer approve test runs. That said, bad patches haven't been a problem so far. The final question had to do with dependencies between patches; Snowpatch has no real solution for that problem at this time. 

[A video of this talk](<https://www.youtube.com/watch?v=IRtBqPNsM9g>) is available on YouTube. 

[Thanks to linux.conf.au and the Linux Foundation for supporting my travel to the event.]
