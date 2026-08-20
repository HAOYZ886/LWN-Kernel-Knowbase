---
title: "Driver porting: mutual exclusion with seqlocks"
url: https://lwn.net/Articles/22818/
date: "February 14, 2003"
category: "Locking mechanisms-seqlocks"
author: 
---

This article is part of the LWN [Porting Drivers to 2.6 series](<https://lwn.net/Articles/driver-porting/>).   
---  
The 2.5.60 kernel added a new type of lock called a "seqlock." Seqlocks are a specialized primitive intended for the following sort of situation: 

  * A small amount of data is to be protected. 
  * That data is simple (no pointers), and is frequently accessed. 
  * Access to the data does not create side effects. 
  * It is important that writers not be starved for access. 

The situation for which seqlocks were originally designed is to control access to system time variables - `jiffies_64` and `xtime`. Those variables are constantly being read, so that action should be fast. It is also important, however, that the update of those variables, which happens in the timer interrupt, not have to wait while the readers clear out. 

Seqlocks consist of a regular spinlock and an integer "sequence" count. They may be declared and initialized in two ways, as follows: 

```
#include <linux/seqlock.h>
    
        seqlock_t lock1 = SEQLOCK_UNLOCKED;
        seqlock_t lock2;
    
        seqlock_init(&lock2);
```

Writers must take out exclusive access before making changes to the protected data. The usual series of events is something like: 

```
seqlock_t the_lock = SEQLOCK_UNLOCKED;
        /* ... */
    
        write_seqlock(&the_lock);
        /* Make changes here */
        write_sequnlock(&the_lock);
```

The call to `write_seqlock()` locks the spinlock and increments the sequence number. When the work is done, `write_sequnlock()` increments the sequence number again, then releases the spinlock. 

Read access to the data uses no locking at all; instead, the reader uses the lock's sequence number to detect access collisions with a writer and retry the read if necessary. The code tends to look like: 

```
unsigned int seq;
    
        do {
            seq = read_seqbegin(&the_lock);
    	/* Make a copy of the data of interest */
        } while read_seqretry(&the_lock, seq);
```

The call to `read_seqretry()` makes a couple of simple checks. If the initial sequence number obtained from `read_seqbegin()` is odd, it means that a writer was in the middle of updating the data when the reader began reading. If the initial number does not match the seqlock's sequence number at the end, then a writer showed up in the middle of the process. Either way, the data obtained could be inconsistent, and the reader must go around and try again. In the most common case, though, no collision will occur, and the reader gets very fast access with no locking or retries required. 

Of course, the usual variants on the locking primitives exist for exclusion of local interrupts or bottom halves; for reference, here's the full set: 

```
void write_seqlock(seqlock_t *sl);
        void write_sequnlock(seqlock_t *sl);
        int write_tryseqlock(seqlock_t *sl);
        void write_seqlock_irqsave(seqlock_t *sl, long flags);
        void write_sequnlock_irqrestore(seqlock_t *sl, long flags);
        void write_seqlock_irq(seqlock_t *sl);
        void write_sequnlock_irq(seqlock_t *sl);
        void write_seqlock_bh(seqlock_t *sl);
        void write_sequnlock_bh(seqlock_t *sl);
    
        unsigned int read_seqbegin(seqlock_t *sl);
        int read_seqretry(seqlock_t *sl, unsigned int iv);
        unsigned int read_seqbegin_irqsave(seqlock_t *sl, long flags);
        int read_seqretry_irqrestore(seqlock_t *sl, unsigned int iv, long flags);
```
