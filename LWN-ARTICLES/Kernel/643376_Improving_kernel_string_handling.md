---
title: Improving kernel string handling
url: https://lwn.net/Articles/643376/
date: "May 6, 2015"
category: String processing
author: "By Jonathan Corbet May 6, 2015"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
May 6, 2015

The handling and parsing of string data has long been acknowledged as a fertile breeding ground for bugs and security issues; that is doubly true when the C language — whose string model leaves a bit to be desired — is in use. Various attempts have been made to improve C string handling, both in the kernel and in user space, but few think that the problem has been solved. A couple of current projects may improve the situation on the kernel side, though. 

#### String copying

The venerable `strcpy()` family of functions has long been seen as error-prone and best avoided. In most settings, they are replaced with functions like `strncpy()` or `strlcpy()`. The [last time](<https://lwn.net/Articles/612244/>) your editor wrote about criticisms of `strlcpy()`, he was treated to a long series of incendiary emails from one of its supporters. So, for the purposes of this article, suffice to say that not all developers are fond of those functions. Even so, the kernel contains implementations of both, and there are over 1,000 call sites for each. 

That doesn't mean that there isn't room for improvement, though. Chris Metcalf thinks he has an improvement in the form of the proposed [`strscpy()`](<https://lwn.net/Articles/643140/>) API, which provides two new functions: 

```
ssize_t strscpy(char *dest, const char *src, size_t count);
        ssize_t strscpy_truncate(char *dest, const char *src, size_t count);
```

As with similar functions, `strscpy()` will copy a maximum of `count` bytes from `src` to `dest`, but it differs in the details. The return value in this case is the number of bytes copied, unless the source string is longer than `count` bytes; in that case, the return value will be `-E2BIG` instead. Another difference is that, in the overflow case, `dest` will be set to the empty string rather than a truncated version of `src`. 

This behavior is designed to make overflows as obvious as possible and to prevent code from blithely proceeding with a truncated string. When questioned on this behavior, Chris [justified](<https://lwn.net/Articles/643382/>) it this way: 

1\. A truncated string with an error return may still cause program errors, even if the caller checks for the error return, if the buffer is later interpreted as a valid string due to some other program error. It's defensive programming. 

2\. Programmers are fond of ignoring error returns. My experience with truncated strings is that in too many cases, truncation causes program errors down the line. It's better to ensure that no partial string is returned in this case. 

In a perfect world, all error returns would be checked, and there would be no need for this, but we definitely don't live in that world :-) 

For cases where the code can handle a truncated string, `strscpy_truncate()` can be used. Its return value convention is the same, but it will fit as much of the string as possible (null-terminated) in `dest`. 

#### Integer parsing

The kernel must often turn strings into integer values; the interpretation of numbers written to sysfs files or found on the kernel command line are a couple of obvious examples. This parsing can be done with functions like `simple_strtoul()` (which decodes a string to an `unsigned long`), but they were [marked as being obsolete](<http://git.kernel.org/linus/67d0a0754455f89ef3946946159d8ec9e45ce33a>) in 2011. The checkpatch script complains about their use, but there are still about 1,000 call sites in the kernel. Current advice is to use `kstrtoul()` and the better part of a dozen variants, also [added in 2011](<https://lwn.net/Articles/435022/>). There are almost 2,000 uses of these functions in the kernel, but Alexey Dobriyan [thinks we can do better](<https://lwn.net/Articles/643159/>). 

Alexey has a few complaints about the current APIs. One of the reasons for moving beyond the `simple_strto*()` functions was that they would silently stop conversion at a non-digit character — "`123abc`" would be successfully converted to 123. That is the sort of behavior for which PHP is [roundly criticized](<https://news.ycombinator.com/item?id=9484757>), but, Alexey says, there are times when it is useful. He gives the parsing of device numbers (usually given in the "`major:minor`" format) as an example. The `kstrto*()` family cannot easily be used for that kind of parsing, but there are plenty of reasons to not go back to `simple_strto*()` for that kind of work. 

His suggestion is the addition of a new function: 

```
int parse_integer(const char *s, unsigned int base, <type> *val);
```

In truth, `parse_integer()` is not a function; it is instead a rather unsightly macro that arranges to do the right thing for a wide variety of types for `val`. So, if `val` is an `unsigned short`, the decoding will be done on an unsigned basis and will be checked to ensure that the resulting value does not exceed the range of a `short`. 

A successful decoding will cause the result to be placed in `val`; the number of characters decoded will come back as the return value. If it is expected that the entire string will be decoded, a quick check to see whether `s[return_value]` is a null byte can verify that. Otherwise, parsing of the string can continue from the indicated point. If the `base` is ORed with the undocumented value `PARSE_INTEGER_NEWLINE`, a final newline character will be skipped over — useful for parsing input to sysfs files. If no characters at all are converted, the return value will be `-EINVAL`; an overflow will return `-ERANGE` instead. 

Alexey's patch set turns the `kstrto*()` functions into calls to `parse_integer()`; it also converts a number of `simple_strto*()` calls to direct `parse_integer()` calls. The end result is an apparent simplification of the code and net reduction in lines of code. 

Whether either of these patch sets will find its way into the kernel is not entirely clear; kernel developers do not, in general, tend to get too excited about string-parsing functions. In both cases, though, the potential exists for improvements to the massive amounts of parsing code found in the kernel while simultaneously making it simpler. In the end, most developers will find it hard to argue against something like that.
