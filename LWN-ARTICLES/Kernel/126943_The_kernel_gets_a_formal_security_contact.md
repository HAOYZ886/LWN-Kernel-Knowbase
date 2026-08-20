---
title: The kernel gets a formal security contact
url: https://lwn.net/Articles/126943/
date: "March 9, 2005"
category: Security
author: 
---

The Linux kernel has been nearly unique in that it has operated without any sort of formal security organization. Security-related patches would be sent to a (hopefully) relevant maintainer, who would (hopefully) get it merged into the mainline. With luck, distributors would notice the merging of security-related patches and issue the appropriate updates. 

The whole system was somewhat unwieldy (though it worked most of the time), but, with [this message from Chris Wright](<https://lwn.net/Articles/126942/>), things are beginning to change. There is now an official security contact address - `security@kernel.org` \- which is distributed to a set of "security officers" who will take the appropriate action in response to security-related bugs. The people behind that alias, as of this writing, are Linus Torvalds, Andrew Morton, Alan Cox, Marcelo Tosatti, H. Peter Anvin, and Chris Wright 

The posting also includes a disclosure policy, which reads as: 

The goal of the Linux kernel security team is to work with the bug submitter to bug resolution as well as disclosure. We prefer to fully disclose the bug as soon as possible. It is reasonable to delay disclosure when the bug or the fix is not yet fully understood, the solution is not well-tested or for vendor coordination. However, we expect these delays to be short, measurable in days, not weeks or months. A disclosure date is negotiated by the security team working with the bug submitter as well as vendors. However, the kernel security team holds the final say when setting a disclosure date. The timeframe for disclosure is from immediate (esp. if it's already publically known) to a few weeks. As a basic default policy, we expect report date to disclosure date to be on the order of 7 days. 

So the mechanism is now in place. What remains to be seen is how well it works when the next security hole turns up.
