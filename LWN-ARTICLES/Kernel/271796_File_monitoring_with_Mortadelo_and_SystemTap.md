---
title: File monitoring with Mortadelo and SystemTap
url: https://lwn.net/Articles/271796/
date: "March 5, 2008"
category: SystemTap
author: "By Jake Edge March 5, 2008"
---

By **Jake Edge**  
March 5, 2008

[SystemTap](<http://sourceware.org/systemtap/>) is a tool to help gather information about running Linux systems which has been available for some time now. But applications that use the tool have been few and far between. [Mortadelo](<http://www.gnome.org/~federico/news-2007-06.html>) is a GUI tool that uses SystemTap to observe and record system calls. It is more of a proof-of-concept than a complete application--though it is useful in its current form--but it does start to show some of the things that can be done using SystemTap. 

Mortadelo specifically intercepts system calls that deal with accessing files, collecting the arguments to the calls as well the return codes. It is patterned after the Windows Filemon program, which is used in much the same way that a Linux user might use `strace`--only with a GUI. Problems with permissions or files that do not exist are the kinds of things that Mortadelo could be used to diagnose. 

[![\[Mortadelo\]](https://static.lwn.net/images/mortadelo_sm.png)](<http://lwn.net/Articles/271828/>)

The data collected is displayed in a list in the GUI (shown at left), which can then be filtered using regular expressions to pull out the information of interest. Because it uses SystemTap, Mortadelo gathers information from all running processes at once, allowing the user to choose which parts they are interested in. The filtering is somewhat primitive, in that particular fields cannot be chosen to filter on, but still useful because it searches each entry fully. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

System calls that return an error are highlighted in red making it easy to pick them out. By choosing appropriate strings to filter on, all permission errors in the system or every access of a particular filename can be seen. The GUI allows one to start and stop the recording as well as to save the captured data to a file. Each entry includes a timestamp, the process name and pid, the system call, return code, and arguments. 

The application is written in C#, using the Mono framework; one of the authors has an interesting weblog entry [comparing Mono and Python](<http://www.gnome.org/~federico/news-2007-06.html#18>) for developing this kind of tool. Mortadelo's interface to SystemTap is fairly straightforward, it spawns a `stap` command and sends it the probe points and code via `stdin`. It then reads the `stap` output, parsing it and displaying it in the window. 

There were some tricks to getting it to build and run, but Eugene Teo's [instructions for running it on Fedora 8](<http://www.kernel.sg/roller/eugene/entry/mortadelo_a_graphical_systemwide_strace>) were quite helpful. Part of the problem was in getting SystemTap going on the system, which is a [problem we have mentioned before](<http://lwn.net/Articles/244536/>). There were some other small hurdles as well, but Teo's hints and proper application of `grep` were enough to get past those. 

Mortadelo's impact isn't so much in the application itself as it is in some of the ideas behind it. Using SystemTap for GUI tools will help users and administrators, especially those who are not command-line savvy. If Mortadelo, or some descendant of it, becomes popular, that will help make SystemTap use more widespread. Distributors will start packaging it in more readily usable forms, perhaps installing it by default. That will in turn help anyone tasked with keeping a Linux system smoothly functioning, whether they are GUI-centric or not.
