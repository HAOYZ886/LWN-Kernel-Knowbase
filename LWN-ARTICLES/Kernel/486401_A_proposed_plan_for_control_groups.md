---
title: A proposed plan for control groups
url: https://lwn.net/Articles/486401/
date: "March 14, 2012"
category: Control groups
author: "By Jonathan Corbet March 14, 2012"
---

By **Jonathan Corbet**  
March 14, 2012

After the [late-February discussion on the future of control groups](<https://lwn.net/Articles/484251/>), Tejun Heo has boiled down the comments and [come to some conclusions](<https://lwn.net/Articles/486402/>) as to where he would like to go with this subsystem. The first of these is that multiple hierarchies are doomed in the long term: 

At least to me, nobody seems to have strong enough justification for orthogonal multiple hierarchies, so, yeah, unless something else happens, I'm scheduling multiple hierarchy support for the chopping block. This is a long term thing (think years), so no need to panic right now and as is life plans may change and fail to materialize, but I intend to at least move away from it. 

So there will, someday, be a single control group hierarchy. It will not, however, be tied to the process tree; it will be an independent tree of groups allowing processes to be combined in arbitrary ways. 

The responses to Tejun's conclusions have mostly focused on details (how to handle controllers that are not fully hierarchical, for example). There does not appear to be any determined opposition to the idea of removing the multiple hierarchy feature at some point when it can be done without breaking systems, so users of control groups should consider the writing to be on the wall.
