---
title: Git approaches 1.0
url: https://lwn.net/Articles/145194/
date: "July 27, 2005"
category: "Development tools-Git; Git"
author: 
---

On April 5, 2005, it was announced that BitMover would "focus exclusively" on its commercial BitKeeper offering and withdraw the free-beer client used by a number of free software developers. This was a nervous moment; BitKeeper had become an integral part of the Linux kernel development process. Nobody wanted to go back to the old days - when no source code management system was used at all - but there was no clear successor to BitKeeper on offer. 

And where might such a successor have been expected to come from? We had been told many times that the development of BitKeeper required numerous person-years of work and millions of dollars of funding. The free software community was simply not up to the task of creating a tool with that sort of capabilities - especially not in a hurry. The kernel development community, having lost a tool it relied upon heavily, appeared doomed to a long painful period of adjustment. 

Two full days later, Linus [announced](<https://lwn.net/Articles/131312/>) the first release of a tool called "git." It was, he said, "_really_ nasty," but it was a starting point. On April 20, fifteen days after the withdrawal of BitKeeper, the 2.6.12-rc3 kernel prepatch, done entirely with git, was released. The git tool, in those days, was clearly suitable only for early adopters, but, even then, it was also clearly going somewhere. 

Git brings with it some truly innovative concepts; it is not a clone of any other source code management system. Indeed, at its core, it is not really an SCM at all. What git offers is a content-addressable object filesystem. If you store a file in git, it does not really have a name; instead, it can be looked up using its contents (as represented by an SHA hash). A hierarchical grouping of files - a particular kernel release, for example - is represented by a separate "tree" object listing which files are part of the group and where they are to be found. Files do not have any history - they simply exist or not, and two versions of the same file are only linked by virtue of being in the same place in two different tree objects. 

This way of organizing things is hard to grasp, initially, but it makes some interesting things possible. One of the harder problems in many SCM systems - handling the renaming of files - requires no special care with git. A single git repository can hold any number of branches or parallel trees without confusion. File integrity checking is built into the basic lookup mechanism, so that corruption will be detected automatically, and, if desired, kernel releases can be cryptographically signed easily. Perhaps most importantly, however: git made certain options, such as the merging of patches, very fast. 

It's worth noting that git is not a clone of BitKeeper, or of any other SCM. Certainly it incorporates lessons learned from years of use of BitKeeper and other tools; it supports changesets, for example, and is designed to be used in a distributed mode. But git is something new, it brings a unique approach to the problem. 

Watching the git development process snowball over the last few months has been fascinating. A large and active development community coalesced around git in short order; interestingly, relatively few of the core git developers were significant kernel contributors. In a short period of time, git has acquired most of the features expected from an SCM, its rough edges have been smoothed, it has picked up a variety of graphical interfaces, and it is widely used in the kernel development community. Git is clearly a success. 

The git developers are now working toward a 1.0 release. As part of that process, Linus has now [handed git over](<http://lwn.net/Articles/145123/>) to a new maintainer: Junio Hamano. Junio has been an active git developer for some time; he will now [attempt to take the project forward](<http://lwn.net/Articles/145124/>) as its leader. He will have plenty of work ahead of him as git moves into a more stable (though still fast-moving) phase. 

Git is an example of how well the free software process can work. Linus has shown us, once again, that he knows how to get a successful free software project started: put out a minimal (but well thought out) core that begins to solve a problem, then let the community run with it. The result is a vibrant, living project which incorporates the best of what has been learned before while simultaneously breaking new ground. The creator of the Linux kernel appears to have launched another winner. 

But, then, some things still seem to surprise even Linus: 

August 25, 1991| July 26, 2005  
---|---  
""I'm doing a (free) operating system (just a hobby, won't be big and professional like gnu) for 386(486) AT clones."" |  ""...this thing ended up being a bit bigger and more professional than I originally even envisioned.""   
  
Let this be a lesson to all free software developers out there: the humblest of projects can, with the right ideas and participation, become far more "big and professional" than one might ever imagine.
