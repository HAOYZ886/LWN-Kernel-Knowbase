---
title: Automated kernel testing
url: https://lwn.net/Articles/139123/
date: "June 8, 2005"
category: "Benchmarking; Development tools-Testing; Regression testing"
author: 
---

If there is one thing that almost all kernel developers agree with, it's that more testing is a good thing - especially if the results are presented in a useful way. Martin Bligh thus got a warm reception when he [announced](<https://lwn.net/Articles/138460/>) a new kernel testing facility. As he put it: 

Currently it builds and boots any mainline, -mjb, -mm kernel within about 15 minutes of release. runs dbench, tbench, kernbench, reaim and fsx. Currently I'm using a 4x AMD64 box, a 16x NUMA-Q, 4x NUMA-Q, 32x x440 (ia32) PPC64 Power 5 LPAR, PPC64 Power 4 LPAR, and PPC64 Power 4 bare metal system. 

This is, indeed, a fairly wide range of coverage. The [results](<http://ftp.kernel.org/pub/linux/kernel/people/mbligh/abat/regression_matrix.html>) are presented as a simple table, showing which kernels passed the tests and which did not. When a kernel fails a test, the relevant information is provided (though, often, that information is simply "did not boot," which is not entirely helpful). 

[![\[benchmark results\]](https://static.lwn.net/images/ns/kernel/kernbench.moe.png)](<http://ftp.kernel.org/pub/linux/kernel/people/mbligh/abat/perf/kernbench.moe.png>) These results have been augmented with [benchmark results](<http://ftp.kernel.org/pub/linux/kernel/people/mbligh/abat/perf/perf_matrix.html>), presented in a handy graphic form. The graph shown on the right, for example, notes that kernbench performance improved significantly around 2.6.6, and has held steady since 2.6.10. The -mm trees, however, perform notably worse than the mainline, and the difference between the two has been growing. The results have already led to some investigation into what is going on; the current suspect is the (36!) scheduler patches currently living in -mm. 

Numerous others have worked at testing and benchmarking kernel releases. Martin's work, however, has the advantages of being automated and presenting the results in a reasonable way. With these attributes, this project stands a good chance of helping the developers to produce better kernels in the near future.
