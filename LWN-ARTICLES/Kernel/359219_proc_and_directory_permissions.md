---
title: /proc and directory permissions
url: https://lwn.net/Articles/359219/
date: "October 28, 2009"
category: Security
author: "By Jake Edge October 28, 2009"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jake Edge**  
October 28, 2009

In a discussion of the [O_NODE open flag patch](<http://lwn.net/Articles/354186/>), an interesting, though obscure, security hole came to light. Jamie Lokier [noticed](<https://lwn.net/Articles/359224/>) the problem, and Pavel Machek eventually [posted](<http://seclists.org/bugtraq/2009/Oct/179>) it to the Bugtraq security mailing list. 

Normally, one would expect that a file in a directory with 700 permissions would be inaccessible to all but the owner of the directory (and root, of course). Lokier and Machek showed that there is a way around that restriction by using an entry in an attacking process's `fd` directory in the `/proc` filesystem. 

If the directory is open to the attacker at some time, while the file is present, the attacker can open the file for reading and hold it open even if the victim changes the directory permissions. Any normal write to the open file descriptor will fail because it was opened read-only, but writing to `/proc/$$/fd/N`, where N is the open file descriptor number, will succeed based on the permissions of the _file_. If the file allows the attacking process to write to it, writing to the `/proc` file will succeed regardless of the permissions of the parent directory. This is rather counter-intuitive, and, even though it is a rather contrived example, seems to constitute a security hole. 

The Bugtraq thread got off course quickly, by noting that a similar effect could be achieved creating a hardlink to the file before the directory permissions were changed. While that is true, Machek's example looked for that case by checking the link count on the file after the directory permissions had been changed. The hardlink scenario would be detected at that point. 

One can imagine situations where programs do not put the right permissions on the files they use and administrators attempt to work around that problem by restricting access to the parent directory. Using this technique, an attacker could still access those files, in a way that was difficult to detect. As Machek [noted](<http://seclists.org/bugtraq/2009/Oct/181>), unmounting the `/proc` filesystem removes the problem, but ""I do not think mounting /proc should change access control semantics."" 

There is currently some [discussion](<https://lwn.net/Articles/359229/>) of how, and to some extent whether, to address the problem, but a consensus (and patch) has not yet emerged.
