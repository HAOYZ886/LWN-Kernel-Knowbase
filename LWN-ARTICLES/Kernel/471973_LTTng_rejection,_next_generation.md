---
title: "LTTng rejection, next generation"
url: https://lwn.net/Articles/471973/
date: "December 14, 2011"
category: "Development tools-Kernel tracing; Tracing"
author: "By Jonathan Corbet December 14, 2011"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
December 14, 2011

The story of tracing in the Linux kernel sometimes seems to resemble a bad multi-season TV soap opera. We have no end of strong characters, plot twists, independent story lines, recurring themes, and conflicting agendas. The cast changes slowly over time, but things never seem to come to any sort of satisfying conclusion. Those watching the show might be forgiven for thinking that one of those story lines might be about to be wrapped up when the [LTTng tracing system](<http://lttng.org/>) was [pulled into the staging tree](<https://lwn.net/Articles/471975/>) for the 3.3 merge window. But they should have known that they were just being set up for another sad twist in the plot. 

LTTng descends from some of the earliest dynamic tracing work done for Linux. Its distinguishing characteristics include integrated kernel- and user-space tracing, performance sufficient to deal with high-bandwidth event streams, and a well-developed set of capture and analysis tools. LTTng has always been maintained out of the mainline kernel tree, but it is packaged by a number of distributors and has base of dedicated users, some of whom have been happy to fund ongoing LTTng development work. 

Had LTTng been merged years ago, the story may have been much simpler, but, for a number of reasons (including the simple fact that, for years, any sort of tracing capability was hard to sell to the kernel development community) that did not happen. So we have ended up with a number of projects in this area, including SystemTap (which also remains out-of-tree), and the in-tree ftrace and perf subsystems. Naturally, none of these solutions has proved entirely satisfactory so, while there has been a fair amount of pressure to consolidate the various tracing projects, that has tended not to happen. 

That is not to say that there has been no progress at all. Some agreement has been reached on the format of tracepoints themselves; much of the work in that area was done by primary LTTng maintainer Mathieu Desnoyers. As a result, the number of tracepoints in the kernel has been growing rapidly, making kernel operations more visible to users in a number of ways. A lot of talk about merging more infrastructure has been heard over the years - said talk was often audible from a great distance at various conferences - but progress has been minimal. It seems to be easy for developers in this area to get bogged down on the details of ring buffers, event formats, and so on at the expense of producing an actual, usable solution. 

To Mathieu, merging into the staging tree must have looked like an attractive way to push things forward. The relaxed rules for that tree would allow the code into the mainline where its visibility would increase, any remaining issues could be fixed up, and more users could be found. It all seemed to be working - some cleanup patches from new developers were posted - until Mathieu [tried to add exports](<https://lwn.net/Articles/471982/>) for some core kernel symbols so LTTng could access them. That attracted the attention of the core kernel developers who, to put it gently, were not impressed with what they saw. 

In the end, Ingo Molnar [vetoed the whole patch series](<https://lwn.net/Articles/471984/>) and asked Greg Kroah-Hartman to remove the LTTng code from staging. Greg complied with that request, with the result that LTTng is, once again, no closer to merging into the mainline than it was before. This particular story line, it seems, has at least one more season to run yet. 

What is it about LTTng that makes it unsuitable for merging into the mainline? It starts with a lot of duplicated infrastructure. Inevitably, LTTng brings in its own ring buffer to communicate events to user space, despite the fact that the two ring buffers used by perf and ftrace are already seen as being too many. There is a new instrumentation mechanism for system calls - something that the kernel already has. And, of course, there is a new user-space ABI to control all of this - again an unwelcome addition when there is strong pressure from some directions to merge the existing in-kernel tracing ABIs. 

Duplicated infrastructure always tends to be hard to merge into the mainline; duplicated user-space ABIs, which must be supported forever, are even more so. It is thus not surprising that there is pushback against these patches, even without considering the highly contentious nature of the discussion around tracing work in general. Ingo claims to be receptive to merging the parts of LTTng that are better than what the kernel has now \- after it has been unified with the existing infrastructure, of course - but, he says, Mathieu has been more interested in maintaining LTTng as a separate "brand" and has been unwilling to merge things in this way. 

Mathieu's [response](<https://lwn.net/Articles/471992/>) has not done much to address those concerns. Duplicate infrastructure, he said, is fine as long as there is no agreement on how that infrastructure should work. Thus, he said, it is better to get his ring buffer into the mainline and to try to work out the differences there. He takes a similar approach to the ABI: 

Interfaces to user-space: very much like filesystems, these ABIs don't need to be shared across projects that have different use-cases. Having multiple tracer ABIs, if self-contained, should not hurt anybody and just increase the rate of innovation. 

The points that are missed here are that (1) filesystems do, in fact, share the same ABI, and (2) there is indeed a cost to multiple ABIs for tracing. Those ABIs have to be maintained indefinitely and they fragment the efforts of tool developers who find themselves forced to choose one or the other. Unless he can produce a convincing proof that the existing kernel interfaces cannot possibly be extended to meet LTTng's needs, Mathieu will almost certainly not succeed in getting a new tracing ABI into the mainline. 

Two notable conclusions were reached at the [2011 Kernel Summit](<https://lwn.net/Articles/KernelSummit2011/>). One was that maintainers should say "no" more often and accept fewer new features into the mainline; that would argue that Ingo and others are right to block the addition of LTTng in its current form. But the other conclusion was that code that has been shipped for years and that has real users should be strongly considered for merging even if it has known technical shortcomings. That, of course, would argue for merging LTTng, which certainly meets those conditions. Given the players involved, that conflict seems almost certain to be resolved with LTTng remaining an active project out of the mainline. Tune in next year for another episode of "As the tracing world turns."
