---
title: Ushering out strlcpy()
url: https://lwn.net/Articles/905777/
date: "August 25, 2022"
category: String processing
author: "By Jonathan Corbet August 25, 2022"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
August 25, 2022

With all of the complex problems that must be solved in the kernel, one might think that copying a string would draw little attention. Even with the hazards that C strings present, simply moving some bytes should not be all that hard. But string-copy functions have been a frequent subject of debate over the years, with different variants being in fashion at times. Now it seems that the BSD-derived [`strlcpy()`](<https://man.openbsd.org/strlcpy.3>) function may finally be on its way out of the kernel. 

In the beginning, copying strings in C was simple. Your editor's dog-eared, first-edition copy of _The C Programming Language_ provides an implementation of `strcpy()` on page 101: 

```
strcpy(s, t)
        char *s, *t;
        {
            while (*s++ = *t++)
    	    ;
        }
```

This function has a few shortcomings, the most obvious of which is that it will overrun the destination buffer if the source string is too long. Developers working in C eventually concluded that this could be a problem, so other string-copying functions were developed, starting with [`strncpy()`](<https://man7.org/linux/man-pages/man3/strncpy.3p.html>): 

```
char *strncpy(char *dest, char *src, size_t n);
```

This function will copy at most `n` bytes from `src` to `dest`, so, if `n` is no larger than the length of `dest`, then that array cannot be overrun. `strncpy()` has a couple of quirks, though. It is defined to NUL-fill `dest` if `src` is shorter than `n`, so it ends up always writing the full array. If `src` is longer than `n`, then `dest` will not be NUL-terminated at all — an invitation to trouble if the caller does not carefully check the return value. That return value is ~~the address of the first NUL character written to` dest` unless `src` is too long, in which case `strncpy()` returns `&dest[n]` — an address beyond the actual array~~ `dest` regardless of whether truncation occurs or not. As a result, checking for truncation is a bit tricky and often not done. [Thanks to Rasmus Villemoes for pointing out the error in our earlier description of the `strncpy()` return value.] 

#### `strlcpy()` and `strscpy()`

The BSD answer to the problems with `strncpy()` was to introduce a new function called `strlcpy()`: 

```
size_t strlcpy(char *dest, const char *src, size_t n);
```

This function, too, will copy a maximum of `n` bytes from `src` to `dest`; unlike `strncpy()`, it will always ensure that `dest` is NUL-terminated. The return value is always the length of `src` regardless of whether it was truncated in the copy or not; developers must compare the returned length against `n` to determine whether truncation has occurred. 

The first uses of `strlcpy()` in the kernel entered briefly during the 2.4 stable series — sort of. The media subsystem had a couple of implementations defined as: 

```
#define strlcpy(dest,src,len) strncpy(dest,src,(len)-1)
```

As one might imagine, there was not a lot of checking of return values going on at that point. That macro disappeared relatively quickly, but a real `strlcpy()` implementation appeared in the [2.5.70 release](<https://lore.kernel.org/lkml/Pine.LNX.4.44.0305261903330.2164-100000@home.transmeta.com/>) in May 2003; that release also converted many callers in the kernel over to this new function. Everything seemed good for quite some time. 

In 2014, though, criticism of `strlcpy()` started to be heard, resulting in, among other things, an [extended discussion](<https://lwn.net/Articles/612244/>) over whether to add an implementation to the GNU C library; to this day, glibc lacks `strlcpy()`. Kernel developers, too, started to feel disenchanted with this API. In 2015, [yet another string-copy function](<https://lwn.net/Articles/659214/>) was added to the kernel by Chris Metcalf: 

```
ssize_t strscpy(char *dest, const char *src, size_t count);
```

This function, like the others, will copy `src` to `dest` without overrunning the latter. Like `strlcpy()`, it ensures that the result is NUL-terminated. The difference is in the return value; it is the number of characters copied (without the trailing NUL byte) if the string fits, and `-E2BIG` otherwise. 

#### Reasons to like `strscpy()`

Why is `strscpy()` better? One claimed advantage is the return value, which makes it easy to check whether the source string was truncated or not. There are a few other points as well, though; to get into those, it is instructive to look at [the kernel's implementation of `strlcpy()`](<https://elixir.bootlin.com/linux/v5.19.3/source/lib/string.c#L125>): 

```
size_t strlcpy(char *dest, const char *src, size_t size)
        {
    	size_t ret = strlen(src);
    
    	if (size) {
    	    size_t len = (ret >= size) ? size - 1 : ret;
    	    memcpy(dest, src, len);
    	    dest[len] = '\0';
    	}
    	return ret;
        }
```

One obvious shortcoming is that this function will read the entire source string regardless of whether that data will be copied or not. Given the defined semantics of `strlcpy()`, this inefficiency simply cannot be fixed; there is no other way to return the length of the source string. This is not just a question of efficiency, though; as recently [pointed out](<https://lwn.net/ml/linux-kernel/CAHk-=wi+xbVq++uqW9YgWpHjyBHNB8a-xad+Xp23-B+eodLCEA@mail.gmail.com/>) by Linus Torvalds, bad things can happen if the source string is untrusted — which is one of the intended use cases for this function. If `src` is not NUL-terminated, then `strlcpy()` will continue merrily off the end until it _does_ find a NUL byte, which may be way beyond the source array — if it doesn't crash first. 

Finally, `strlcpy()` is subject to a race condition. The length of `src` is calculated, then later used to perform the copy and returned to the caller. But if `src` changes in the middle, strange things could happen; at best the return value will not match what is actually in the `dest` string. This problem is specific to the implementation rather than the definition, and could thus be fixed, but nobody seems to think it's worth the effort. 

The [implementation of `strscpy()`](<https://elixir.bootlin.com/linux/v5.19.3/source/lib/string.c#L151>) avoids all of these problems and is also more efficient. It is also rather more complex as a result, of course. 

#### The end of `strlcpy()` in the kernel?

When `strlcpy()` was first introduced, the intent was to replace all of the `strncpy()` calls in the kernel and get rid of the latter function altogether. In the 6.0-rc2 kernel, though, there are still nearly 900 `strncpy()` call sites remaining; that number grew by two in the 6.0 merge window. At the introduction of `strscpy()`, instead, Torvalds explicitly [did not want](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=30c44659f4a3e7e1f9f47e895591b4b40bf62671>) to see any sort of mass conversion of `strlcpy()` calls. In 6.0-rc2, there are just over 1,400 `strlcpy()` calls and nearly 1,800 `strscpy()` calls. 

Nearly seven years later, the attitude seems to have changed a bit; Torvalds now says that ""strlcpy() does need to go"". A number of subsystems have made conversion passes, and the number of `strlcpy()` call sites has fallen by 85 since 5.19. Whether it will ever be possible to remove `strlcpy()` entirely is unclear; `strncpy()` is still holding strong despite its known hazards and a decision to get rid of it nearly 20 years ago. Once something gets into the kernel, taking it out again can be a difficult process. 

There may be hope, though, in this case. As Torvalds [observed](<https://lwn.net/ml/linux-kernel/CAHk-%3DwjGmhaE-Y8GqWKPtWYOi%3DbOarFgo7UkzHNoOVEKnkXXrQ%40mail.gmail.com/>) in response to a set of conversions from Wolfram Sang, most of the callers to `strlcpy()` never use the return value; those could all be converted to `strscpy()` with no change in behavior. All that would be needed, he suggested, was for somebody to create a [Coccinelle](<https://coccinelle.gitlabpages.inria.fr/website/>) script to do the work. Sang [rose to the challenge](<https://lwn.net/ml/linux-kernel/YvhXzarjOLEJ8nsW@shikoro/>) and has created [a branch with the conversions done](<https://git.kernel.org/pub/scm/linux/kernel/git/wsa/linux.git/log/?h=renesas/strlcpy>). That work, obviously, won't be considered for 6.0, but might show up in a 6.1 pull request. 

That would leave relatively few `strlcpy()` users in the kernel. Those could be cleaned up one by one, and it might just be possible to get rid of `strlcpy()` entirely. That would end a 20-year sporadic discussion on the best way to do bounded string copies in the kernel — all of those remaining `strncpy()` calls notwithstanding — at least until some clever developer comes up an even better function and starts the whole process anew.
