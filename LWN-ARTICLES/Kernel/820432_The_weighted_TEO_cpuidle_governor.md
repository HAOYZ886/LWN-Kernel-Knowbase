---
title: The weighted TEO cpuidle governor
url: https://lwn.net/Articles/820432/
date: "May 14, 2020"
category: "Power management-cpuidle"
author: "By Jonathan Corbet May 14, 2020 OSPM"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
May 14, 2020

* * *

[OSPM](<https://lwn.net/Articles/820337/>)

Life gets complicated for the kernel when there is nothing for the system to do. The obvious response is to put the CPU into an idle state to save power, but which one? CPUs offer a wide range of sleep states with different power-usage and latency characteristics. Picking too shallow a state will waste energy, while going too deep hurts latency and can impact the performance of the system as a whole. The [timer-events-oriented (TEO) cpuidle governor](<https://lwn.net/Articles/775618/>) is a relatively new attempt to improve the kernel's choice of sleep states; at the 2020 [Power Management and Scheduling in the Linux Kernel Summit](<http://retis.sssup.it/ospm-summit/>), Pratik Sampat presented [a variant of the TEO governor](<https://lwn.net/ml/linux-kernel/20200511141055.43029-2-psampat@linux.ibm.com/>) that tries to improve its choices further. 

Sampat started with a bit of background. The TEO governor is based on the idea that timer events are the most likely way that a system will wake up; they also happen to be the most deterministic, since they are known before the system goes idle. But CPUs are subject to wakeups from other sources — interrupts in particular — and that complicates the situation. So the TEO governor maintains a short history of actual idle times that is used to come up with a (hopefully) better guess for what the next idle period will really be. 

This history is an eight-entry circular buffer that indicates the recent pattern of non-timer wakeups. When the time comes to pick an idle state, the TEO governor looks at how many of those wakeups led to a sleep time that was less than expected; if the answer is "a majority of them", then the average observed sleep time is used to select an idle state that is shallower than would have otherwise been chosen. It works well, he said, but maybe it can be made better? 

He started by testing the idea of whether more history would improve the situation. Increasing the size of the idle-times buffer to 128 did not really help, though. With a set of benchmark results, Sampat showed performance numbers that were sometimes better and sometimes worse; latency often improved, but power consumption got much worse. More history led to the selection of shallower sleep states more often, in other words. 

It turns out, he said, that an average is not a good model of the distribution of sleep times, and a longer history may not reflect what is going to happen in the future. So he concluded that what is needed is to store and manage the history differently. The cpuidle governor would benefit from a way to answer a specific question: if the kernel is about to pick a given sleep state, what are the chances that the actual sleep time will better match a sleep that is one level shallower? 

#### The weighted governor

The result was the weighted-history TEO governor, which replaces the history buffer with an NxN matrix, where N is the number of sleep states supported by the processor. The rows correspond to the sleep state the TEO governor would pick in any given situation; each column along that row indicates the probability that the corresponding state should actually be chosen. If the system in question had three sleep states ("shallow", "medium", and "deep"), the matrix would be initialized to look like this: 

> | Shallow| Medium| Deep  
> ---|---|---|---  
> Shallow| 70% | 15% | 15%  
> Medium| 15% | 70% | 15%  
> Deep| 15% | 15% | 70%  
  
In other words, the matrix is set up so that the chances of each state selection being correct are 70%, with the remaining 30% spread across all the other states. Giovanni Gherdovich asked whether this initial distribution was hard-coded; the answer was "yes for now", and that the values have been chosen from a set of experiments Sampat ran. 

After each wakeup, the actual behavior is measured and the probabilities are tweaked accordingly. The actual amount of adjustment that should be performed is still unclear, he said; more experiments and testing are needed. 

When it comes time to make a prediction, the governor uses a biased random-number generator to pick a state; the biasing is done so that the chances of picking any particular state are the same as the observed probability that said state is the correct one. Why do that rather than just pick the highest-probability state? Often it turns out that the probabilities are fairly close, so a subset of the available states are all about as likely to be correct. The system will self-correct when the random-number generator steers it wrong. 

#### Results

A number of benchmark results followed, showing variable results. With schedbench, latency was better some times and worse others, but power consumption was always less. The accuracy of the sleep-state choices was similar to the unweighted TEO governor for a small number of threads, but improved for larger numbers of threads. Rafael Wysocki, the author of the TEO governor, said that he was surprised to see TEO doing as well as it does; he deliberately chose a simple algorithm to minimize the overhead involved. 

Sampat modified the ebizzy benchmark to make it do occasional sleeps, and got better results than TEO for both throughput and power consumption. The pgbench benchmark showed mixed results, with things getting worse as more [![\[Pratik Sampat and Rafael Wysocki\]](https://static.lwn.net/images/conf/2020/ospm/Sampat-Wysocki-sm.png)](<https://lwn.net/Articles/820440/>) clients were added. Hackbench results saw better results with relatively short run times, and a consistent 8-10% improvement in power consumption. 

At this point, some confusion about the results became evident. Sampat characterized the results as "overshooting" or "undershooting", which most people expected to refer to the sleep state chosen, but actually referred to the sleep residency time. So "overshooting" meant picking a sleep state that was too shallow — the residency time overshot the estimate. This terminology seems highly likely to change in the near future. 

Wysocki observed that picking a sleep state that is too shallow is generally better than picking one that is too deep. Not sleeping deeply enough will cost some power, but sleeping too deeply can hurt the performance of the system (in both latency and throughput terms). 

Sampat finished with an overview of the work that is yet to be done. The aging algorithm still needs some work; workloads change over time, and old history can lead to poor predictions going forward. He tried simply decaying the highest-probability state, but that led to large variance in the results. 

Another issue is the initial weights put in the matrix; these were determined through experiments, but more testing is needed. Wysocki disagreed, though, saying that with proper aging, the initial states don't matter much. The governor will correct itself over time. But that depends on the aging working well, so that is the important part to work on. The session concluded with Wysocki saying that the work looks promising and can be discussed further on the mailing list.
