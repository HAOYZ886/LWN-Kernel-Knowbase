---
title: "Digging into the community's lore with lei"
url: https://lwn.net/Articles/878205/
date: "December 13, 2021"
category: Development tools
author: "By Jonathan Corbet December 13, 2021"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
December 13, 2021

Email is often seen as a technology with a dim future; it is slow, easily faked, and buried in spam. Kids These Days want nothing to do with it, and email has lost its charm with many others as well. But many development projects are still dependent on it, and even non-developers still cope with large volumes of mail. While development forges show one possible path away from email, they are not the only one. What if new structures could be built on top of email to address some of its worst problems while keeping the good parts that many projects depend on? The "lei" system recently launched by Konstantin Ryabitsev is a hint of how such a future might look. 

One of the initial motivations for creating LWN, back in 1997, was to spare readers from the impossible task of keeping up with the linux-kernel mailing list. After all, that list was receiving an astounding 100 messages every day, and no rational human being would try to read such a thing. Some 24 years later, that situation has changed: linux-kernel now runs over 1,000 messages per day, and there are dozens of other busy, kernel-oriented mailing lists as well. It is easy to miss important messages while trying to follow that kind of traffic — and few developers even try. 

While much of the traffic that appears on any mailing list is quickly forgettable, some of it has lasting value; that means that good archives are needed. For most of the kernel project's history, those archives did not exist. There were indeed archives for most lists, but they were scattered, of mixed reliability, difficult to search, and usually incomplete. It is only a few years ago that Ryabitsev put together [lore.kernel.org](<https://lore.kernel.org>) to serve as a better solution to this problem. By using a search-friendly archiving system ([public-inbox](<https://lwn.net/Articles/748184/>)), building complete archives from pieces obtained from numerous sources, and archiving most kernel-oriented lists, Ryabitsev was able to create a resource that quickly became indispensable within the community. 

Lei (which stands for "local email interface") comes out of the public-inbox community. It works nicely with lore, to the point that Ryabitsev refers to the whole system as "lore+lei". The idea behind this combination is to create a new way of dealing with email that allows developers to see interesting messages without having to subscribe to an entire list. 

Public-inbox is built on some interesting ideas, including the use of Git to store the archive itself. The real key to its usefulness, though, is the use of [Xapian](<https://xapian.org/>) to implement a fast, focused search capability. The "fast" part allows for nearly instantaneous searches within the millions of messages in the email archive; [this query](<https://lore.kernel.org/all/?q=dromedary>), for example, shows immediately that the term "dromedary" has been used exactly 30 times in all of the lists archived on lore. 

The search mechanism differs from that found in many email clients in that it implements search terms that are useful in the context of a technical mailing list. So, while searching for "dromedary" finds occurrences of that word, "nq:dromedary", instead, only turns up occurrences that are not in quoted text being replied to. That [reduces the number of hits](<https://lore.kernel.org/all/?q=nq%3Adromedary>) to 13 without missing any of the original occurrences of the word. It is also possible to search for terms in a number of message headers, the names of files touched by patches, the names of functions changed by patches, and more; see [this page](<https://lore.kernel.org/all/_/text/help/>) for details. 

The purpose of lei, in short, is to take advantage of the search features built into public-inbox to give developers highly filtered views of mailing-list traffic. It takes a search query and uses it to populate a mailbox, which can then be perused with the user's client of choice. As an example, take this query, which appeared in [Ryabitsev's announcement of lei](<https://lwn.net/ml/workflows/lorelei.part1.202111051304.mdtebsxahljcrxak@meerkat.local/>) in November: 

```
lei q -I https://lore.kernel.org/all/ -o ~/Mail/floppy \
          --threads --dedupe=mid \
          '(dfn:drivers/block/floppy.c OR dfhh:floppy_* OR s:floppy \
          OR ((nq:bug OR nq:regression) AND nq:floppy)) \
          AND rt:1.month.ago..'
```

The idea here is to obtain all emails that might be relevant to the maintainer of the kernel's vitally important floppy driver. It looks for any of: 

  * patches that touches `floppy.c` (`dfn:drivers/block/floppy.c`) 
  * patches that change a function whose name starts with `floppy_` (`dfhh:floppy_*`) 
  * messages with "floppy" in the subject (`s:floppy`) 
  * messages that mention both "floppy" and either "bug" or "regression" in non-quoted text (`(nq:bug OR nq:regression) AND nq:floppy`) 

The search is limited to messages sent in the last month. The `lei` command will go to lore and execute this search on all mailing lists, collect the matched messages, and store them in the maildir folder `~/Mail/floppy`. They can then use whichever tools are best for working with the messages in that folder. 

Lei will remember this search, so updating the folder with new messages at a later time is a matter of a simple "`lei up`" command. As described in [this followup post](<https://lwn.net/ml/workflows/lorelei.part2.202111121411.sznnvkvcywfbdghl@meerkat.local/>), it is also possible to instruct lei to store the retrieved messages in a remote IMAP folder rather than a local maildir folder. 

The intent behind this work is clear: it lets kernel developers keep up with email traffic that is of interest to them without the need to subscribe to a set of high-volume mailing lists. No developer can subscribe to all lists where relevant messages might appear; with lore+lei, they no longer have to even try. It may also make email more reliable for many users. There is, for example, an ongoing low rumble of complaints regarding problems getting email from kernel lists delivered to Gmail users; use of lore+lei can route around such problems entirely. 

In one sense, lei follows the pattern we have seen in other parts of the Internet. Those of us who watched the World Wide Web develop all those years ago will remember the extensive efforts that went into trying to index the whole thing. Yahoo got started as a hierarchical directory of the web, for example. After a while it became clear that such efforts were hopeless and that search was the right tool for the job of finding things on the net. Lei is a statement that the same is true for email conversations; organizing our discussions into a set of topic-focused lists has gotten us far, but this approach is clearly under strain now. 

Maybe, the reasoning goes, it's time to forget about all those lists and just use searches instead. As a step in that direction, Ryabitsev has created [a mailing list for patches](<https://lwn.net/ml/workflows/20211129215634.gydsyfyhu6rhbbe2@meerkat.local/>) — all patches, regardless of which subsystem they affect. Developers are encouraged to copy their patch postings to this list. Anybody who is interested in specific patches can use tools like Lei to filter out the rest. 

What is potentially being lost here, of course, is the serendipity of finding interesting emails that one might never think to search for. If lei serves to further isolate kernel developers into their own niches, there could be an adverse effect on kernel development as a whole. Cross-subsystem discussions could become harder, and developers could lose awareness of what is happening elsewhere in the project. Filter bubbles are already a problem in the wider world; they could make it harder to maintain the cohesiveness of a large free-software project as well. 

Then again, such a world sounds like fertile ground for news sites providing a broad view of what's happening in the community, so perhaps it's not an entirely bad thing. 

Finally, there is one other aspect of this work that is worth thinking about. The functionality implemented in lore+lei is highly useful in its own right, but one could also argue that it's really just the database layer that will sit underneath a new generation of collaborative-development tools. The fact that those tools don't exist yet is inconvenient, but hopefully there are developers out there who are starting to think about how to fill that void. That would, in the end, finally provide a path for email-dependent free-software communities to, at least on the surface of things, move away from email and onto something better. The email-based infrastructure underlying it all could become an implementation detail that users need not worry about if it does not interest them. That could be a future worth working toward.
