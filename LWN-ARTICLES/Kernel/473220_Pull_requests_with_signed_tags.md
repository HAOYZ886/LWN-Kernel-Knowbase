---
title: Pull requests with signed tags
url: https://lwn.net/Articles/473220/
date: "December 21, 2011"
category: "Security-Patch verification"
author: "By Jonathan Corbet December 21, 2011"
---

By **Jonathan Corbet**  
December 21, 2011

One of the ongoing echoes from the compromise of kernel.org is an increased interest in verifying the integrity of pull requests sent to Linus. One way of doing that is for the developer to add a cryptographic signature to the email containing the pull request. If the top commit ID is included in the message, the pull request (and the code it covers) can be authenticated, but the digital signature itself is not stored in the mainline repository, making it hard to re-verify requests at some future time. 

An alternative is to use git to create a signed tag, which stores the signature in the repository itself. In the future, that may become the accepted way to get code into the mainline. Linus has [described](<https://lwn.net/Articles/473222/>) some pending changes to git that make the capture and storage of that information simple. So simple, in fact, that there is no longer any need to worry about branches or unique tag names: 

Everybody: you can now create a signed tag, and just point me at it. You don't even have to have a separate branch for me to pull any more, just the signed tag is fine. 

So it would actually be nicer if you used temporary tag names the way you use temporary branch names when you ask me to pull. The tag *content* will be saved from now on (unless I screw up while traveling or something and pull with a machine that has an older git version), so there's very little advantage in then saving the tags separately by having ugly tag-names with long lifetimes. 

All of this evidently works now, with existing stable git releases; only the process of merging such a tag requires the newer code. So, soon, signed tags may be the standard way to identify changes to be pulled.
