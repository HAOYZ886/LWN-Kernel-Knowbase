---
title: Revocable references for transient devices
url: https://lwn.net/Articles/1038523/
date: "September 22, 2025"
category: "Device drivers-Support APIs; Race conditions"
author: "By Jonathan Corbet September 22, 2025"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
September 22, 2025

Computers were once relatively static devices; if a peripheral was present at boot, it was unlikely to disappear while the system was operating. Those days are far behind us, though; devices can come and go at any time, often with no notice. That impermanence can create challenges for kernel code, which may not be expecting resources it is managing to make an abrupt exit. The [revocable resource management patch set](<https://lwn.net/ml/all/20250912081718.3827390-1-tzungbi@kernel.org>) from Tzung-Bi Shih is meant to help with the creation of more robust — and more secure — kernel subsystems in a dynamic world. 

Low-level drivers that manage transient devices must be prepared for those devices to vanish; there is no way around that. But those drivers often create data structures that are associated with the devices they manage, and export those structures to higher-level software. That creates a different sort of problem: the low-level driver knows when it can no longer communicate with a departed device, but it can be harder to know when it can safely free those data structures. Higher-level code may hold references to those structures for some time after the device disappears; freeing them prematurely would expose the kernel to use-after-free bugs and the exploits that are likely to follow. 

Shih's patch set addresses this problem by turning those problematic references into a form of weak reference. The holder of such a reference must validate it prior to each use, and access to the resource can be revoked at (almost) any time. When used correctly, this API is meant to ensure that resources stay around as long as users exist, and to let the provider of those resources know when they can be safely freed. 

Consider an entirely made-up example of a serial-port device that can come and go at the user's whim. When one of these devices is discovered, the driver will create a structure describing it that can be used by higher levels of software. An example user of this structure could be the [SLIP](<https://en.wikipedia.org/wiki/Serial_Line_Internet_Protocol>) network-interface driver, which many Linux users happily forgot about decades ago, but which still exists and is useful to, for example, communicate with your [vape-powered web server](<https://bogdanthegeek.github.io/blog/projects/vapeserver/>). The serial and SLIP drivers must have some sort of agreement regarding when the low-level structures can be accessed, or the state of the system will take a turn for the worse. 

The resource provider (the low-level driver) starts by creating a `revocable_provider` structure for the structure that it will make available to higher levels: 

```
struct revocable_provider *stuff = revocable_provider_alloc(void *res);
```

The `res` parameter should point to the resource that is being managed. The user of this resource (the higher-level driver), instead, should obtain a `revocable` structure with call to: 

```
struct revocable *revocable_alloc(struct revocable_provider *prov);
```

In the [example code](<https://lwn.net/ml/all/20250912081718.3827390-6-tzungbi@kernel.org>) provided with the patch set, the resource consumer has direct access to the `revocable_provider` structure and makes the call itself. One could also imagine an API where the provider hides that structure and allocates the `revocable` structure for the user. 

Sooner or later, the user will need access to the underlying resource. It does not have a pointer to that resource, though; it only has the `revocable` structure. To get that pointer, it must make a call to: 

```
void *revocable_try_access(struct revocable *resource);
```

In normal operation, where the device still exists, this call will do two things: it enters a [sleepable RCU (SRCU)](<https://lwn.net/Articles/202847/>) read-side critical section, and it returns the pointer that the caller needs. The caller can then use that pointer to access the resource, secure in the knowledge that the pointer is valid and will continue to be for the duration of the access. Once the task at hand is complete, the user calls: 

```
void revocable_release(struct revocable *resource);
```

That exits the SRCU critical section, and releases the reference to the resource; the caller should not retain the pointer to that resource past this call. There is also a convenience macro provided for this pair of calls: 

```
REVOCABLE(struct revocable *resource, void *ptr) {
        	/*
    	 * "ptr" will point to the resource here.  It may be NULL,
    	 * though, so code must always check.
    	 */
        }
```

As the comment above suggests, a call to `revocable_try_access()` might return `NULL`, which is an indication that access to the resource has been revoked. Needless to say, the caller must always check for that case and respond accordingly. 

Revocation happens on the provider side, for example, if the owner of the vape unplugs it to put it to its more traditional use. Once the low-level driver becomes aware that the device has evaporated, it revokes access to the resource it exported with: 

```
void revocable_provider_free(struct revocable_provider *stuff);
```

This call may, of course, race with the activity of users that have legitimately obtained a pointer to the resource with `revocable_try_access()`; as long as those users are out there, the provider cannot free that resource. To handle this situation, `revocable_provider_free()` makes a call to [`synchronize_srcu()`](<https://docs.kernel.org/core-api/kernel-api.html#c.synchronize_srcu>), which will block for as long as any read-side SRCU critical sections for that resource remain active. Once `revocable_provider_free()` returns, the provider knows that no users of the resource remain, so it can be safely freed. 

This whole interface may seem familiar to those who have been watching the effort to enable kernel development in Rust: it is patterned after the [`Revocable` type](<https://rust.docs.kernel.org/kernel/revocable/struct.Revocable.html>) used there. 

The initial response to this work has been mostly positive; Greg Kroah-Hartman, for example, [said:](<https://lwn.net/ml/all/2025091224-blaming-untapped-6883@gregkh>)

> This is, frankly, wonderful work. Thanks so much for doing this, it's what many of us have been wanting to see for a very long time but none of us got around to actually doing it. 

Danilo Krummrich, who wrote the Rust `Revocable` implementation, had [some more specific comments](<https://lwn.net/ml/all/DCQP9ZJ0DFBO.3O3W57IDYN08I@kernel.org>), suggesting changes to the naming and the API. Shih, meanwhile, has [requested a discussion](<https://lwn.net/ml/all/aMWEhqia_jpl12uI@tzungbi-laptop>) of the work at the Kernel Summit, which will be held this December in Tokyo. This is relatively new work, and it seems likely to evolve considerably before it is ready for use in the mainline kernel. Given the potential it has to address the sorts of life-cycle bugs that have plagued kernel developers for years, though, it seems almost certain that some future form of this API will be adopted.
