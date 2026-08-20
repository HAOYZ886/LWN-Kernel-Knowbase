---
title: The MD roadmap
url: https://lwn.net/Articles/428220/
date: "February 16, 2011"
category: "Block layer-RAID; RAID"
author: "By Jonathan Corbet February 16, 2011"
---

By **Jonathan Corbet**  
February 16, 2011

Users of the MD (multiple disk or RAID) subsystem in Linux may be interested in the [MD roadmap](<https://lwn.net/Articles/428206/>) posted by maintainer Neil Brown. It discusses a number of things he has planned for MD in quite a bit of detail; as Neil put it: 

A particular need I am finding for this road map is to make explicit the required ordering and interdependence of certain tasks. Hopefully that will make it easier to address them in an appropriate order, and mean that I waste less time saying "this is too hard, I might go read some email instead". 

There are a lot of enhancements in the pipeline. A bad block log would allow RAID arrays to continue functioning in the presence of bad blocks without needing to immediately eject the offending drive. There is a variant on "hot replace" which would allow a new drive to be inserted before removing the old one, thus allowing the array to continue with a full complement of drives while the new one is being populated. Tracking of areas which are known not to contain useful data would reduce synchronization costs. A number of proposed enhancements to the "reshape" functionality would make it more robust and flexible and allow operations to be undone. A number of other changes are contemplated as well; see Neil's post for the full list.
