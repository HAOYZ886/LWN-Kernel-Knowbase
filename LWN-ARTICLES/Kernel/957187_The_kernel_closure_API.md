---
title: "The kernel \"closure\" API"
url: https://lwn.net/Articles/957187/
date: "January 11, 2024"
category: Closures
author: "By Jonathan Corbet January 11, 2024"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
January 11, 2024

The data structure known as a "closure" first found its way into the mainline kernel with the addition of [bcache](<https://bcache.evilpiepirate.org/>) in the 3.10 development cycle. With the advent of [bcachefs](<https://bcachefs.org/>) in 6.7, though, it acquired a second user and was moved to the kernel's `lib` directory, making it available to other kernel users as well. The documentation of closures in the source is better than that of many things in the kernel, but there is still room for a gentler introduction. 

As [`include/linux/closure.h`](<https://elixir.bootlin.com/linux/v6.7/source/include/linux/closure.h>) notes: ""Closure is perhaps the most overused and abused term in computer science, but since I've been unable to come up with anything better you're stuck with it again"". In the kernel sense, a closure can be thought of as a reference count tracking some number of things that need to happen, along with some synchronization features and a hierarchical organization. 

To start working with closures, one should allocate a structure of type `struct closure` and initialize it with: 

```
#include <linux/closure.h>
    
        closure_init(struct closure *cl, struct closure *parent);
```

Where `cl` is the closure to be initialized, and `parent` is used to create a parent relationship, which will be described below. On return from this call, the caller owns a reference to the closure that must eventually be given back. 

A closure's reference count can be manipulated with: 

```
void closure_get(struct closure *cl);
        void closure_put(struct closure *cl);
```

Closures have a few mildly quirky rules, one of which is that only references obtained with `closure_get()` can be released with `closure_put()`; the initial reference obtained from `closure_init()` is special and must be handled differently. 

There are a couple of ways of managing that initial reference; to understand them, it's worth keeping in mind what closures are for. Essentially, they allow a thread running in the kernel to place one or more operations (a set of I/O requests, for example) in motion and then wait for all of those operations to complete. To do so, that thread will initialize its closure, then start those other operations, each of which will involve calling `closure_get()` to obtain a reference to the closure. As each operation completes, a `closure_put()` call is made. When the closure's reference count drops to one, all of those operations are complete and the next step, whatever it is, can be taken. 

It is up to the creator of the closure to arrange for that next step once the closure has dropped back to just the initial reference. One option for doing that is for the initiating thread to simply wait until the reference count drops by calling both of: 

```
bool closure_wait(struct closure_waitlist *list, struct closure *cl);
        void closure_sync(struct closure *cl);
```

The caller should allocate `list` separately. Another rule of closures is that they can only be on one wait list at a time; if the given `cl` is already on a list, `closure_wait()` will immediately return `false`. Otherwise it will place the closure on the given `list`. A call to `closure_sync()` will then block the current thread until the reference count drops to one. 

If the initiating thread does not want to wait synchronously for the closure to complete, the alternative is to arrange for a sort of callback when the reference count drops to one: 

```
typedef void (closure_fn) (struct closure *);
        void continue_at(struct closure *cl, closure_fn *callback,
        		     struct workqueue *wq);
```

This call will arrange for `callback()` to be called when the last `closure_put()` call is made — the point where only the initial reference to the closure remains. If `wq` is non-NULL, it specifies the workqueue to be used to make this call; otherwise the call will made directly from `closure_put()`. The call to `continue_at()` drops the caller's reference to `cl` (which, remember, is the initial reference created when the closure was set up), so the caller should not touch it further; indeed, the rules for closures say that the caller should return immediately after the call. 

The way to destroy a closure is to call `continue_at()` with a NULL `callback()` pointer; that is the signal that the closure is done. The macro `closure_return()` is defined as a shorthand for this call: 

```
#define closure_return(_cl)	continue_at((_cl), NULL, NULL)
```

There is also a variant, `closure_return_with_destructor()`, that takes a second `closure_fn()` pointer indicating a function to call when all references have been dropped and the closure is finished. 

As noted above, closures can be initialized with a parent pointer; this allows the caller to set up a hierarchy of dependent events. When a closure is initialized, it will take a reference (with `closure_get()`) on the parent if one is specified; as a result, the parent will continue to exist for a long as the new closure does. When a closure is finished with the special `continue_at()` call, the reference to the parent will be dropped with `closure_put()`. This mechanism ensures that the parent closure will not complete until all of its child closures have finished. 

Needless to say, there are other complications in the closure API as well, but the above covers the core of it. As of this writing, only bcache and bcachefs use closures. In the past, there have been occasional vague [objections](<https://lwn.net/ml/linux-kernel/20190613072841.GA7996@infradead.org/>) to the closure abstraction, but those have not prevented its use so far. Whether its usage will grow will depend entirely on whether other developers find it useful.
