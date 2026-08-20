---
title: Another container implementation
url: https://lwn.net/Articles/200073/
date: "September 19, 2006"
category: "Containers; Virtualization-Containers"
author: 
---

Containers have been an area of increased developer interest over the last year or so. The container concept offers many of the advantages of full paravirtualization, but at a much lower cost, allowing more virtual machines to be run on the same host. The only problem is getting everybody to agree about just what a container is. The recent [container patch set](<https://lwn.net/Articles/199643/>) from Rohit Seth is another attempt to flesh out this concept. 

Many approaches to containers are oriented around process trees - one process explicitly encloses itself within a container, and becomes the "init" process there; the container is then populated with the children of the initial process. Rohit's patch maintains part of that functionality - when a process calls `fork()`, the child will belong to the same container as the parent (if any), but the mechanism is a bit more flexible than that. Arbitrary processes can be added to - and removed from - a container at any time. 

Such changes are effected through a [configfs](<https://lwn.net/Articles/148180/>) interface. If configfs is mounted on `/config`, the system administrator can work with containers by moving into `/config/containers`. A new container is created by making a new directory there; containers, thus, are identified through a simple, flat namespace. A container's directory contains several files: 

  * `addtask`: writing a process ID into this file will add the corresponding process to the container. Processes already belonging to a container cannot be added directly to a new container; they must be explicitly removed from the old one first. 

  * `rmtask`: a process may be removed from a container by writing its ID to this file. 

  * `page_limit`: the maximum number of active memory pages which may be used by the container. 

There are also a few informational files for getting statistics about how the container is operating. 

The memory limit works by adding a container pointer to each `mm_struct` and `address_space` structure on the system. As pages are used or freed, the container's total count is updated accordingly. Should the container go over its limit, a separate process (a workqueue) goes to work freeing up pages belonging to the container. If the limit is exceeded in a big way, processes within the container will (when they try to add pages) be put on hold briefly to let the reaper catch up. 

Rohit's containers are thus concerned with controlling aggregate resource usage. In this sense, they resemble the [resource beancounters](<http://lwn.net/Articles/197433/>) patch - but they do not use any of the beancounter code. These containers also lack one other feature found in most other implementations: any sort of namespace control. Processes placed into one of these containers will still see - and have access to - the entire system. 

So these containers are only a partial solution to the problem, at least at this point. Namespace control features could presumably be added later on, though how that control would interact with the ability to add and remove processes at arbitrary times would be interesting to see. Meanwhile we have another approach to (at least part of) the problem to look at.
