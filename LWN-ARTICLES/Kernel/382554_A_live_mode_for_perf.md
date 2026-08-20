---
title: "A \"live mode\" for perf"
url: https://lwn.net/Articles/382554/
date: "April 7, 2010"
category: Performance monitoring
author: "By Jake Edge April 7, 2010"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
April 7, 2010

The perf tracing tool has evolved quickly. When last we looked, Tom Zanussi had [added Python and Perl scripting to perf](<http://lwn.net/Articles/373842/>). Next up would seem to be [perf "live mode"](<http://lwn.net/Articles/381634/>), where perf no longer requires two steps: record the data, then analyze. Live mode will allow `perf trace record` and `perf trace report` to operate via a pipe, which allows instantaneous, as well as continuously updating (_a la_ `top`), output. 

So that no existing perf users need to change their scripts, Zanussi only added the new capabilities when perf recognizes that its `record` output is going to stdout or `report` input is coming from stdin. In that case, perf handles the data through a pipe, and uses special synthesized events to provide header information. This will also allow perf to operate over the network by piping its `record` output to netcat, and then reading it via netcat on another system and piping it into `report`. 

All of the scripts that are installed in the standard perf location (i.e. those which are listed in `perf trace -l`) are automatically able to be run in live mode: 

```
$ perf trace syscall-counts
```

will run both ends of the the syscall-counts script with a pipe in between, a more usable shorthand for: 

```
$ perf trace record syscall-counts -o - | perf trace report syscall-counts -i -
```

which itself is shorthand for: 

```
perf record -c 1 -f -a -M -R -e raw_syscalls:sys_enter -o - | \
      perf trace -i - -s ~/libexec/perf-core/scripts/python/syscall-counts.py
```

Zanussi also included several sample `top`-style scripts that can be used to monitor read/write or system call activity updated every three seconds. It looks to be a very useful addition to perf, which is rapidly becoming the "swiss army knife" of kernel monitoring.
