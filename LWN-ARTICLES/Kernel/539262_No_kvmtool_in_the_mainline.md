---
title: No kvmtool in the mainline
url: https://lwn.net/Articles/539262/
date: "February 20, 2013"
category: "KVM; Virtualization-KVM"
author: "By Jonathan Corbet February 20, 2013"
---

By **Jonathan Corbet**  
February 20, 2013

The story of the "native Linux KVM tool" (or, more recently, "kvmtool") has been playing out since early 2011. This tool serves as a simple replacement for the QEMU emulator, making it easy to set up and run guests under KVM. The kvmtool developers have been working under the assumption that their code would be merged into the mainline kernel, as was done with `perf`, but others have [disagreed](<https://lwn.net/Articles/456132/>) with that idea. The result has been a repetitive conversation every merge window or two as kvmtool was proposed for merging. 

The conversation for the 3.9 merge window has seemingly been a bit more decisive, though. Ingo Molnar (along with kvmtool developer Pekka Enberg) presented [a long list of reasons](<https://lwn.net/Articles/539265/>) why they thought it made sense to put kvmtool into the mainline repository. Ingo even [compared kernel tooling to Somalia](<https://lwn.net/Articles/539266/>), saying that it was made up of ""disjunct entities with not much commonality or shared infrastructure"", though, presumably, with fewer pirates. Few others came to the defense of kvmtool, leaving Ingo and Pekka to carry forward the argument on their own. 

Linus [responded](<https://lwn.net/Articles/539267/>) that he saw no convincing reason to put kvmtool in the mainline; indeed, he thought that tying kvmtool with the kernel could be retarding its development. He concluded with: 

So here, let me state it very very clearly: I will not be merging kvmtool. It's not about "useful code". It's not about the project keeping to improve. Both of those would seem to be *better* outside the kernel, where there isn't that artificial and actually harmful tie-in. 

That is probably the end of the discussion unless somebody can come up with a new argument that Linus will find more convincing. At this point, it seems that kvmtool is destined to remain out of the mainline kernel repository.
