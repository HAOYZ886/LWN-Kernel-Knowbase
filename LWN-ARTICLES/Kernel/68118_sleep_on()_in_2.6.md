---
title: sleep_on() in 2.6.
url: https://lwn.net/Articles/68118/
date: "January 27, 2004"
category: Race conditions; sleep on
author: 
---

One of the many goals for the 2.5 development series was the removal of the `sleep_on()` function (and its variants). The purpose of `sleep_on()` is to cause a process to block until some condition comes true; unfortunately, it is almost impossible to use safely. Almost every call to `sleep_on()` looks something like the following: 

```
while (we_have_to_wait)
    	sleep_on(&some_wait_queue);
```

The problem is that the situation can change between the test (in the `while` loop) and when the process actually goes to sleep. If the wakeup event happens between the two, the process will miss it and may sleep forevermore. Given that 2.6 was intended to be a more responsive kernel than its predecessors, this behavior is considered undesirable. The only way to avoid it, however, is to hold the Big Kernel Lock (BKL) in the code which calls `sleep_on()` \- and the code which performs the wakeup. Since elimination of the BKL was also on the to-do list, there is little enthusiasm for fixing `sleep_on()` race conditions that way. 

The 2.4 kernel provided a couple of safer ways to sleep: the `wait_event()` macro or a full "manual sleep" calling `schedule()` directly (though the latter can be hard to do correctly). In 2.5, the `prepare_to_sleep()` function was added as an easier (and better performing) way of doing manual sleeps. Even so, the 2.6.2-rc2 kernel still has over 400 calls to the various forms of `sleep_on()`. Clearly, the goal of getting rid of that function was not achieved. 

At this point, many people will have concluded that the effort to remove `sleep_on()` has been put on hold until 2.7 opens up. It seems, however, that most users of `sleep_on()` may yet get fixed in 2.6. In response to some discussion on the topic, Al Viro [stated](<https://lwn.net/Articles/68122/>): 

We need to remove racy uses anyway - that can't wait for 2.7. And I really wonder if there will be anything left after that - right now only reiserfs uses look like something that might be not immediately broken. 

He also noted that any use of `sleep_on()` within device drivers is inherently broken. 

Andrew Morton took the next step in [2.6.2-rc1-mm2](<https://lwn.net/Articles/67628/>); that kernel includes a patch which dumps out a bunch of debugging information whenever `sleep_on()` is called without the BKL held. That code has already turned up a few bad calls which have been duly reported to the kernel list. Fixes for those calls have been somewhat slower in coming. They will likely arrive, however, and as Al speculated, by the time all the bad calls are fixed there may not be a whole lot left. `sleep_on()` will undoubtedly exist when the 2.7.0 kernel is released, but there may be very few callers of it by then.
