---
title: strscpy() and the hazards of improved interfaces
url: https://lwn.net/Articles/659214/
date: "October 7, 2015"
category: String processing
author: "By Jonathan Corbet October 7, 2015"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
October 7, 2015

Back in the distant past (May 2015), LWN [looked at](<https://lwn.net/Articles/643376/>) a couple of efforts to provide improved string-handling primitives to the kernel. One of those two was recently merged, while the other has run into trouble; both cases highlight a fundamental concern Linus has about this type of kernel patch. The end result is that it is possible to evolve the kernel toward safer interfaces, but attempts to do so as a series of mass changes will probably not end well. 

Normally, one does not expect to see a new API merged into the mainline for an -rc4 release, but Linus decided to make an exception when he pulled in the `strscpy()` patch just before 4.3-rc4. That patch set has changed a bit since it was examined here, though the intent is the same: to provide a string-copy API that is safer and easier to use than `strncpy()` or `strlcpy()`. The new copy function is: 

```
ssize_t strscpy(char *dest, const char *src, size_t count);
```

This function will copy the string found in `src` to `dest`, taking care not to overflow `dest`, which is `count` bytes long. Unlike `strncpy()`, it always null-terminates the destination string. The return value is the number of characters copied (without the trailing `NUL` byte) — unless the string would not fit into `dest`, in which case the return value is `-E2BIG`. 

Unlike previous versions, `strscpy()` will always copy what it can, returning a truncated string in `dest` if the whole thing does not fit. That change took away the need for the `strscpy_truncate()` variant, so that function is no longer provided. Opinions may differ on whether returning a truncated string is the right thing to do, but there were enough opinions in favor of doing so that this change needed to be made to get the patch merged. 

There are a number of advantages claimed for this API. It lacks an internal race condition found in the others, making it more robust in the face of a string that changes while it is being copied. The return value, it is claimed, more clearly indicates overflow than the value returned by `strlcpy()`. Unlike `strncpy()`, the result is always a null-terminated string. In the end, we might have finally come up with a reasonable string-copy function after about four attempts — not bad for such a complex task. 

Anybody who is firing up their editor to start converting call sites in the kernel to `strscpy()` may want to reconsider, though. There is a warning in both the fate of `parse_integer()` and Linus's comments around the merging of `strscpy()`. 

`parse_integer()` is the other string function covered in the May article; its purpose is to make string-to-integer conversions easier and more robust. Linus recently [got rather upset](<https://lwn.net/Articles/659215/>) about this patch set which, he thought, changed the semantics of the API and introduced bugs. Various call sites were changed to the new functions and, in the process, some of them were broken. The idea was that `parse_integer()` would be a replacement for the kernel's existing integer-conversion functions (`simple_strtoul()`, `kstrtoul()`, and the like) but that the actual act of replacing those functions introduced regressions. 

Linus was clearly afraid that the `strscpy()` patch could end up being a source of regressions as well. That wouldn't happen with the patch set itself, which does not convert any existing `strncpy()` or `strlcpy()` call sites. The problem happens when other, well-intentioned developers start doing those conversions. Linus described his worries in [the merge commit](<http://git.kernel.org/linus/30c44659f4a3e7e1f9f47e895591b4b40bf62671>) that brought in `strscpy()`: 

So why did I waffle about this for so long? 

Every time we introduce a new-and-improved interface, people start doing these interminable series of trivial conversion patches. 

And every time that happens, somebody does some silly mistake, and the conversion patch to the improved interface actually makes things worse. Because the patch is mindnumbing and trivial, nobody has the attention span to look at it carefully, and it's usually done over large swatches of source code which means that not every conversion gets tested. 

To try to head off such an outcome, Linus has made it clear that he will not be accepting patches that do mass conversions to `strscpy()` (note though that certain developers are already [considering mass conversions](<https://lwn.net/Articles/659474/>) anyway). It is there to be used with new code, but existing code should not be converted without some compelling reason to do so — or without a high level of attention to the possible implications of the change. 

One might be tempted to think that this proclamation from Linus signals the end of the "trivial clean-up patch" era. But that would almost certainly be reading too much into what he said. Patches that do not make functional changes to the code do not, one would hope, pose the same sort of risk that API replacements do. So the flow of white-space adjustments is likely to continue unabated. But developers who want to convert a bunch of working code to a "safer" interface may want to think twice before sending in a patch.
