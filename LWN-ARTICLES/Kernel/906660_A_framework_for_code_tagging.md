---
title: A framework for code tagging
url: https://lwn.net/Articles/906660/
date: "September 1, 2022"
category: Build system
author: "By Jonathan Corbet September 1, 2022"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
September 1, 2022

Kernel code can, at times, be quite inward looking; it often refers to itself. To enable this introspection, the kernel has evolved several mechanisms for identifying specific locations in the code and carrying out actions related to those locations. The [code-tagging framework](<https://lwn.net/ml/linux-kernel/20220830214919.53220-1-surenb@google.com/>) patch set, posted by Suren Baghdasaryan and Kent Overstreet, is an attempt to replace various _ad hoc_ implementations with a single framework, and to add some new applications as well. 

There are a number of reasons for the kernel to need to identify specific locations within the code. For example, kernel code is not normally allowed to incur page faults, but the functions that access user-space memory will often do just that. To do the right thing in that situation, the kernel build process makes a note of the location of every user-space access operation; when a page fault happens, that list is checked and, if the fault happened in an expected location, it is handled normally. The kernel's [dynamic debugging mechanism](<https://www.kernel.org/doc/html/latest/admin-guide/dynamic-debug-howto.html>) is another example; each debugging print statement is tracked and can be enabled independently. 

The usual trick for implementing this kind of mechanism is to create a special ELF section in the kernel binary; that section is then populated with structures recording the points of interest within the kernel. At run time, the kernel can locate that section, where it will find an array of structures with the needed information. At its core, the tagging framework is a set of functions and macros that make the creation of and access to this special section easier. 

A code tag denotes a location within the code itself; that location is represented by a new structure: 

```
struct codetag {
    	unsigned int flags;
    	unsigned int lineno;
    	const char *modname;
    	const char *function;
    	const char *filename;
        };
```

This structure tracks a location but has no other information; it is meant to be embedded within another structure specific to the tagging application. For example, a large part of the patch set is dedicated to the creation of a mechanism to track memory allocations; it can record how much memory is allocated and freed at each call site, and thus be used to track down memory leaks. To do this, it will create a tag at each allocation location with a structure like: 

```
struct alloc_tag {
    	struct codetag			ct;
    	unsigned long			last_wrap;
    	struct raw_lazy_percpu_counter	call_count;
    	struct raw_lazy_percpu_counter	bytes_allocated;
        };
```

The `raw_lazy_percpu_counter` is a new counter type that is also [added](<https://lwn.net/ml/linux-kernel/20220830214919.53220-4-surenb@google.com/>) by the patch set. At this point we have a structure that can associate these counters with the location stored in the `codetag` structure. 

One of these structures is placed into the special `alloc_tags` ELF section with a bit of macro magic: 

```
#define DEFINE_ALLOC_TAG(_alloc_tag)				\
    	static struct alloc_tag _alloc_tag __used __aligned(8)		\
    	__section("alloc_tags") = { .ct = CODE_TAG_INIT }
```

A bit more macro trickery is then used to replace the existing [`alloc_pages()`](<https://www.kernel.org/doc/html/latest/core-api/mm-api.html#c.alloc_pages>) function with a version that places the tag and remembers allocation calls: 

```
#define alloc_tag_add(_ref, _bytes)					\
        do {								\
    	DEFINE_ALLOC_TAG(_alloc_tag);					\
    	if (_ref && !WARN_ONCE(_ref->ct, "alloc_tag was not cleared"))	\
    	    __alloc_tag_add(&_alloc_tag, _ref, _bytes);			\
        } while (0)
    
        #define pgtag_alloc_pages(gfp, order)				\
        ({									\
    	struct page *_page = _alloc_pages((gfp), (order));		\
    									\
    	if (_page)							\
    	    alloc_tag_add(get_page_tag_ref(_page), PAGE_SIZE << (order));\
    	_page;								\
        })
    
        #define alloc_pages(gfp, order) pgtag_alloc_pages(gfp, order)
```

The end result is that each call to `alloc_pages()` is changed to create a static `alloc_tag` structure that records the location of the call site; this structure is placed in the `alloc_tags` section. When an allocation call is made, the two counters in that structure are incremented accordingly (in the not-shown `__alloc_tag_add()` function). Behind the scenes, the code also makes a note (in the [`page_ext` structure](<https://elixir.bootlin.com/linux/v5.19.5/source/include/linux/page_ext.h#L28>) for the allocated pages) of the tag location for the allocation call site; this lets the kernel track which call site allocated each page. When the allocated pages are later freed, that information can be used to decrement the counts for that call site. 

What comes out of all this work is an array of `alloc_pages()` call sites, each of which tracks the amount of memory that was allocated there and which has not yet been freed. The framework also includes infrastructure for iterating through this array and for presenting its contents in the debugfs filesystem. It is not hard to see how this information could be useful for a developer trying to track down a memory leak. Other patches in this series add similar tracking to the slab allocator and the ability to store the call stack for each allocation, giving more information on where the real source of a memory leak might be. 

An entirely different application of this framework is dynamic fault injection. Driver code could, for example, include a sequence like: 

```
if (dynamic_fault("foo-driver-init"))
            return -EIO;  /* Simulate a failure */
```

The `dynamic_fault()` function, once again, places a code tag at the call site. It normally returns false, so the simulated failure code is not run. There is a knob that will appear under `/sys/kernel/debug/dynamic_faults`, though, that can be used to enable this fault site and test whether the driver's error handling works correctly. 

There is even more in the patch series, including a latency-tracking mechanism and a reimplementation of the dynamic debugging facility. The point that is being made is that the code-tagging framework makes it relatively easy to add this sort of feature to the kernel in a way that has a minimal performance impact. 

Most of the early discussion around this patch set has been inspired by [Peter Zijlstra's question](<https://lwn.net/ml/linux-kernel/Yw8P8xZ4zqu121xL@hirez.programming.kicks-ass.net/>) about just what this facility adds that is not already provided by the kernel's tracepoint mechanism. Overstreet [responded](<https://lwn.net/ml/linux-kernel/20220831155941.q5umplytbx6offku@moria.home.lan/>), somewhat defensively, that there are a number of advantages to the code-tagging mechanism. They include capturing all activity from boot rather than just from when tracing was started, better performance, better ease of use, and no problems with dropped events. He said that the question should be asked the other way around: tracing proponents should show how that subsystem could be used to provide a similar capability with comparable performance and ease of use. 

In response, Zijlstra [pointed out](<https://lwn.net/ml/linux-kernel/YxBZv1pZ6N2vwcP3@hirez.programming.kicks-ass.net/>) that use of ftrace is not necessary to attach to tracepoints; attaching custom handlers to tracepoints would address concerns about performance and dropped events. Mel Gorman [added](<https://lwn.net/ml/linux-kernel/20220901110501.o5rq5yzltomirxiw@suse.de/>) that the tracepoint approach is more flexible, works with older kernels, and is more widely available. He also pointed to [a patch set from Oscar Salvador](<https://lwn.net/ml/linux-kernel/20220901044249.4624-1-osalvador@suse.de/>) implementing a different approach to memory-leak detection. Michal Hocko [worried](<https://lwn.net/ml/linux-kernel/YxBc1xuGbB36f8zC@dhcp22.suse.cz/>) about the difficulties of reviewing and maintaining a patch set of this size. 

This is a new and large patch set; it is likely to be under discussion for some time. The code-tagging part itself seems like it should be a relatively uncontroversial cleaning up of the code; it can, in theory, replace a number of independent implementations in the kernel with a single framework. Each of the add-on changes is likely to require additional discussion, though; one doesn't just walk into the memory-management subsystem and change the core allocator code without having to answer some questions. Chances are that this patch set will end up being split into its various components somewhere along the way so that each can be considered on its own merits.
