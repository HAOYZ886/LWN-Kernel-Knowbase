---
title: "Finding a patch's kernel version with git"
url: https://lwn.net/Articles/392293/
date: "June 16, 2010"
category: "Development tools-Git; Git"
author: "By Jake Edge June 16, 2010"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
June 16, 2010

Back in May, Jan Kara posted a VFS [patch](<https://lwn.net/Articles/392303/>) that fixed a regression and he sent the patch to the stable tree folks as well. Linus Torvalds [noted](<https://lwn.net/Articles/392304/>) that it had been introduced in the merge window, so it wasn't relevant for the stable tree. That led to a discussion about how to figure out which kernel version includes a particular patch. While the conversation is a month old, the advice is pretty much timeless. 

Andrew Morton's [method](<https://lwn.net/Articles/392305/>) is rather sub-optimal: ""I just keep lots of kernel trees around and poke about with `patch \--dry-run'. PITA."" Christoph Hellwig and James Bottomley both suggested `git-describe <revid>`, which will show the tag of the version a patch was applied to, or was pulled into if you use the `--contains` flag. As one might guess, though, Torvalds had some more [elaborate suggestions](<https://lwn.net/Articles/392306/>). One can use `git name-rev` in much the same way as `git-describe --contains`, but a more ""obscure"" way to get the same kind of information is: 

```
git log --tags --source --author=viro --oneline fs/namei.c
```

which shows commits by Al Viro of `fs/namei.c` along with the tagged version each commit was included into. On a recent kernel tree, the start of that output looks like: 

```
d83c49f v2.6.34 Fix the regression created by "set S_DEAD on unlink()..." commit
        3e297b6 v2.6.34-rc3 Restore LOOKUP_DIRECTORY hint handling in final lookup on op
        781b167 v2.6.34-rc2 Fix a dumb typo - use of & instead of &&
        1f36f77 v2.6.34-rc2 Switch !O_CREAT case to use of do_last()
```

While the specific example Torvalds gave might not be widely applicable, the basic idea behind it is. Using `git-blame` to track down the commit where a particular change was made is often useful, but the dates in the log can be misleading with regards to which kernel(s) the change ended up in. Using some combination of `describe` and `log` will make figuring those kinds of things out much easier.
