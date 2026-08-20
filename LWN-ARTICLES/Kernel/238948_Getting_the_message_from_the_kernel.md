---
title: Getting the message from the kernel
url: https://lwn.net/Articles/238948/
date: "June 19, 2007"
category: Messages
author: 
---

As a general rule, Linux users would rather not hear from their kernel. If all is well, devices are working, applications are running, and the kernel just quietly makes it all happen. When things go wrong, however, it may become necessary to dig through the messages that the kernel puts out. These messages sometimes make sense to the developers who created them, but they are not always clear to the rest of the world. Neal Stephenson, in his [_In the Beginning was the Command Line_](<http://project.cyberpunk.ru/lib/in_the_beginning_was_the_command_line/>), describes Linux kernel messages as having ""the semi-inscrutable menace of graffiti tags."" For a kernel developer, often as not, the main value of a kernel message is to pinpoint the location of the complaining code - from which the real problem can be determined. 

Non-developers have a harder time using kernel messages in that way, though, and people who are not native English speakers are at even more of a disadvantage. So it is not surprising that the topic of fixing up kernel messages has popped up occasionally. It's back, possibly in a more serious form this time around. 

People who would reform kernel messages generally have two goals in mind: 

  * They would like for every message to have a unique identifier attached to it. This idea brings back memories of VMS or most IBM operating systems, which have used message identifiers for decades. The main purpose behind message identifiers is to allow the system administrator (or the support person they have called) to look up the identifier in a manual and figure out what the message is really saying. Various legacy operating systems have come with message manuals which take up significant amounts of shelf space; they contain a (relatively) detailed explanation of the problem and suggestions for how to make the problem go away. 

  * It is much easier to maintain translations for messages which have unique identifiers attached to them. A Linux system which could output messages in multiple languages would be more approachable for much of the potential user base. 

The problem, of course, is that attaching identifiers to messages is a significant job. There are tens of thousands of `printk()` calls in the kernel; each of them would need to have an identifier assigned and the code changed. New messages are added - in large numbers - with every kernel release; it's easy to imagine that the overhead of putting identifiers onto all of those messages would irritate developers in a hurry. For these reasons, Linus has, in the past, [rejected](<http://lwn.net/Articles/28917/>) schemes aimed at improving kernel messaging. 

The idea has come back anyway. [A new approach](<https://lwn.net/Articles/238961/>) has been proposed by users in Japan who are having trouble supporting Linux as well as they would like. In this scheme, every kernel message would be assigned a component name and a message number. The component would be a per-file define: 

```
#define KMSG_COMPONENT "railgun"
```

Then `printk` calls would be modified to include the message number: 

```
printk(KMSG_ERR(100) "Rail gun fired accidentally - sorry\n")
```

The end result would be a message prepended with the string "`railgun.100:`", enabling the message to be translated or looked up in a manual. To help ensure that there _is_ a manual, the proposal requires kerneldoc-style documentation of messages within the source; something like: 

```
/**
         * message
         * @100: 
         *
         * Description:
         * The rail gun fired accidentally in the absence of a specific 
         * user request.  
         *
         * User Response:
         * Operator should be sure to stand to the side.
         */
```

The kerneldoc scripts would be upgraded to collect all of these message descriptions and turn them into a printable manual. Another tool would check source files and complain about messages which lack accompanying descriptions. 

Schemes like this have been greeted with complaints in the past, and the same happened this time around. The overhead of documenting messages in this way is more than many developers want to take on; David Miller [expressed this feeling](<https://lwn.net/Articles/238962/>) well: 

I think my general response to something like this, if it goes in, would be to stop emitting useful kernel log messages in the code I write because having to document it too on top of that is just too much extra work to be worthwhile. 

Keeping the message descriptions current would also be a challenge - code is often changed without updating the neighboring comments; there is no reason to believe that message descriptions would get a higher level of attention. 

Andrew Morton has come back with [a counter proposal](<https://lwn.net/Articles/238963/>) designed for easier developer acceptance. His scheme would add a new form of `printk()` which would take a message ID in some as-yet-undetermined format. That ID would be output with the message, but everything else - translations, descriptions, condolences, etc. - would be kept in a database outside of the kernel. 

The key point is that developers would not be expected to do much of anything with this database - or even with their kernel messages. Instead, there would be a "kernel messages team" charged with maintaining this information. Occasionally somebody from that team would look over new code, add message IDs where needed, and send a patch to the maintainer. Unless they were personally interested in helping, developers would not have to worry about the new mechanism at all. 

There are a few gaps in this proposal; how the kernel message team would be funded (or otherwise motivated) is one of them. But it may be sufficiently low-impact to be accepted by the rest of the development community. Someday soon, Linux users, too, may have to make room on their shelves for a hefty messages manual.
