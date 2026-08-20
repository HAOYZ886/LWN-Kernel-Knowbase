---
title: "Kernel Summit 2006: Automated testing"
url: https://lwn.net/Articles/191924/
date: "July 19, 2006"
category: Development tools; Regression testing
author: 
---

[2006 Kernel Summit coverage](<https://lwn.net/Articles/KernelSummit2006/>) on LWN.net.   
---  
Martin Bligh addressed the kernel summit group about the automated testing work he and Andy Whitcroft are doing. It was one of the more presentation-oriented sessions; among other things, few people, even in this crowd, can find a way to disagree with more kernel testing. 

Martin and Andy have set up an array of systems used to test kernel releases. There are redundant machines with identical hardware, an arrangement which has useful benefits: if only one of a set of identical machines fails a test, the chances of hardware issues being the problem are nearly 100%. These machines are equipped with serial consoles, software-controlled power reset switches, etc. 

This sort of testing is worthwhile for a number of reasons. The lack of a development kernel series compresses the testing period in a big way, so anything which can be done to increase the amount of testing during the development cycle - before bugs can go on to affect users - is worthwhile. If all of the tests are passed, there can be a certain amount of confidence that no major regressions have been introduced. And, unlike volunteers, automated testing systems don't have a tendency to find other interests and disappear. 

There is an impressive set of tools which can run the tests, possibly applying one or more extra patches first. Even more features are planned for the future - including, for example, the ability to automatically bisect a patch stream and identify the patch which caused a specific failure. The source for the testing harness is available, and Martin encouraged companies which are in the business of submitting patches to start running it. Running patches through the testing routine before sending them out can, it seems, help to avoid a certain amount of embarrassment. 

Results from the testing project are posted at [test.kernel.org](<http://test.kernel.org/>). 

  * [Next: The VFS layer](<https://lwn.net/Articles/191926/>).
