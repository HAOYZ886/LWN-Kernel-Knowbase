---
title: "trace-cmd: A front-end for Ftrace"
url: https://lwn.net/Articles/410200/
date: "October 20, 2010"
category: "Development tools-Kernel tracing; Ftrace; Tracing-Ftrace"
author: "October 20, 2010 This article was contributed by Steven Rostedt"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

October 20, 2010

This article was contributed by Steven Rostedt

Previous LWN articles have explained the basic way to use Ftrace directly through the debugfs filesystem ([part 1](<http://lwn.net/Articles/365835/>) and [part 2](<http://lwn.net/Articles/366796/>)). While the debugfs interface is rather simple, it can also be awkward to work with. It is especially convenient, though, for embedded platforms where it may be difficult to build and install special user tools on the device. On the desktop, it may be more convenient to have a command-line tool that works with Ftrace instead of echoing various commands into strange files and reading the result from another file. This tool does exist, and it is called `trace-cmd`. 

`trace-cmd` is a user-space front-end command-line tool for Ftrace. You can download it from the git repository at [ `git://git.kernel.org/pub/scm/linux/kernel/git/rostedt/trace-cmd.git`](<http://git.kernel.org/?p=linux/kernel/git/rostedt/trace-cmd.git;a=summary>). Some distributions ship it as a package, and some that currently do not, will soon. There are full man pages included, which are installed with a `make install_doc`. This article will not go over the information that is already in the man pages, but instead will explain a little about how `trace-cmd` works and how to use it. 

#### How it works

A simple use case of `trace-cmd` is to record a trace and then report it. 

```
# trace-cmd record -e ext4 ls
        [...]
        # trace-cmd report
        version = 6
        CPU 1 is empty
        cpus=2
               trace-cmd-7374  [000]  1062.484227: ext4_request_inode:   \
    	   		   	  dev 253:2 dir 40801 mode 33188
               trace-cmd-7374  [000]  1062.484309: ext4_allocate_inode:  \
    	   		   	  dev 253:2 ino 10454 dir 40801 mode 33188
```

The above example enables the ext4 tracepoints for Ftrace, runs the `ls` command and records the Ftrace data into a file named `trace.dat`. The `report` command reads the `trace.dat` file and outputs the tracing data to standard output. Some metadata is also shown before the trace output is displayed: the version of the file, any empty CPU buffers, and the number of CPUs that were recorded. 

By default, the `record` and `report` options write and read to the `trace.dat` file. You can use the `-o` or `-i` options to pick a different file to write to or read from respectively, but this article will use the default name when referencing the data file created by `trace-cmd`. 

When recording a trace, `trace-cmd` will fork off a process for each CPU on the system. Each of these processes will open the file in debugfs that represents the CPU the process is dedicated to record from. The process recording CPU0 will open `/sys/kernel/debug/tracing/per_cpu/cpu0/trace_pipe_raw`, the process recording CPU1 will open a similar file in the `cpu1` directory, and so on. The `trace_pipe_raw` file is a mapping directly to the Ftrace internal buffer for each CPU. Each of these CPU processes will read these files using splice to record into a temporary file during the trace. At the end of the record, the main process will concatenate the temporary files into a single `trace.dat` file. 

There's no need to manually mount the debugfs filesystem before using the tool as `trace-cmd` will look to see if and where it is mounted. If debugfs is not mounted, it will automatically mount it at `/sys/kernel/debug`. 

#### Recording a trace

As noted above, `trace-cmd` forks off a process for each CPU dedicated to record from that CPU but, in order to prevent scheduling interference, the threads are not pinned to a CPU. Pinning the threads to the CPU being traced may result in better cache usage, so a future version of `trace-cmd` may add an option to do that. The Ftrace ring buffers are allocated one per CPU, and each thread will read from a particular CPU's ring buffer. It is important to mention this because these threads can show up in the trace. 

A common request is to have `trace-cmd` ignore events that are caused by `trace-cmd` itself. But it is not wise to ignore these events because they show where the tracer may have made an impact on what it is tracing. These events can be filtered out after the trace, but they are good to keep around in the `trace.dat` file in case some delay was caused by the trace itself, and the events may show that. 

As `trace-cmd` is a front end to Ftrace, the arguments of `record` reflect some of the features of Ftrace. The `-e` option enables an event. The argument following the `-e` can be an event name, event subsystem name, or the special name `all`. The `all` name will make `trace-cmd` enable all events that the system supports. If a subsystem name is specified, then all events under that subsystem will be enabled during the trace. For example, specifying `sched` will enable all the events within the `sched` subsystem. To enable a single event, the event name can be used by itself, or the `subsystem:event` format can be used. If the subsystem name is left off, then all events with the given name will be enabled. Currently this would not be an issue because, as of this writing, all events have unique names. If more than one event or subsystem is to be traced, then multiple `-e` options may be specified. 

Ftrace also has special plugin tracers that do not simply trace specific events. These tracers include the function, function graph, and latency tracers. Through the debugfs `tracing` directory, these plugins are enabled by echoing the type of tracer into the `current_tracer` file. With `trace-cmd record`, they are enabled with the `-p` option. Using the tracer plugin name as the argument for `-p` enables that plugin. You can still specify one or more events with a plugin, but you may only specify a single plugin, or no plugin at all. 

When the record is finished, `trace-cmd` examines the kernel buffers and outputs some statistics, which may be a little confusing. Here's an example: 

```
Kernel buffer statistics:
          Note: "entries" are the entries left in the kernel ring buffer and are not
                recorded in the trace data. They should all be zero.
    
        CPU: 0
        entries: 0
        overrun: 0
        commit overrun: 0
    
        CPU: 1
        [...]
```

As the output explains, the `entries` field is not the number of entries that were traced, but the number of entries left in the kernel buffer. If entries were dropped because `trace-cmd` could not read the buffer faster than it was being written to, and the writer overflowed the buffer, then either the `overrun` or `commit overrun` values would be something other than zero. The `overrun` value is the number of entries that were dropped due to the buffer filling up, and the writer deleting the older entries. 

The `commit overrun` is much less likely to occur. Writes to the buffer is a three step process. First the writer reserves space in the ring buffer, then it writes to it, then it commits the change. Writing to the buffer does not disable interrupts. If a write is preempted by an interrupt, and the interrupt performs a large amount of tracing where it fills the buffer up to the point of the space that was reserved for the write it preempted, then it must drop events because it cannot touch the reserved space until it is committed. These dropped events are the `commit overrun`. This is highly unlikely to happen unless you have a small buffer. 

#### Filtering while recording

As explained in "[Secrets of the Ftrace function tracer](<http://lwn.net/Articles/370423/>)", Ftrace allows you to filter what functions will be traced by the function tracer. Also, you can graph a single function, or a select set of functions, with the function graph tracer. These features are not lost when using `trace-cmd`. 

```
# trace-cmd record -p function -l 'sched_*' -n 'sched_slice'
```

(Note that the above does not specify a command to execute, so `trace-cmd` will record until `Ctrl^C` is hit.) 

The `-l` option is the same as echoing its argument into `set_ftrace_filter`, and the `-n` option is the same as echoing its argument into `set_ftrace_notrace`. You can have more than one `-l` or `-n` option on the command line. `trace-cmd` will simply write all the arguments into the appropriate file. Note, those options are only useful with the `function` and `function_graph` plugins. The `-g` option (not shown) will pass its argument into the `set_graph_function` file. 

Here is a nice trick to see how long interrupts take in the kernel: 

```
# trace-cmd record -p function_graph -l do_IRQ -e irq_handler_entry sleep 10
        # trace-cmd report
        version = 6
        cpus=2
                Xorg-4262  [001] 212767.154882: funcgraph_entry:                   |  do_IRQ() {
                Xorg-4262  [001] 212767.154887: irq_handler_entry:    irq=21 name=sata_nv
                Xorg-4262  [001] 212767.154952: funcgraph_exit:       + 71.706 us  |  }
                Xorg-4262  [001] 212767.156948: funcgraph_entry:                   |  do_IRQ() {
                Xorg-4262  [001] 212767.156952: irq_handler_entry:    irq=22 name=ehci_hcd:usb1
                Xorg-4262  [001] 212767.156955: irq_handler_entry:    irq=22 name=NVidia CK804
                Xorg-4262  [001] 212767.156985: funcgraph_exit:       + 37.795 us  |  }
```

The events can also be filtered. To know what fields can be used for filtering a specific event, look in the `format` file from `/sys/kernel/debug/tracing/events/<subsystem>/<event>/format`, or run 

```
# trace-cmd report --events | less
```

on a `trace.dat` file that was created by the local system. The `--events` argument will list the event formats of all events that were available in the system that created the tracing file: 

```
$ trace-cmd report --events
        [...]
        name: **kmalloc_node**
        ID: 338
        format:
            field:unsigned short common_type;       offset:0;       size:2; signed:0;
            field:unsigned char common_flags;       offset:2;       size:1; signed:0;
            field:unsigned char common_preempt_count;       offset:3;       size:1; signed:0;
            field:int common_pid;   offset:4;       size:4; signed:1;
            field:int common_lock_depth;    offset:8;       size:4; signed:1;
    
            field:unsigned long call_site;  offset:16;      size:8; signed:0;
            field:const void * ptr; offset:24;      size:8; signed:0;
            field:size_t **bytes_req** ; offset:32;      size:8; signed:0;
        [...]
```

Using the `kmalloc_node` event, we can filter on all requests that were greater than 1000 bytes: 

```
# trace-cmd record -e kmalloc_node -f 'bytes_req > 1000'
```

The `-f` option specifies a filter for the event (specified in a `-e` option) preceding it. 

#### Reading the trace

As the initial example showed, to read the trace simply run the `report` command. By default, it will read the `trace.dat` file, unless the `-i` option specifies a different file to read, or the input file may simply be specified as the last argument. 

```
$ trace-cmd report -i mytrace.dat
        $ trace-cmd report mytrace.dat
```

The above two examples give the same result. The report command is not a privileged operation and only requires read permission on the data file it is reading. 

```
$ trace-cmd report
        version = 6
        cpus=2
          trace-cmd-8412  [000] 13140.422056: sched_switch:         8412:120:S ==> 0:120: swapper
            <idle>-0     [000] 13140.422068: power_start:          type=1 state=2
            <idle>-0     [000] 13140.422174: irq_handler_entry:    irq=0 handler=timer
            <idle>-0     [000] 13140.422180: irq_handler_exit:     irq=0 return=handled
```

The output is similar to what you would see in `/sys/kernel/debug/tracing/trace`. 

Having the trace data in a file gives some advantages over reading from a debugfs file. We can now easily filter what events we want to see, or pick a specific CPU to output. 

You can do extensive filtering on events and what CPUs you want to focus on: 

```
$ trace-cmd report --cpu 0 -F 'sched_wakeup: success == 1'
        version = 6
        cpus=2
              ls-8414  [000] 13140.423106: sched_wakeup: 8414:?:? + 8412:120:? trace-cmd Success
              ls-8414  [000] 13140.424179: sched_wakeup: 8414:?:? + 1155:120:? kondemand/0 Success
              ls-8414  [000] 13140.426925: sched_wakeup: 8414:?:? + 704:120:? phy0 Success
              ls-8414  [000] 13140.431172: sched_wakeup: 8414:?:? + 9:120:? events/0 Success
          events/0-9   [000] 13140.431182: sched_wakeup: 9:?:? + 11734:120:? sshd Success
              ls-8414  [000] 13140.434173: sched_wakeup: 8414:?:? + 1155:120:? kondemand/0 Success
```

The `--cpu 0` limits the output to only show the events that occurred on CPU 0. The `-F` option limits the output further to only show `sched_wakeup` events that have its success field equal to 1\. For more information about the filtering, consult the `trace-cmd-report(1)` man page. 

#### Tracing over the network

There may be situations where you want to trace an embedded device or some machine with very little disk space. Perhaps another machine has lots of disk space and you want to record the trace to that machine or maybe you are tracing the filesystem itself and minimal interference to that code is needed. This is where tracing over the network comes in handy. 

To set up a trace server, simply run something like the following command: 

```
$ trace-cmd listen -p 12345 -D -d /images/tracing/ -l /images/tracing/logfile
```

The only required option in the above is the `-p` option, which tells `trace-cmd` what port to listen on. The `-D` puts `trace-cmd` into daemon mode, while the `-d /images/tracing/` tells `trace-cmd` to output the trace files from the connections it receives into the `/images/tracing/` directory. Obviously, any directory you have write permission for can be used here. The `-l /images/tracing/logfile` tells `trace-cmd` to not write messages to standard output, but to the `/images/tracing/logfile` file instead. The listen command is not privileged, and can be run by any user. 

On the embedded device (or whatever client is used), instead of specifying a output file in the `trace-cmd record` the `-N` option is used followed by `host:port` syntax. 

```
# trace-cmd record -N gandalf:12345 -e sched_switch -e sched_wakeup -e irq hackbench 50
```

Back on the host `gandalf` a file is made in the `/images/tracing/` directory of the format "`trace.<client-host>:<client-port>.dat`". 

```
$ ls /images/tracing/
        logfile  trace.frodo:35287.dat
    
        $ cat /images/tracing/logfile
        [29078]Connected with frodo:35287
        [29078]cpus=2
        [29078]pagesize=4096
    
        $ trace-cmd report /images/tracing/trace.frodo\:35287.dat
        version = 6
        cpus=2
            <...>-17215 [000] 19858.840695: sched_switch:      17215:120:S ==> 0:120: swapper
           <idle>-0     [000] 19858.840934: irq_handler_entry: irq=30 handler=iwl3945
           <idle>-0     [000] 19858.840959: irq_handler_exit:  irq=30 return=handled
           <idle>-0     [000] 19858.840960: softirq_entry:     softirq=6 action=TASKLET
           <idle>-0     [000] 19858.841005: softirq_exit:      softirq=6 action=TASKLET
          [...]
```

`trace-cmd` is versatile enough to handle heterogeneous systems. All the information needed to create and read the `trace.dat` file is passed from the client to the host. The host could be a 64-bit x86 and the client a 32-bit PowerPC and the above would not change. A big-endian machine can read a little-endian file and vice versa. The compatibility to read various system types is not limited to network tracing. If a trace is performed on a big-endian 32-bit system, the resulting file can still be read from a little-endian 64-bit system. 

#### For the lazy Ftrace user

If using the internal kernel Ftrace buffer is sufficient and there is no need to record the trace, `trace-cmd` can still be useful. Pretty much all of the `record` options can be used with the `trace-cmd start` command. `start` does not create a `trace.dat` file, but simply starts Ftrace. Similarly, the `stop` command is just a convenient way to do: 

```
$ echo 0 > /sys/kernel/debug/tracing/tracing_on
```

For example: 

```
# trace-cmd start -p function_graph -g ip_rcv
    
        # sleep 10
    
        # trace-cmd stop
    
        # cat /sys/kernel/debug/tracing/trace
        # tracer: function_graph
        #
        # CPU  DURATION                  FUNCTION CALLS
        # |     |   |                     |   |   |   |
         1)               |  ip_rcv() {
         1)               |    T.769() {
         1)               |      nf_hook_slow() {
         1)   0.497 us    |        add_preempt_count();
         1)               |        nf_iterate() {
         1)   0.458 us    |          ip_sabotage_in();
```

If there is a case where the trace needs to be converted into a `trace.dat` file, the `extract` command can be used. After the above trace was done: 

```
# trace-cmd extract -o kernel-buf.dat
    
        # trace-cmd report kernel-buf.dat
        version = 6
        cpus=2
           <idle>-0  [001] 214146.661193: funcgraph_entry:          |  ip_rcv() {
           <idle>-0  [001] 214146.661196: funcgraph_entry:          |    T.769() {
           <idle>-0  [001] 214146.661197: funcgraph_entry:          |      nf_hook_slow() {
           <idle>-0  [001] 214146.661197: funcgraph_entry: 0.497 us |        add_preempt_count();
           <idle>-0  [001] 214146.661198: funcgraph_entry:          |        nf_iterate() {
           <idle>-0  [001] 214146.661199: funcgraph_entry: 0.458 us |          ip_sabotage_in();
```

To disable all tracing, which will ensure that no overhead is left from using the function tracers or events, the `reset` command can be used. It will disable all of Ftrace and bring the system back to full performance. 

```
# trace-cmd reset
```

#### What's next?

This article explains some of the use cases for `trace-cmd`. There is still more that it can do but we just did not have the space to fit it all in. This article and the `trace-cmd` man pages should be enough to get you on your way to using `trace-cmd` in a productive manner. 

So what's next? As this article shows how `trace-cmd` is a front-end tool for Ftrace, the next article will present `kernelshark` which is a graphical front-end to `trace-cmd`. Stay tuned.
