---
title: Toward the merging of Xen
url: https://lwn.net/Articles/125831/
date: "March 2, 2005"
category: Xen
author: 
---

The [Xen virtual machine](<http://www.cl.cam.ac.uk/Research/SRG/netos/xen/>) has been getting a great deal of attention. Xen allows virtual systems to be run, over Linux, with high performance. Each machine can run a different operating system (perhaps even Windows, eventually), can have its resource usage limited, and can even be moved between physical hosts while it is running. Xen is of interest to people doing kernel development, or who are interested in providing virtual hosting services. 

Xen works by creating its own virtual hardware architecture, to which guest kernels are ported. The separate architecture is required to enable Xen to truly isolate guest systems in such a way that they cannot break out. This approach also allows Xen to perform various performance-enhancing tricks, such as allowing Xen systems to communicate by transparently remapping pages between them. For Linux, the Xen patches create a completely new architecture (`arch/xen`) which, while resembling the i386 architecture (and copying many files from it), is separate from it. 

For some time now, certain kernel developers have been saying that the merging of Xen was imminent. Nobody seems to object to having support for Xen in the mainline kernel, but there is one little glitch: back in December, Andi Kleen [objected](<https://lwn.net/Articles/125846/>) to the creation of a separate Xen architecture. The creation of a completely new architecture which duplicates much of the i386 code will, says Andi, lead to long-term maintenance problems. He would much rather see Xen support merged into an i386 subarchitecture. 

Xen developer Ian Pratt initially [responded](<https://lwn.net/Articles/125847/>) that such a merge was not feasible, and, besides, maintaining the separate architecture had not been a problem for them so far. Andi remained convinced, however, that things would not work well in the long term. The discussion slowed to a halt without any real decisions being made, one way or another. 

Andrew Morton recently decided to [restart the conversation](<https://lwn.net/Articles/125849/>) with an opinion of his own: 

I tend to agree with Andi, and I'm not sure that the Xen team fully appreciate the downside of having an own-architecture in the kernel.org kernel and the upside of having their code integrated with the most-maintained architecture. It could be that the potential problems haven't been sufficiently well communicated. 

Ian Pratt came back with [a new proposal](<https://lwn.net/Articles/125851/>). The Xen group would start by doing the easy parts of merging the Xen code directly into the i386 architecture. Most of this work, he says, would involve cleaning up the i386 code; the result would be a halving of the number of files modified by the Xen patches. The remaining changes would then go in as an i386 subarchitecture except for any Xen code which is useful for all architectures; that, instead, would end up in `drivers/xen/core`. Further unification and cleanup could happen after the merge takes place. 

This approach appears to have satisfied the critics, the obligatory minor quibbles notwithstanding. So that is probably the path Xen will take to get into the mainline. There is, it would seem, a fair amount of work to be done before that mainline merge can actually happen, though, so it's not at all clear that it can be done in time for 2.6.12.
