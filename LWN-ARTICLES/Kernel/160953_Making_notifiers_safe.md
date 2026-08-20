---
title: Making notifiers safe
url: https://lwn.net/Articles/160953/
date: "November 21, 2005"
category: "Notifiers; Race conditions; Read-copy-update"
author: 
---

The kernel contains a mechanism, called "notifiers" or "notifier chains," which allows kernel code to ask to be told when something interesting happens. A number of notifier chains are currently in use in the kernel; chains exist for memory hotplug events, CPU frequency policy changes, USB hotplug events, module loading and unloading, system reboots, network device changes, and more. Notifiers are a simple and easy way to get the word out, so they are increasingly being used throughout the kernel. 

The interface to notifiers is simple. There is one structure type: 

```
struct notifier_block
        {
            int (*notifier_call)(struct notifier_block *self, 
                                 unsigned long event, void *data);
            struct notifier_block *next;
            int priority;
        };
```

A notifier chain is thus a simple, singly-linked list with no separate head. A kernel subsystem which wishes to be notified of specific events fills out a `notifier_block` structure and passes it to: 

```
int notifier_chain_register(struct notifier_block **chain, 
                                      struct notifier_block *notifier);
```

The chain is kept sorted in increasing priority order. Sending out an event is a matter of calling: 

```
int notifier_call_chain(struct notifier_block **chain, 
                                unsigned long event, void *data);
```

Notifiers registered in the chain will be called, in increasing priority order, with the given `event` and `data` values. Any notifier can return a value with the `NOTIFY_STOP_MASK` bit set, with the result that no further notifiers will be called. The return value from the last notifier is return from `notify_call_chain()`. In some cases, the combination of `NOTIFY_STOP_MASK` and the return value is used to allow notifiers to veto proposed actions. 

The current notifier implementation is quite simple, not much more than one page of code. Alan Stern recently noticed a little problem, however: `notifier_call_chain()` goes through the list without any sort of locking. Changes to the notifier list are protected by a global notifier lock, but that lock is ignored when notifiers are called. Thus, if `notifier_call_chain()` is called while some other part is adding or removing notifiers, a mess could result. 

One might be tempted to fix the problem by simply acquiring the lock in `notifier_call_chain()`, but life it not so simple. The current lock for notifiers is a spinlock, but, as it turns out, some notifier functions can sleep. So holding the lock while calling notifiers is not possible. Switching the lock to a semaphore is also out for similar reasons: some notifier chains can be called from atomic contexts. So a more complicated fix is called for. 

That fix has been [posted](<https://lwn.net/Articles/160967/>) by Chandra Seetharaman. It appears that notifier chains have to be split into two types: those which can sleep, and those which are entirely atomic. A new `notifier_type` enum has been created with two values: `ATOMIC_NOTIFIER` and `BLOCKING_NOTIFIER`. There is also now an explicit type (`struct notifier_head`) for the head of a notifier chain. Chains are now declared with something like: 

```
NOTIFIER_HEAD(name, type);
```

Some new rules have been adopted for notifiers as well; one of those is that notifiers are only added or removed in non-atomic context. With that rule in place, each `notifier_head` structure can contain a semaphore (an `rwsem`, actually) which protects access to the chain. The new registration function is: 

```
int notifier_chain_register(struct notifier_head *chain,
                                    struct notifier_block *notifier);
```

Addition of a notifier is relatively easy to do in a safe manner. The "next" pointer in the new entry is set first, followed by the "next" pointer in the appropriate place in the list. By throwing in some memory barriers, the patch ensures that the chain is always in a consistent state. 

The new form of `notifier_call_chain()` is: 

```
int notifier_call_chain(struct notifier_head *chain,
                                unsigned long event, void *data);
```

If the chain is of the `BLOCKING_NOTIFIER` variety, `notifier_call_chain()` can simply acquire the chain semaphore and call the notifiers safely. Acquiring the semaphore is not possible for `ATOMIC_NOTIFIER` chains, however, so, in that case, the code simply calls `rcu_read_lock()` to ensure that it will not be preempted while calling the notifiers. 

The new prototype for the unregistration function is: 

```
int notifier_chain_unregister(struct notifier_head *chain,
                                      struct notifier_block *notifier);
```

For blocking chains, removal of notifiers is straightforward; the code can simply acquire the semaphore and do its work knowing that nobody else will be traversing the chain. For atomic notifiers, however, `notifier_call_chain()` does not acquire the semaphore, so the possibility of races is real. Removing the notifier from the chain is still straightforward: a single pointer assignment takes the notifier out in an atomic manner. But code in another processor may have stumbled across that notifier before it was removed from the chain; in that case, it may still have a reference to it. So the destruction of the removed notifier must wait until the kernel can be sure that no references remain. 

This is just the sort of situation that the read-copy-update (RCU) mechanism was created for. In many applications, the way to destroy this structure would be to set up an `rcu_head` structure, pass it to `call_rcu()`, and wait for a callback to finish the job. In this case, however, callers to `notifier_chain_unregister()` are not expecting callbacks later on, and, in any case, notifier removal is not a performance-critical operation. So the unregister code simply calls `synchronize_rcu()` to block until all current RCU read locks have been released. Once `synchronize_rcu()` has returned, the unregistration code can safely return as well, knowing that no references to the removed notifier exist. 

The new design adds one other new constraint: notifiers cannot remove themselves from the chain. Both the use of the semaphore and the use of RCU would lead to deadlocks in that situation, resulting in developer notifications by way of bugzilla and annoyed email.
