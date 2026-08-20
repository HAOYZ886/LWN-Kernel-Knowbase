---
title: "Tightening security: not for the impatient"
url: https://lwn.net/Articles/503660/
date: "June 27, 2012"
category: Security; Symbolic links
author: "By Jonathan Corbet June 27, 2012"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
June 27, 2012

It has often been said that memory management patches can take a long time to be accepted into the mainline kernel. Because memory management performance regressions can take years to be discovered, developers in this area have become highly conservative; making memory management changes is not a recommended endeavor for those lacking patience. But there may be an area where progress can be even more glacial, for different reasons. Security-oriented changes are subject to arbitrary delays because tighter security can break programs and irritate users. 

Consider the classic symbolic link vulnerability, wherein an attacker fools a privileged program into writing to a file behind an attacker-controlled symbolic link. Such vulnerabilities can be exploited to overwrite files that the attacker would not otherwise have access to. One does not have to dig far into the [LWN vulnerability list](</Vulnerabilities/>) to see that the identification and patching of symbolic link vulnerabilities is an ongoing process. One might think that, if somebody could come up with a way to eliminate such vulnerabilities altogether, it would be adopted in a hurry. 

As it happens, Kees Cook [has a way](<https://lwn.net/Articles/503671/>) to deal with this class of vulnerabilities. It is based on the observations that symbolic link vulnerabilities almost always involve links placed in `/tmp`, and that `/tmp` has the "sticky" bit set in any contemporary distribution. Given that: 

The solution is to permit symlinks to only be followed when outside a sticky world-writable directory, or when the uid of the symlink and follower match, or when the directory owner matches the symlink's owner. 

In short, this change would make it so that nobody could create symbolic links in `/tmp` and expect a privileged program to follow them. Lest one think that Kees is taking credit for this concept, he posted a bit of history for this idea, starting with [a 1996 Bugtraq message](<http://marc.info/?l=bugtraq&m=87602167419830&w=2>) from Zygo Blaxell and [a kernel patch by Andrew Tridgell](<http://lkml.indiana.edu/hypermail/linux/kernel/9610.2/0086.html>) from the same year. This idea, in other words, has been floating around for at least 16 years, but an implementation has never found its way into the mainline kernel. Memory management changes are amazingly fast in comparison. 

The reason for the resistance, of course, is that this is a change in filesystem semantics. There are concerns that it would break POSIX compliance, though Kees claims that POSIX is silent on this particular behavior. Also of concern is the possibility of breaking existing applications. Kees responds that any broken applications would be easily noticed (while those suffering from symbolic link vulnerabilities are not), and that no applications relying on existing behavior have ever been found. There have also been disagreements over how the patch should be implemented, but those have seemingly mostly been resolved. 

So Kees thinks that his current [patch set](<https://lwn.net/Articles/503676/>) (a variant of one we have [seen](<https://lwn.net/Articles/390323/>) [before](<https://lwn.net/Articles/472071/>)) should be considered for merging, finally. The patches implement the symbolic link restrictions, but also add a new rule for hard links: a hard link to a file can only be created if the user owns the file or has write access to it. Once again, this change eliminates a class of attacks, but at a small cost: older versions of the "at" daemon break unless a small patch is applied. No other problems have been found, Kees says, after 1.5 years of experience with this patch in the Ubuntu kernel. 

Whether that is enough evidence to get the changes merged this time around remains to be seen. It has only been 16 years, after all, and one would not want to be too hasty about such a thing. 

Meanwhile, Kees has put together a separate security-oriented patch that has run into some concerns of its own. On Linux systems, there is a sysctl knob (`suid_dumpable`) that controls whether a crashing setuid process generates a core dump or not. Setting it to a non-zero value allows core dumps to happen; setting it to two applies certain restrictions that are intended to make it safe. But, Kees says, that's not the case; it allows a user to create a file called `core` in almost any directory, containing arbitrary text (environment strings, for example). This capability is not necessarily as harmless as one might think; as [the 2006 cron vulnerability](<https://lwn.net/Articles/191080/>) shows, some programs will happily pick out the strings they understand in a file full of junk, happily ignoring the rest. Thus, he claims, allowing users to create files in arbitrary locations is asking for trouble. 

His response has been through a number of iterations: 

  * [Version 1](<https://lwn.net/Articles/503678/>) disallowed storing core dumps from privileged executables into a file. If the `core_pattern` knob is set to a pipe, instead, core dumps happen as before. This was seen as an incompatible ABI change, though, and one that would cause surprising results. 

  * [Version 2](<https://lwn.net/Articles/503680/>) added a new setting (3) that would only allow setuid core dumps to a pipe. The previous "safe" setting (2) was deprecated; attempting to set it would fail with an EINVAL error. This version ran into trouble as a result of how it interacted with the sysctl mechanism. 

  * [Version 3](<https://lwn.net/Articles/503315/>) fixed the sysctl difficulties but was opposed by Andrew Morton, who feared that the deprecation of the previous mode would break current systems in surprising ways. He suggested keeping `suid_dumpable=2` as a working mode with a warning. 

  * [Version 4](<https://lwn.net/Articles/503681/>) went back to something closer to version 1, but with some loud warnings emitted. But then Eric Biederman asked whether disallowing relative paths would be a sufficient fix. 

  * Thus, [version 5](<https://lwn.net/Articles/503682/>) (the current version, as of this writing), just disallows the writing of setuid core dumps to relative paths. Should `core_pattern` be set to a relative path ("`core`", for example), a warning will be logged instead. 

Thus far, there has not been much in the way of complaints about the fifth iteration of the patch. So, possibly, it will not be necessary to wait for years until this particular bit of security tightening gets into the mainline kernel. Of course, unlike the system's link behavior, the core dump behavior can be changed now by concerned system administrators—no need to wait at all.
