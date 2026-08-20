---
title: "Who maintains this file?"
url: https://lwn.net/Articles/246201/
date: "August 21, 2007"
category: Development tools
author: "By Jonathan Corbet August 21, 2007"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
August 21, 2007

Kernel developers are generally encouraged to split patches into small pieces before posting them to the mailing lists. Making each change self-contained and easy to understand helps reviewers do their job and is thus a good thing. That said, anybody who doubted that one can get too much of a good thing surely learned the truth when Joe Perches submitted [this patch set](<http://lwn.net/Articles/245340/>) made up of almost 550 patches, all to the same file. It is fair to say that this deluge of patches was not universally welcomed. 

Packaging aside, the ultimate goal of Joe's patch was not particularly controversial: he would like to make it possible to easily find out who is the maintainer of a specific file in the kernel tree. So, for each entry in the `MAINTAINERS` file, he added one or more lines with patterns describing which files belong to that entry. With that information in place, his `get_maintainer.pl` script can quickly identify who is responsible for any file in the tree. No more digging through `MAINTAINERS` or trying to extract email addresses from copyright notices in the source. 

It's an appealing idea, but nobody seems to be entirely clear on how to implement it. Keeping this information in a central file has a number of obvious disadvantages. It would clearly go out of date quickly, for example. The `MAINTAINERS` file tends to get stale as it is; the chances of it being patched for every new or renamed file seem quite small. If developers, contrary to expectations, do keep this file up to date, one can expect large numbers of conflicts as all the resulting patches try to touch the same file. 

The patch conflict problem could be mitigated by splitting up the `MAINTAINERS` file into per-directory versions, much like what was done with the kernel configuration file in the past. There are now over 400 `Kconfig` files in the mainline tree; some developers have expressed dismay at the idea of similar numbers of `MAINTAINERS` files being scattered around the tree. And, in any case, per-directory files aren't much more likely to be updated than the single, central file. 

So around came another idea: why not just put the maintainer information into the source files? The result would be nicely split documentation which gets put in front of the relevant developers every time they edit the file. The record for maintenance of documentation in the code is far from perfect, but it is much better than the record for completely out-of-line documentation. 

One question which comes up when this approach is considered is whether the resulting information should go into the binary kernel image or not. It would be easy to define a new tag like: 

```
MODULE_MAINTAINER("Your name here");
```

The provided information could then go into a special section in the kernel image where special tools could find it. Doing things this way would make it possible for people who don't have a kernel tree handy to look up a maintainer. On the other hand, it would bloat the kernel image and fix information in a binary, widely-distributed form where it could persist long after it goes out of date. So ex-maintainers could continue receiving mail for years after they have changed all of the relevant documentation. 

An alternative would be to just put the maintainer information at the top of the file as a comment. Then it would only be in the source, and would, presumably, be relatively easy to keep up to date. At least, until, say, a mailing list for a major subsystem moves and all of the associated source files have to be changed. For example, Adrian Bunk [noted](<https://lwn.net/Articles/246262/>) that the move of the netdev mailing list to vger would have forced patches to about 1300 files. 

Yet another approach is to find a way to store the information in the git repository. Git already maintains quite a bit of metadata about source files; to some it seems natural to add maintainer information as well. So far, the git developers have not shown a lot of appetite for adding this sort of feature. But Linus did [point out](<https://lwn.net/Articles/246204/>) that one could already use git to a similar effect with a simple command: 

Do a script like this: 

```
#!/bin/sh
    	git log --since=6.months.ago -- "$@" |
    		grep -i '^    [-a-z]*by:.*@' |
    		sort | uniq -c |
    		sort -r -n | head
```

and it gives you a rather good picture of who is involved with a particular subdirectory or file. 

The advantage of doing things this way is that the resulting output gives a current picture of who has actually been working on a file - a picture which requires no explicit maintenance at all. That list of people is probably a much better group to send copies of patches to than whoever might be listed in a maintainers file; they are the ones who know about what is happening in that part of the tree now. 

No real resolution has been reached on this topic. It may be that Linus's approach may be the one taken by default; it already works without the need to merge any patches at all. The question may well stay around for a while, though. Approximately 2,000 developers put patches into the mainline over the course of one year; keeping track of which of those developers is the best to notify of changes to a particular file is never going to be easy.
