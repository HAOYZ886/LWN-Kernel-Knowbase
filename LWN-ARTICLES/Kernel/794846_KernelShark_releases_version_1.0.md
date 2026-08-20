---
title: KernelShark releases version 1.0
url: https://lwn.net/Articles/794846/
date: "July 31, 2019"
category: "Development tools-Kernel tracing; Ftrace; Tracing"
author: "By Jake Edge July 31, 2019"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jake Edge**  
July 31, 2019

It has been the better part of a decade since the last [KernelShark article](<https://lwn.net/Articles/425583/>) appeared here; in the interim, the kernel-tracing visualization tool has undergone some major changes. While the high-level appearance is largely similar, the underlying code has switched from GTK+ 2.0 to Qt 5\. On July 26, maintainer Steven Rostedt [announced](<https://lwn.net/ml/linux-kernel/20190726095730.0674d81d@gandalf.local.home/>) the release of KernelShark version 1.0, which makes it a good time to take another peek. 

[KernelShark](<http://www.kernelshark.org/>) is a graphical interface to help track down information in the voluminous kernel traces that [`trace-cmd`](<http://man7.org/linux/man-pages/man1/trace-cmd.1.html>) can produce. `trace-cmd` is a front end for the [ftrace kernel tracer](<https://www.kernel.org/doc/Documentation/trace/ftrace.txt>). Rostedt wrote about [`trace-cmd`](<https://lwn.net/Articles/410200/>) and ftrace ([part 1](<https://lwn.net/Articles/365835/>) and [part 2](<https://lwn.net/Articles/366796/>)) for LWN nearly a decade ago as well. Ftrace can collect an enormous amount of information from within a running kernel; `trace-cmd` simply makes it much easier for users to configure and manage those traces. KernelShark adds yet another level of capabilities. 

> [ ![\[KernelShark\]](https://static.lwn.net/images/2019/kshark1-sm.png) ](<https://lwn.net/Articles/794932/>)

As can be seen in the screen shot from a lightly loaded system (click through for a full-resolution view), KernelShark maintains its overall look, with two main panes to display the data it found in `trace.dat` (by default); `trace.dat` is the name of the default `trace-cmd` output file. The top gives a graphical view of the trace events that were gathered, organized by CPU, at least by default. Each horizontal bar indicates the activity on a particular CPU; that activity is color-coded based on the task running. Full-height indicators on the bar denote an event captured, while half-height indicators just show that the task is running; a bare line shows idle time. 

The graphical display can show task bars instead of, or in addition to, the CPU bars. The task bars show the events only for tasks of interest, which are chosen from the "Tasks" entry in the "Plots" menu. The display shows a segment of the time covered by the trace; initially, it shows the entire duration, with start and end times at either end. But one can zoom in on the trace, reducing the amount of time shown and increasing the granularity of the view (the screen shot below is zoomed in from the one above). That will allow users to find and focus on places of interest in the trace. 

Zooming can be accomplished a number of different ways. There are buttons directly above the graphical pane; "+" and "-" increase and decrease the zoom factor, while "++" and "--" zoom all the way in or out. The mouse scroll wheel can also be used to zoom in and out. One note: if there are more horizontal bars than the graphical pane can show, so that there is a scroll bar for the pane, it can be a little unexpected to get the zoom function rather than the expected scroll behavior. Scrolling requires using the scroll bar (or increasing the size of the pane or KernelShark window to eliminate the need). Yet another mechanism to zoom in is to click and drag within the graphical view, which outlines a rectangle that will define the region of interest; when the mouse is released it zooms in to show that region. 

The second main pane is the list pane, which shows the events that were recorded in the trace. For each event, there is an index number for the event's position in the trace, the CPU on which it was recorded, a timestamp, the name of the task and its process ID, some flags (interrupts disable, need reschedule, etc.), the event name, and the output from the event. Above the list pane is a search interface that can be used to search any of the columns, as well as a "Graph follows" checkbox, which governs the action in the graphical pane when a particular list entry is selected. 

> [ ![\[KernelShark zoomed in\]](https://static.lwn.net/images/2019/kshark2-sm.png) ](<https://lwn.net/Articles/794933/>)

If "Graph follows" is checked (which is the default), then a marker (vertical line) is placed in the graphical view where an event that was selected in the list view occurs. Users can see what else is going on around that time at various timescales by zooming in and panning (using the arrow keys). There are two markers available (A and B); they can be set from either the list view or the graphical view. Double-clicking in the graphical view will set the marker (whichever has its button highlighted above the view), but it is being set on an "nearby" event, so the double-click must be close to an event or the marker will not be placed anywhere. Placing the marker will highlight the event in question in the list view as well. 

Each marker has a timestamp associated with it (i.e. the same as the timestamp on the event it refers to) that is listed with the buttons above the graphical view. If both markers are set, the difference between their timestamps is also displayed there to 100ns precision. 

There are multiple filtering capabilities available from the "Filter" menu. Tasks, events, and CPUs can be removed (or added back in) from the graphical view, list view, or both. The advanced filtering can do even more, allowing access to the filtering mechanisms provided by `trace-cmd`/ftrace. Filters can also be saved in order to reuse them later. 

KernelShark saves its current state when it exits and restores the state of that session when it is restarted. Beyond that, sessions can be saved at any time and then, naturally, can be loaded when they are needed. 

The [KernelShark documentation](<http://kernelshark.org/Documentation.html>) is rather terse, but covers most everything needed. The "Tools" menu has some entries that could use some explanation (e.g. on plugins, the seemingly non-functional color-scheme slider, and the "Record" option), however. The short [build instructions](<http://www.kernelshark.org/build.html>) provided everything needed to build KernelShark on a Fedora system; there is dependency information for Ubuntu as well. 

KernelShark currently lives in the [`trace-cmd` repository](<https://git.kernel.org/pub/scm/linux/kernel/git/rostedt/trace-cmd.git>), though that will change eventually. In the announcement, Rostedt noted that he will be transitioning the maintainership of KernelShark to Yordan Karadzhov; they will be sharing the maintainer duties for a while. As part of that transition, KernelShark will be moving to its own repository. 

For those digging into kernel traces, KernelShark looks to be an excellent tool to help find whatever problem they are investigating. It would seem that the KernelShark team is not resting on its laurels, either, as Rostedt specifically mentions getting started on work toward KernelShark 2.0. No roadmap seems to be available for that, but one would guess that user suggestions (and, better still, code) would be welcome at this point.
