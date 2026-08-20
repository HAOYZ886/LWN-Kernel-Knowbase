---
title: "Huge pages part 5: A deeper look at TLBs and costs"
url: https://lwn.net/Articles/379748/
date: "March 23, 2010"
category: "Huge pages; Memory management-Huge pages; Memory management-Translation lookaside buffer"
author: "March 23, 2010 This article was contributed by Mel Gorman"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

March 23, 2010

This article was contributed by Mel Gorman

[_Editor's note: this is the fifth and final installment in Mel Gorman's series on the use of huge pages in Linux. Parts[1](<http://lwn.net/Articles/374424/>), [2](<http://lwn.net/Articles/375096/>), [3](<http://lwn.net/Articles/376606/>) and [4](<http://lwn.net/Articles/378641/>) are available for those who have not read them yet. Many thanks to Mel for letting us run this series at LWN_.] 

This chapter is not necessary to understand how huge pages are used and performance benefits from huge pages are often easiest to measure using an application-specific benchmark. However, there are the rare cases where a deeper understanding of the TLB can be enlightening. In this chapter, a closer look is taken at TLBs and analysing performance from a huge page perspective. 

#### 1 TLB Size and Characteristics

First off, it can be useful to know what sort of TLB the system has. On X86 and X86-64, the tool `x86info` can be used to discover the TLB size. 

```
$ x86info -c
          ...
          TLB info
           Instruction TLB: 4K pages, 4-way associative, 128 entries.
           Instruction TLB: 4MB pages, fully associative, 2 entries
           Data TLB: 4K pages, 4-way associative, 128 entries.
           Data TLB: 4MB pages, 4-way associative, 8 entries
          ...
```

On the PPC64 architecture, there is no automatic means of determining the number of TLB slots. PPC64 uses multiple translation-related caches of which the TLB is at the lowest layer. It is safe to assume on older revisions of POWER - such as the PPC970 - that 1024 entries are available. POWER 5+ systems will have 2048 entries and POWER 6 does not use a TLB. On PPC64, the topmost translation layer uses an Effective to Real Address Translation (ERAT) cache. On POWER 6, it supports 4K and 64K entries but typically the default huge page size of 16MB consumes multiple ERAT entries. Hence, the article will focus more on the TLB than on ERAT. 

#### 2 Calculating TLB Translation Cost

When deciding whether huge pages will be of benefit, the first step is estimating how much time is being spent translating addresses. This will approximate the upper-boundary of performance gains that can be achieved using huge pages. This requires that the number of TLB misses that occurred is calculated as well as the average cost of a TLB miss. 

On much modern hardware, there is a Performance Measurement Unit (PMU) which provides a small number of hardware-based counters. The PMU is programmed to increment when a specific low-level event occurs and interrupt the CPU when a threshold, called the sample period, is reached. In many cases, there will be one low-level event that corresponds to a TLB miss so a reasonable estimate can be made of the number of TLB misses. 

On Linux, the PMU can be programmed with `oprofile` on almost any kernel currently in use, or with `perf` on recent kernels. Unfortunately, `perf` is not suitable for the analysis we need in this installment. `Perf` maps high-level requests, such as cache misses, to suitable low-level events. However it is not currently able to map certain TLB events, such as the number of cycles spent walking a page table. It is technically possible to specify a raw event ID to `perf`, but figuring out the raw ID is error-prone and tricky to verify. Hence, we will be using `oprofile` to program the PMU in this installment. 

A detailed examination of the hardware specification may yield an estimate for the cost of a TLB miss, but it is time-consuming and documentation is not always sufficient. Broadly speaking, there are three means of estimating the TLB cost in the absence of documentation. The simplest case is where the TLB is software-filled and the operating system is responsible for filling the TLB. Using a profiling tool, the number of times the TLB miss handler was called and the time spent can be recorded. This gives an average cost of the TLB miss but software-filled TLBs are not common in mainstream machines. The second method is to use an analysis program such as Calibrator [manegold04] that guesses characteristics of cache and the TLB. While there are other tools that exist that claim to be more accurate [yotov04a][yotov04b], Calibrator has the advantage of being still available for download and it works very well for X86 and X86-64 architectures. Its use is described below. 

Calibrator does not work well on PPC64 as the TLB is the lowest layer where as Calibrator measures the cost of an ERAT miss at the highest layer. On PPC64, there is a hardware counter that calculates the number of cycles spent doing page table walks. Hence, when automatic measurement fails, it may be possible to measure the TLB cost using the PMU as described in Section 2.3, below. 

Once the number of TLB misses and the average cost of a miss is known, the percentage time spent servicing TLB misses is easily calculated. 

#### 2.1 Estimating Number of TLB Misses

`Oprofile` can be used to estimate the number of TLB misses using the PMU. This article will not go in-depth on how PMUs and `oprofile` work but, broadly speaking, the PMU counts low-level events such as a TLB miss. To avoid excessive overhead, only a sample-period number of events are recorded. When the sample-period is reached, an interrupt is raised and `oprofile` records the details of that event. An estimate of the real number of TLB misses that occurred is then 

> EstimatedTLBMisses = TLBMissesSampled * SamplePeriod 

The output below shows an example `oprofile` session that sampled Data-TLB (DTLB) misses within a benchmark. 

```
$ opcontrol --setup --event PM_CYC_GRP22:50000 --event PM_DTLB_MISS_GRP22:1000
                  --vmlinux=/vmlinux
      $ opcontrol --start
      Using 2.6+ OProfile kernel interface.
      Reading module info.
      Using log file /var/lib/oprofile/samples/oprofiled.log
      Daemon started.
      Profiler running.
      $ ./benchmark
      $ opcontrol --stop
      $ opcontrol --dump
      $ opreport
      CPU: ppc64 970MP, speed 2500 MHz (estimated)
      Counted PM_CYC_GRP22 events ((Group 22 pm_pe_bench4) Processor cycles)
              with a unit mask of 0x00 (No unit mask) count 50000
      Counted PM_DTLB_MISS_GRP22 events ((Group 22 pm_pe_bench4) Data TLB misses)
              with a unit mask of 0x00 (No unit mask) count 1000
      PM_CYC_GRP22:5...|PM_DTLB_MISS_G...|
        samples|      %|  samples|      %|
      ------------------------------------
         622512 98.4696      9651 97.8506 benchmark
           4170  0.6596        11  0.1115 libc-2.9.so
           3074  0.4862         1  0.0101 oprofiled
            840  0.1329         4  0.0406 bash
            731  0.1156       181  1.8351 vmlinux-2.6.31-rc5
            572  0.0905        14  0.1419 ld-2.9.so
```

Note in the figure that 9651 samples were taken and the sample period was 1000. Therefore it is reasonable to assume, using the equation above, that the benchmark incurred 9,651,000 DTLB misses. Analysis of a more complex benchmark would also include misses incurred by libraries. 

#### 2.2 Estimating TLB Miss Cost using Calibrator

Calibrator should be used on machines where the TLB is the primary cache for translating virtual to physical addresses. This is the case for X86 and X86-64 machines but not for PPC64 where there are additional translation layers. The first step is to setup a working directory and obtain the calibrator tool. 

```
$ wget http://homepages.cwi.nl/~manegold/Calibrator/v0.9e/calibrator.c
      $ gcc calibrator.c -lm -o calibrator
      calibrator.c:131: warning: conflicting types for built-in function 'round'
```

The warning is harmless. Note the lack of compiler optimisation options specified which is important so as not to skew the results reported by the tool. Running Calibrator with no parameters gives: 

```
$ ./calibrator 
      Calibrator v0.9e
      (by Stefan.Manegold@cwi.nl, http://www.cwi.nl/ manegold/)
    
      ! usage: './calibrator <MHz> <size>[k|M|G] <filename>` !
```

The CPU `MHz` parameter is used to estimate the time in nanoseconds a TLB miss costs. The information is not automatically retrieved from `/proc/` as the tool was intended to be usable on Windows, but [this shell script](<https://lwn.net/Articles/379764/>) should discover the MHz value on many Linux installations. `size` is the size of work array to allocate. It must be sufficiently large that the cache and TLB reach are both exceeded to have any chance of accuracy but in practice much higher values were required. The poorly named parameter `filename` is the prefix given to the output graphs and gnuplot files. 

[This page](<https://lwn.net/Articles/379767/>) contains a wrapper script around Calibrator that outputs the approximate cost of a TLB miss as well as how many TLB misses must occur to consume a second of system time. An example running the script on an Intel Core Duo T2600 is as follows: 

```
$ ./run-calibrator.sh
      Running calibrator with size 13631488: 19 cycles 8.80 ns 
      Running calibrator with size 17563648: 19 cycles 8.80 ns matched 1 times
      Running calibrator with size 21495808: 19 cycles 8.80 ns matched 2 times
      Running calibrator with size 25427968: 19 cycles 8.80 ns matched 3 times
    
      TLB_MISS_LATENCY_TIME=8.80
      TLB_MISS_LATENCY_CYCLES=19
      TLB_MISSES_COST_ONE_SECOND=114052631
```

In this specific example, the estimated cost of a TLB miss is 19 clock cycles or 8.80ns. It is interesting to note that the cost of an L2 cache miss on the target machine is 210 cycles, making it likely that the hardware is hiding most of the latency cost using pre-fetching or a related technique. Compare the output with the following from an older generation machine based on the AMD Athlon 64 3000+, which has a two-level TLB structure: 

```
$ ./run-calibrator.sh 
      Running calibrator with size 13631488: 16 cycles 8.18 ns 
      Running calibrator with size 17563648: 19 cycles 9.62 ns 
      Running calibrator with size 21495808: 19 cycles 9.54 ns matched 1 times
      Running calibrator with size 25427968: 19 cycles 9.57 ns matched 2 times
      Running calibrator with size 29360128: 34 cycles 16.96 ns 
      Running calibrator with size 33292288: 34 cycles 16.99 ns matched 1 times
      Running calibrator with size 37224448: 37 cycles 18.17 ns 
      Running calibrator with size 41156608: 37 cycles 18.17 ns matched 1 times
      Running calibrator with size 45088768: 36 cycles 18.16 ns matched 2 times
      Running calibrator with size 49020928: 37 cycles 18.17 ns matched 3 times
    
      TLB_MISS_LATENCY_TIME=18.17
      TLB_MISS_LATENCY_CYCLES=37
      TLB_MISSES_COST_ONE_SECOND=54297297
```

While calibrator will give a reasonable estimate of the cost, some manual adjustment may be required based on observation. 

#### 2.3 Estimating TLB Miss Cost using Hardware

When the TLB is not the topmost translation layer, Calibrator is not suitable to measure the cost of a TLB miss. In the specific case of PPC64, Calibrator measures the cost of an ERAT miss but the ERAT does not always support all the huge page sizes. In the event a TLB exists on POWER, it is the lowest level of translation and it supports huge pages. Due to this, measuring the cost of a TLB miss requires help from the PMU. 

Two counters are minimally required - one to measure the number of TLB misses and a second to measure the number of cycles spent walking page tables. The exact name of the counters will vary but for the PPC970MP, the `PM_DTLB_MISS_GRP22` counter for TLB misses and `PM_DATA_TABLEWALK_CYC_GRP30` counters are suitable. 

To use the PMU, a consistent test workload is required that generates a relatively fixed number of TLB misses per run. The simplest workload to use in this case is [STREAM](<http://www.cs.virginia.edu/stream/>). First, download and build stream: 

```
$ wget http://www.cs.virginia.edu/stream/FTP/Code/stream.c
      $ gcc -O3 -DN=44739240 stream.c -o stream
```

The value of `N` is set such that the total working set of the benchmark will be approximately 1GB. 

Ideally, the number of DTLB misses and cycles spent walking page tables would be measured at the same time but due to limitations of the PPC970MP, they must be measured in two separate runs. Because of this, it is _very_ important that the cycles be sampled at the same time and it is _essential_ that the samples taken for cycles in each of the two runs are approximately the same. This will require you to scale the sample rate for the DTLB and page table walk events appropriately. Here are two `oprofile` reports based on running STREAM. 

```
CPU: ppc64 970MP, speed 2500 MHz (estimated)
      Counted PM_CYC_GRP30 events ((Group 30 pm_isource) Processor cycles)
              with a unit mask of 0x00 (No unit mask) count 50000
      Counted PM_DATA_TABLEWALK_CYC_GRP30 events ((Group 30 pm_isource) Cycles
    	  doing data tablewalks) with a unit mask of 0x00 (No unit mask)
    	  count 10000
      PM_CYC_GRP30:5...|PM_DATA_TABLEW...|
        samples|      %|  samples|      %|
      ------------------------------------
         604695 97.9322    543702 99.3609 stream
    
      CPU: ppc64 970MP, speed 2500 MHz (estimated)
      Counted PM_CYC_GRP23 events ((Group 23 pm_hpmcount1) Processor cycles)
              with a unit mask of 0x00 (No unit mask) count 50000
      Counted PM_DTLB_MISS_GRP23 events ((Group 23 pm_hpmcount1) Data TLB mis
              with a unit mask of 0x00 (No unit mask) count 1000
      PM_CYC_GRP23:5...|PM_DTLB_MISS_G...|
        samples|      %|  samples|      %|
      ------------------------------------
         621541 98.5566      9644 98.0879 stream
```

The first point to note is that the samples taken for `PM_CYC_GRP` are approximately the same. This required that the sample period for `PM_DATA_TABLEWALK_CYC_GRP30` be 10000 instead of the minimum allowed of 1000. The average cost of a DTLB miss is now trivial to estimate. 

```
PageTableCycles = CyclesSampled * SamplePeriod 
        		    = 543702 * 10000
    
        TLBMisses = TLBMissSampled * SamplePeriod 
        	      = 9644 * 1000
    
        TLBMissCost = PageTableWalkCycles/TLBMisses 
                    = 5437020000/9644000 
    		= ~563 cycles
```

Here the TLB-miss cost on PPC64 is observed to be much higher than on comparable X86 hardware. However, take into account that the ERAT translation cache hides most of the cost translating addresses and it's miss cost is comparable. This is similar in principal to having two levels of TLB. 

#### 2.4 Estimating Percentage Time Translating

Once the TLB miss cost estimate is available, estimates for any workload depend on a profile showing cycles spent within the application and the DTLB samples such as the following report. 

```
CPU: ppc64 970MP, speed 2500 MHz (estimated)
      Counted PM_CYC_GRP22 events ((Group 22 pm_pe_bench4) Processor cycles)
              with a unit mask of 0x00 (No unit mask) count 50000
      Counted PM_DTLB_MISS_GRP22 events ((Group 22 pm_pe_bench4) Data TLB misses)
              with a unit mask of 0x00 (No unit mask) count 1000
      PM_CYC_GRP22:5...|PM_DTLB_MISS_G...|
        samples|      %|  samples|      %|
      ------------------------------------
         156295 95.7408      2425 96.4215 stream
```

The calculation of the percentage of time spent servicing TLB misses is then as follows 

```
CyclesExecuted = CyclesSamples * SampleRateOfCycles
         		   = 156292 * 50000 
    		   = 7814600000 cycles
    
        TLBMissCycles = TLBMissSamples * SampleRateOfTLBMiss * TLBMissCost
         		  = 2425 * 1000 * 563 
        		  = 1365275000
    
        PercentageTimeTLBMiss = (TLBMissCycles * 100)/CyclesExecuted 
        			  = 17.57%
```

Hence, the best possible performance gain we might expect from using huge pages with this workload is about 17.57%. 

#### 2.5 Verifying Accuracy

Once a TLB miss cost has been estimated, it should be validated. The easiest means of doing this is with the STREAM benchmark, modified using [this patch](<https://lwn.net/Articles/379769/>) to use `malloc()` and rebuilt. The system must be then minimally configured to use hugepages with the benchmark. The huge page size on PPC64 is 16MB so the following commands will configure the system adequately for the validation. Note that the hugepage pool allocation here represents roughly 1GB of huge pages for the STREAM benchmark. 

```
$ hugeadm --create-global-mounts
        $ hugeadm --pool-pages-min 16M:1040M
        $ hugeadm --pool-list
            Size  Minimum  Current  Maximum  Default
        16777216       65       65       65        *
```

We then run STREAM with base pages and profiling to make a prediction on what the hugepage overhead will be. 

```
$ oprofile_start.sh --sample-cycle-factor 5 --event timer --event dtlb_miss
      [ ... profiler starts ... ]
      $ /usr/bin/time ./stream
      [ ...]
      Function      Rate (MB/s)   Avg time     Min time     Max time
      Copy:        2783.1461       0.2585       0.2572       0.2594
      Scale:       2841.6449       0.2530       0.2519       0.2544
      Add:         3080.5153       0.3499       0.3486       0.3511
      Triad:       3077.4167       0.3498       0.3489       0.3510
      12.10user 1.36system 0:13.69elapsed 98%CPU (0avgtext+0avgdata 0maxresident)k
      0inputs+0outputs (0major+262325minor)pagefaults 0swaps
    
      $ opcontrol --stop
      $ opreport
      CPU: ppc64 970MP, speed 2500 MHz (estimated)
      Counted PM_CYC_GRP23 events ((Group 23 pm_hpmcount1) Processor cycles)
              with a unit mask of 0x00 (No unit mask) count 50000
      Counted PM_DTLB_MISS_GRP23 events ((Group 23 pm_hpmcount1) Data TLB misses)
              with a unit mask of 0x00 (No unit mask) count 1000
      PM_CYC_GRP23:5...|PM_DTLB_MISS_G...|
        samples|      %|  samples|      %|
      ------------------------------------
         599073 98.2975      9492 97.1844 stream
```

Using the methods described earlier, it is predicted that 17.84% of time is spent translating addresses. Note that `time` reported that the benchmark took 13.69 seconds to complete. Now rerun the benchmark using huge pages. 

```
$ oprofile_start.sh --sample-cycle-factor 5 --event timer --event dtlb_miss
      [ ... profiler starts ... ]
      $ hugectl --heap /usr/bin/time ./stream
      [ ...]
      Function      Rate (MB/s)   Avg time     Min time     Max time
      Copy:        3127.4279       0.2295       0.2289       0.2308
      Scale:       3116.6594       0.2303       0.2297       0.2317
      Add:         3596.7276       0.2988       0.2985       0.2992
      Triad:       3604.6241       0.2982       0.2979       0.2985
      10.92user 0.82system 0:11.95elapsed 98%CPU (0avgtext+0avgdata 0maxresident)k
      0inputs+0outputs (0major+295minor)pagefaults 0swaps
    
      $ opcontrol --stop
      $ opreport
      CPU: ppc64 970MP, speed 2500 MHz (estimated)
      Counted PM_CYC_GRP23 events ((Group 23 pm_hpmcount1) Processor cycles)
              with a unit mask of 0x00 (No unit mask) count 50000
      Counted PM_DTLB_MISS_GRP23 events ((Group 23 pm_hpmcount1) Data TLB misses)
              with a unit mask of 0x00 (No unit mask) count 1000
      PM_CYC_GRP23:5...|PM_DTLB_MISS_G...|
        samples|      %|  samples|      %|
      ------------------------------------
         538776 98.4168         0       0 stream
```

DTLB misses are not negligible within the STREAM benchmark and it now completes in 11.95 seconds instead of 13.69, which is about 12% faster. Of the four operations, Copy is now 12.37% faster, Scale is 9.67% faster, Add is 16.75% faster and Triad is 17.13% faster. Hence, the estimate of 563 cycles for DTLB misses on this machine is reasonable. 

#### 3 Calculating TLB Miss Cost with libhugetlbfs

The methods described in this section for measuring TLB costs were incorporated into `libhugetlbfs` as of release `2.7` in a script called `tlbmiss_cost.sh` and a manual page is included. It automatically detects whether `calibrator` or `oprofile` should be used to measure the cost of a TLB miss and optionally will download the necessary additional programs to use for the measurement. By default, it runs silently but in the following example where a miss cost of 19 cycles was measured, verbose output is enabled to show details of it working. 

```
$ tlbmiss_cost.sh -v
        TRACE: Beginning TLB measurement using calibrator
        TRACE: Measured CPU Speed: 2167 MHz
        TRACE: Starting Working Set Size (WSS): 13631488 bytes
        TRACE: Required tolerance for match: 3 cycles
        TRACE: Measured TLB Latency 19 cycles within tolerance. Matched 1/3
        TRACE: Measured TLB Latency 19 cycles within tolerance. Matched 2/3
        TRACE: Measured TLB Latency 19 cycles within tolerance. Matched 3/3
        TLB_MISS_COST=19
```

#### 4 Summary

While a deep understanding of the TLB and oprofile is not necessary to take advantage of huge pages, it can be instructive to know more about the TLB and the expected performance benefits before any modifications are made to a system configuration. Using oprofile, reasonably accurate predictions can be made in advance. 

#### Conclusion

While virtual memory is an unparalleled success in engineering terms, it is not totally free. Despite multiple page sizes being available for over a decade, support within Linux was historically tricky to use and avoided by even skilled system administrators. Over the last number of years, effort within the community has brought huge pages to the point where they are relatively painless to configure and use with applications, even to the point of requiring no source level modifications to the applications. Using modern tools, it was shown that performance can be improved with minimal effort and a high degree of reliability. 

In the future, there will still be a push for greater transparent support of huge pages, particularly for use with KVM. Patches are currently being developed by Andrea Arcangeli aiming at the goal of greater transparency. This represents a promising ideal but there is little excuse for avoiding huge page usage as they exist today. 

Happy Benchmarking. 

#### Bibliography

libhtlb09
     Various Authors. _libhugetlbfs 2.8 HOWTO_. Packaged with the libhugetlbfs source. <http://sourceforge.net/projects/libhugetlbfs>, 2009\. 

casep78
     Richard P. Case and Andris Padegs. Architecture of the IBM system/370. _Commun. ACM_ , 21(1):73--96, 1978. 

denning71
     Peter J. Denning. On modeling program behavior. In _AFIPS '71 (Fall): Proceedings of the November 16-18, 1971, fall joint computer conference_ , pages 937--944, New York, NY, USA, 1971. ACM. 

denning96
     Peter J. Denning. Virtual memory. _ACM Comput. Surv._ , 28(1):213--216, 1996. 

gorman09a
     Mel Gorman. <http://www.itwire.com/content/view/30575/1090/1/0>. <http://www.csn.ul.ie/~mel/docs/stream-api/>, 2009\. 

henessny90
     Henessny, J. L. and Patterson, D. A. _Computer Architecture a Quantitative Approach_. Morgan Kaufmann Publishers, 1990. 

manegold04
     Stefan Manegold and Peter Boncz. _The Calibrator (v0.9e), a Cache-Memory and TLB Calibration Tool_. <http://homepages.cwi.nl/~manegold/Calibrator/calibrator.shtml>, 2004\. 

mccalpin07
     John D. McCalpin. _STREAM: Sustainable Memory Bandwidth in High Performance Computers_. In a continually updated technical report. <http://www.cs.virginia.edu/stream/>, 2007. 

smith82
     Smith, A. J. Cache memories. _ACM Computing Surveys_ , 14(3):473--530, 1982. 

yotov04a
     Kamen Yotov, Keshav Pingali, and Paul Stodghill. Automatic measurement of memory hierarchy parameters. Technical report, Cornell University, nov 2004. 

yotov04b
     Kamen Yotov, Keshav Pingali, and Paul Stodghill. X-ray : Automatic measurement of hardware parameters. Technical report, Cornell University, oct 2004.
