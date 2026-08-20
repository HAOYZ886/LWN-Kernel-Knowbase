---
title: "Audio latency - resource limits win"
url: https://lwn.net/Articles/134460/
date: "May 3, 2005"
category: Latency
author: 
---

The long debate on how to provide preferential scheduling for audio applications would appear to have come to an end. The realtime Linux security module has not been merged; instead, the mainline now includes [a version of the rlimit patch](<https://lwn.net/Articles/134467/>). This is not the outcome which was most favored by the audio development community, but it will still be useful for them. 

The patch creates two new resource limits. `RLIMIT_NICE` controls the maximum "niceness" that the process can set for itself in the normal timesharing scheduler. The limit has a range of `0..39`, with 39 corresponding to an internal niceness value of -20 - the highest priority. The difference between the resource limit value and the actual niceness values may seem confusing, but apparently it's unavoidable: the Single Unix Standard specifies that resource limits must be unsigned values. 

The other limit is `RLIMIT_RTPRIO`; it can have a range of `0..100`. If it is nonzero, the process is empowered to use the realtime scheduling classes up to the indicated priority. 

The problem with this approach, from the point of view of the audio community, is that it is not currently supported by any distribution. It is easy to set up PAM to give expanded limits to specific users or groups - once PAM has been patched to understand the new limits. Shells, too, must be patched before their `ulimit` commands can be used to change the limits. So it will be some time before an "out of the box" Linux system will be able to take advantage of this new capability. 

In the long term, however, the rlimit patch looks like a minimally invasive way of making realtime scheduling available, in a relatively safe way, to ordinary users. Anybody wanting to play with the new mechanism before their distribution catches up can find instructions and patches on [this web page](<http://www.steamballoon.com/wiki/Rlimits>).
