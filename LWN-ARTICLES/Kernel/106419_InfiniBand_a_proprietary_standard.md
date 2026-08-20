---
title: "InfiniBand: a proprietary standard?"
url: https://lwn.net/Articles/106419/
date: "October 13, 2004"
category: InfiniBand
author: 
---

Greg Kroah-Hartman recently [expressed some concerns](<https://lwn.net/Articles/106420/>) about the InfiniBand specification. It seems that, if you are not a member of the [InfiniBand Trade Association](<http://www.infinibandta.org/home>), a copy of the specification will cost $9500 - and it requires signing a license which reads: 

Upon receipt by IBTA of payment for a single copy license to the Specification, you are entitled to possess one physical copy of the Specification in the form provided to you by IBTA, and to make internal, noncommercial use of the Specification within your organization. 

Such language raises the obvious question: how can anybody write or distribute a free InfiniBand implementation after having signed that sort of license? Things get worse when one looks at [the IBTA membership agreement (PDF)](<http://www.infinibandta.org/meminfo/mem-agreement.pdf>): 

When the member or its Affiliates makes a Contribution or when the Steering Committee adopts and approves for release a Specification, the Member and its Affiliates hereby agree to grant **to other members and their affiliates** under reasonable terms and conditions that are demonstrably free of any unfair discrimination, a nonexclusive, nontransferable, worldwide license under its Necessary Claims to allow such Members to make, have made, use, import, offer to sell, lease, and sell and otherwise distribute Compliant Portions .... 

The Member and its Affiliates retain the independent right to grant or withhold a nonexclusive license or sublicense of patents containing Necessary Claims to non-Members on such terms as the Member may determine. 

(Emphasis added). The InfiniBand standard, in other words, is allowed to contain patented technology, only IBTA members must be given the opportunity to license any patented technology, and only under "reasonable terms and conditions." If said "reasonable terms and conditions" included the right to distribute code under a free license, one would assume those who wrote the agreement would have seen fit to say so. 

The end result is that InfiniBand looks like a closed, proprietary standard, and not something which can be supported in free software. Greg asked, flat out: 

So, OpenIB group, how to you plan to address this issue? Do you all have a position as to how you think your code base can be accepted into the main kernel tree given these recent events? 

In response, there have been some "we don't think it's a problem" mumblings, but nothing that looks like a real answer to this question. Until this all gets straightened out, anybody considering using InfiniBand with free software may well want to think about alternatives.
