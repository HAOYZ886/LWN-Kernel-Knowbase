---
title: "Should the Lustre preparation patches go in?"
url: https://lwn.net/Articles/88680/
date: "June 9, 2004"
category: "Clusters-Filesystems; Filesystems-Cluster; Lustre"
author: 
---

[Lustre](<http://www.clusterfs.com/lustre.html>) is a high-performance, distributed filesystem intended for use in large clusters. It is the latest effort from Peter Braam, who has, in the past, been responsible for the Coda and InterMezzo filesystems. Lustre has not been proposed for merging yet, but it is already in production use at a number of large supercomputing centers. Companies like Dell, Cray, and HP have been helping with its development. 

Mr. Braam has recently posted [the second iteration](<https://lwn.net/Articles/88679/>) of a patch intended to pave the way for inclusion of Lustre. This patch exports some symbols needed by Lustre and makes various virtual filesystem changes. With this patch in place, sites using Lustre would be able to load the filesystem as a separate module without having to patch the kernel directly. Since many of these sites, it seems, use "enterprise" distributions and cannot patch their kernels without invalidating their support agreements, this matters. Almost everybody involved would like Lustre to be usable on mainline kernels. 

Most of the technical objections to the Lustre patches have been addressed; to that end, many changes were made since the first posting. There remains one objection which can still be heard from a small number of developers, however: the patch should not be merged because it provides interfaces which are not used by any code in the kernel tree. This argument has been heard before; the Linux security module patches, for example, were opposed on this basis. 

It is not hard to understand a general reluctance to include (seemingly) unused APIs in the kernel. If an interface is not in active use, chances are that, when somebody _does_ try to use it, they will find that it does not work as advertised. Unused code tends to rot over time. And all code bloats the kernel, so it makes sense to hold off on adding new code until there is a clear use for it. 

It is also true, however, that the addition of new interfaces can help drive development in useful directions. The hooks needed by Lustre should be useful for a number of distributed filesystems, starting with NFS and going on to the various other cluster-oriented filesystems. Until the new interface is available, however, no filesystem will start using it. And, in any case, there is a clear user here in the form of Lustre, which is an available, GPL-licensed filesystem. 

Your editor, putting on his highly unreliable clairvoyant cap, figures that the Lustre developers will eventually get their wish. Certain developers will likely make them sweat for it, however, forcing a few more iterations on the patch before it can be accepted. But in the end, nobody disagrees with the goal (being to provide a high-quality distributed filesystem for high-performance clusters) and the patches were written with a relatively light hand. There is no real reason to keep them out of the kernel.
