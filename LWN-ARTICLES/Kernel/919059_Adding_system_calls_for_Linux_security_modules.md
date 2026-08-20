---
title: Adding system calls for Linux security modules
url: https://lwn.net/Articles/919059/
date: "January 4, 2023"
category: "Security-Security modules"
author: "By Jake Edge January 4, 2023"
---

By **Jake Edge**  
January 4, 2023

The Linux security module (LSM) subsystem has long had limitations on which modules could be combined in a given running kernel. Some parts of the problem have been solved over the years—"smaller" LSMs can be combined at will with a single, more complex LSM—but combining (or "stacking") SELinux with, say, Smack or AppArmor has never been possible. Back in October, we [looked](<https://lwn.net/Articles/912775/>) at the most recent attempt to add that ability, which resulted in patches to add two new system calls for LSM. By the end of December, the number of new system calls had risen to three. 

The underlying problem that Casey Schaufler is trying to solve is the handling of the multiple security contexts and how to report them to user space; that is one of the barriers to stacking two or more context-using LSMs. These contexts are a string representation of the information used by an LSM to make its access-control decisions. Schaufler's efforts to fully solve the LSM-stacking problem have now stretched over the last ten years. 

His original plan for reporting the contexts was to simply have multiple null-terminated entries in the relevant `/proc` file for each attribute when there were multiple context-using LSMs active in the system. Multi-LSM-aware applications could be modified to read past the first null to see if there was additional context information. But in September, new LSM maintainer Paul Moore looked at the patches again with an eye toward merging them. That API, which even Schaufler was not particularly happy with, was not one that Moore was willing to add. He [suggested](<https://lwn.net/ml/linux-security-module/CAHC9VhTDGwO789t59EyOV0SwnwGrdyBhRiuJpoY7cB4MSe02BQ@mail.gmail.com/>) that the time had come to add some LSM-specific system calls to make that information available. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

After some grumbling, Schaufler came up with some patches along the lines of what Moore had suggested. [Version 1 of the patch set](<https://lwn.net/ml/linux-kernel/20221025184519.13231-1-casey@schaufler-ca.com/>) added `lsm_self_attr()` for retrieving attributes for the current process and `lsm_module_list()` to list the active LSMs in the system. In [version 4](<https://lwn.net/ml/linux-kernel/20221229233454.43880-1-casey@schaufler-ca.com/>), which was posted December 29, `lsm_self_attr()` had become `lsm_get_self_attr()`; the patch set added `lsm_set_self_attr()`, which will allow setting security attributes for the current process. 

The comments on this round are scant—non-existent as of this writing—which may reflect a lack of attention during the holiday season or that the code is close to ready to merge. The [previous version](<https://lwn.net/ml/linux-kernel/20221123201552.7865-1-casey%40schaufler-ca.com/>) of the patch set also added the three calls and was not particularly contentious when it was posted toward the end of November. That might suggest the latter interpretation. 

Other than the addition of an interface to set the security attributes, there are not a lot of changes from the original proposal. The "reserved IDs" for LSMs has remained, though there have been multiple objections to them along the way. Essentially, all of the uses of the integer LSM IDs (to identify a specific LSM) start at 100 (`LSM_ID_CAPABILITY` for the capability LSM) and every positive integer below that is reserved for future use. Moore was [adamant](<https://lwn.net/ml/linux-kernel/CAHC9VhR9h1GF6VGovp1+UB-vt+QNofjmecPwLqE3OviKQHRMcg@mail.gmail.com/>) that a pool of reserved IDs remain; this round of the patch set has a comment in `lsm.h` that explains how those might be used: 

> The LSM infrastructure itself has no variable state, but that may change. One proposal would allow loadable modules, in which case an attribute such as LSM_IS_LOADABLE might identify the dynamic modules. Another potential attribute could be which security [module] is associated with network labeling using netlabel. Another possible attribute could be related to stacking behavior in a namespaced environment. While it would be possible to intermingle the LSM infrastructure attribute values with the security module provided values, keeping them separate provides a clearer distinction. 

Looking at the most recent patches, the LSM listing function looks much as it did before: 

```
int lsm_module_list(unsigned int *ids, size_t *size, unsigned long flags);
```

In `ids`, it returns an array of LSM IDs for the active LSMs in the system in the order they were added, as long as there are sufficient entries in `size`. `flags` is reserved for future uses and must be zero. The return value is the number of IDs returned or a negative error code. 

Likewise, `lsm_get_self_attr()` picks up from what the call that it replaced (`lsm_self_attr()`) did: fill in an array of `struct lsm_ctx` entries with all of the attributes that apply to the current process for all of the active LSMs. That structure looks much the same as it did before: 

```
struct lsm_ctx {
    	    __u32		id;
    	    __u64		flags;
    	    __kernel_size_t	ctx_len;
    	    __u8		ctx[];
        };
```

The `id` holds the LSM ID and `flags` is unused as yet; `ctx_len` is the size of the `ctx` array that holds the value. `ctx_len` is at least `strlen(ctx)+1` in size since `ctx` is always null-terminated. The description of the returned array of `lsm_ctx` values can be found in the [patch that adds `lsm_get_self_attr()`](<https://lwn.net/ml/linux-kernel/20221229233454.43880-5-casey@schaufler-ca.com/>). 

The attributes that are currently defined are described in the documentation file (`userspace-api/lsm.rst`) that appears near the top of the [first patch](<https://lwn.net/ml/linux-kernel/20221229233454.43880-2-casey@schaufler-ca.com/>). They consist of a half-dozen types of attributes that are currently available in `/proc` files, such as `LSM_ATTR_CURRENT`, which is the active security context for the process (i.e. `/proc/attr/current`). That attribute is the only one that is shared between all three of the context-using LSMs (SELinux, Smack, and AppArmor). Two of the other attributes (`LSM_ATTR_EXEC` for the context when the process was executed and `LSM_ATTR_PREV` for the previous security context, if any) are shared between SELinux and AppArmor. The others currently only pertain to SELinux. The "Files and directories" section of the [proc man page](<https://man7.org/linux/man-pages/man5/proc.5.html>) has some more information on the attributes and how they are used. 

The call looks basically the same as its predecessor: 

```
int lsm_get_self_attr(struct lsm_ctx *ctx, size_t *size, unsigned int flags);
```

It will return the array in `ctx`, with `size` updated to the returned length. The previous version required `flags` to be zero, but now it can be used to restrict which of the attributes are returned by setting individual bits for each of the attributes of interest. 

The counterpart to that is the new `lsm_set_self_attr()`: 

```
int lsm_set_self_attr(struct lsm_ctx *ctx, size_t size, unsigned int flags);
```

It takes a pointer to a single `lsm_ctx` (of the given `size`) and tries to set the value of the (single) attribute identified in `flags`. The function calls into the LSM code to set the value, which allows the security modules to decide whether to allow the operation or not. The function returns zero on success or a negative error code if it fails. 

Since it was initially posted, a [set of kernel self-tests](<https://lwn.net/ml/linux-kernel/20221229233454.43880-9-casey@schaufler-ca.com/>) for the system calls have been added to the patch set. There were few real complaints about the previous version, so one might just conclude that these system calls are about ready to merge, which would be a significant step toward Schaufler's longtime goal. Perhaps before the end of 2023, we will see the quest for this phase of LSM stacking come to a close. Next up would be completing the handling the networking side of the stacking problem, which will lead to the "[universal stacking](<https://lwn.net/ml/linux-security-module/7d0feee1-a9e6-ac82-6962-89b17af025eb@schaufler-ca.com/>)" nirvana that Schaufler has been seeking—lo these many years.
