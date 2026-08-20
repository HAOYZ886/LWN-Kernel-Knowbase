---
title: The kernel and character set encodings
url: https://lwn.net/Articles/71472/
date: "February 18, 2004"
category: "Character encoding; Filesystems-Case-independent lookups; JFS; UTF-8 encoding"
author: 
---

It all started as [a JFS bug report](<https://lwn.net/Articles/71477/>). The JFS filesystem, it seems, gets upset when user space passes it file names encoded in the UTF-8 format. Rather than create or open a file with the name as given, it gives up and returns `EINVAL`. Patches which fix the problem have been posted, but the resulting discussion has taken rather longer to be resolved. 

JFS has an "`iocharset`" option which can be used to state explicitly, at mount time, which character encoding is being used. There were calls on linux-kernel for this option to be added to other filesystems as well. The idea was rather strongly shot down, however, for a few reasons. One of those is that multiple users could be simultaneously using different character encodings on the same filesystem; a global option for the whole filesystem clearly will not be able to address that case. 

The real reason, however, is that performing character set conversion requires the kernel to interpret the file name strings being passed to it from user space. The kernel hackers are very resistant to the imposition of any such policy; it would go against decades of Unix tradition. Officially, the kernel has no policy regarding which character set is being used for file names, content, or anything else. In each case, the kernel sees nothing more than a stream of bytes. 

That said, the kernel does have some policies regarding file names: they use "`/`" as a directory delimiter, and they are terminated by a `NULL` byte. This policy rules out the use of many encodings which are sometimes employed to represent non-ASCII characters; the fixed-width wide encodings all tend to use lots of bytes containing zero. In reality, the only practical choices for representing characters beyond the ASCII set are iso-8859-1 (which allows the representation of characters used in many continental European languages) and UTF-8, which can encode pretty much anything. 

UTF-8 is relatively easy to use; for US users it looks just like ASCII, but it can handle a far wider range of characters while not breaking (most) code which uses traditional C strings. Thus it is often said that UTF-8 is the encoding used by the Linux kernel. That statement is a mistake, however: Linux does not use any particular encoding. If user space uses UTF-8 to represent extended characters, everything will work. But nothing forces user space to work in that way. 

This approach keeps policy out of the kernel, but some developers are not entirely happy with it. The lack of policy can lead to user-space confusion in a number of ways. For example, if a user creates a file called `WeirdÑame`, that name could be represented in the filesystem in more than one way. Depending on how user space is configured, it could choose either iso-8859-1 or UTF-8; the encoding of that name will be quite different depending on that choice. A different user space could interpret the file name differently in the future, resulting in unreadable filenames and confused users. The kernel, lacking a character encoding policy of its own, will do nothing to help prevent this situation. 

Confusion over character sets can also facilitate the creation of security holes; code which attempts to clean up file names can fail if evil characters are given in an unexpected encoding. Code which expects UTF-8 must also be careful when dealing with the Linux kernel because the kernel itself makes no effort to ensure that any string is, in fact, a legal UTF-8 encoding. 

To complicate the situation even more, Andrew Tridgell posted [another reason](<https://lwn.net/Articles/71487/>) why, he thinks, the kernel will have to adopt a specific character encoding: case insensitivity. Says Tridge: 

The reason is that I think that eventually the Linux kernel will need to efficiently support a userspace policy of case-insensitivity and the only way to do case-insensitive filename operations is to interpret those byte streams as a particular encoding. 

Needless to say, the idea of implementing case-insensitive filesystem operations in the kernel was not particularly popular. Not too many kernel hackers want to complicate the filesystem code to implement what they see as being a broken Windows feature to begin with. There are other difficulties as well: case-insensitive matching must be done differently in different languages. The end result is that case insensitive lookups are not very likely to make it into the kernel anytime soon. 

Linus is not averse to trying to help out Samba and other applications which wish to implement case-insensitive behavior, however. He has [proposed](<https://lwn.net/Articles/71496/>) a new "`magic_open()`" interface which would make it easier for user space to perform case-insensitive lookups without actually doing that work in the kernel. This interface would likely require quite a bit of work before it would do what the Samba developers need, but something derived from it could just make an appearance in the 2.7 development series. 

Meanwhile, the kernel does not seem likely to adopt any sort of official encoding anytime soon. The problems that result from the lack of an encoding policy are mostly seen as user space issues. Proper locale support is still relatively new in Linux, and many rough edges remain. Given the high level of interest in high-quality localization support in Linux, however, one might expect those edges to be smoothed down quickly. 

(For those who would like to learn more about UTF-8, see [this FAQ](<http://www.cl.cam.ac.uk/~mgk25/unicode.html>) or [RFC 3629](<http://www.ietf.org/rfc/rfc3629.txt>)).
