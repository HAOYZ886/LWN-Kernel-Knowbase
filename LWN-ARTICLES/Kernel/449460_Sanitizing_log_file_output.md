---
title: Sanitizing log file output
url: https://lwn.net/Articles/449460/
date: "June 29, 2011"
category: Kernel messages
author: "By Jake Edge June 29, 2011"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jake Edge**  
June 29, 2011

Handling user-controlled data properly is one of the basic principles of computer security. Various kernel log messages allow user-controlled strings to be placed into the messages via the "`%s`" format specifier, which could be used by an attacker to potentially confuse administrators by inserting control characters into the strings. So Vasiliy Kulikov has proposed a [patch](<https://lwn.net/Articles/449531/>) that would escape certain characters that appear in those strings. There is some question as to _which_ characters should be escaped, but the bigger question is an age-old one in security circles: whitelisting vs. blacklisting. 

The problem stems from the idea that administrators will often use tools like `tail` and `more` to view log files on a TTY. If a user can insert control characters (and, in particular, escape sequences) into the log file, they could potentially cause important information to be overlooked--or cause other kinds of confusion. In the worst case, escape sequences could potentially exploit some hole in the terminal emulator program to execute code or cause other misbehavior. In the patch, Kulikov gives the following example: ""Control characters might fool root viewing the logs via tty, e.g. using ^[1A to suppress the previous log line."" For characters that are filtered, the patch simply replaces them with "#xx", where xx is the hex value of the character. 

It's a fairly minor issue, at some level, but it's not at all clear that there is any legitimate use of control characters in those user-supplied strings. The strings could come from various places; two that were mentioned in the discussion were filenames or USB product ID strings. The first [version](<https://lwn.net/Articles/449529/>) of the patch clearly went too far by escaping characters above 0x7e (in addition to control characters), which would exclude Unicode and other non-ASCII characters. But after complaints about that, Kulikov's second version just excludes control characters (i.e. < 0x20) with the exception of newline and tab. 

That didn't sit well with Ingo Molnar, however, who [thought](<https://lwn.net/Articles/449566/>) that rather than whitelisting the known-good characters, blacklisting those known to be potentially harmful should be done instead: 

Also, i think it would be better to make this opt-out, i.e. exclude the handful of control characters that are harmful (such as backline and console escape), instead of trying to include the known-useful ones. 

[...] It's also the better approach for the kernel: we handle known harmful things and are permissive otherwise. 

But, in order to create a blacklist, one must carefully determine the effects of the various control characters on all the different terminal emulators, whereas the whitelist approach has the advantage of being simpler by casting a much wider net. As Kulikov [notes](<https://lwn.net/Articles/449568/>), figuring out which characters are problematic is not necessarily simple: 

Could you instantly answer without reading the previous discussion what control characters are harmful, what are sometimes harmful (on some ttys), and what are always safe and why (or even answer why it is harmful at all)? I'm not a tty guy and I have to read console_codes(4) or similar docs to answer this question, the majority of kernel devs might have to read the docs too. 

The disagreement between Molnar and Kulikov is one that has gone on in the security world for many years. There is no _right_ answer as to which is better. As with most things in security (and software development for that matter), there are tradeoffs between whitelists and blacklists. In general, for user-supplied data (in web applications for example), the consensus has been to whitelist known-good input, rather than attempting to determine all of the "bad" input to exclude. At least in this case, though, Molnar [does not see whitelists as the right approach](<https://lwn.net/Articles/449572/>): 

A black list is well-defined: it disables the display of certain characters because they are *known to be dangerous*. 

A white list on the other hand does it the wrong way around: it tries to put the 'burden of proof' on the useful, good guys - and that's counter-productive really. 

It won't come as a surprise that Kulikov [disagreed](<https://lwn.net/Articles/449573/>) with that analysis: ""What do you do with dangerous characters that are *not yet known* to be dangerous?"" While there is little question that whitelisting the known-good characters is more secure, it is less flexible if there is a legitimate use for other control characters in the user-supplied strings. In addition, Molnar is [skeptical](<https://lwn.net/Articles/449577/>) that there are hidden dangers lurking in the ASCII control characters: ""This claim is silly - do you claim some 'unknown bug' in the ASCII printout space?"" 

In this particular case, either solution should be just fine, as there aren't any good reasons to include those characters, but Molnar is probably right that there aren't hidden dangers in ASCII. There is a question as to whether this change is needed at all, however. The concern that spawned the patch is that administrators might miss important messages or get fooled by carefully crafted input (Willy Tarreau provides an [interesting example](<https://lwn.net/Articles/449587/>) of the latter). Linus Torvalds is [not convinced](<https://lwn.net/Articles/449588/>) that it is really a problem that needs addressing: 

I really think that user space should do its own filtering - nobody does a plain 'cat' on dmesg. Or if they do, they really have themselves to blame. 

And afaik, we don't do any escape sequence handling at the console level either, so you cannot mess up the console with control characters. 

And the most dangerous character seems to be one that you don't filter: the one we really do react to is '\n', and you could possibly make confusing log messages by embedding a newline in your string and then trying to make the rest look like something bad (say, an oops). 

Given Torvalds's skepticism, it doesn't seem all that likely this patch will go anywhere even if it were changed to a blacklisting approach as advocated by Molnar. It is, or should be, a fairly minor concern, but the question about blacklisting vs. whitelisting is one we will likely hear again. There are plenty of examples of both techniques being used in security (and other) contexts. It often comes down to a choice between more security (whitelisting typically) or more usability (blacklisting). This case is no different, really, and others are sure to crop up.
