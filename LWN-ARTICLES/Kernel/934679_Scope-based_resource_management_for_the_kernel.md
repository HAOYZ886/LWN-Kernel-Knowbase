---
title: "Scope-based resource management for the kernel"
url: https://lwn.net/Articles/934679/
date: "June 15, 2023"
category: "C language safety; Releases-6.5; Scoped resource management"
author: "By Jonathan Corbet June 15, 2023"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
June 15, 2023

The C language does not provide the sort of resource-management features found in more recent languages. As a result, bugs involving leaked memory or failure to release a lock are relatively common in programs written in C — including the kernel. The kernel project has never limited itself to the language features found in the C standard, though; kernel developers will happily use extensions provided by compilers if they prove helpful. It looks like a relatively simple compiler-provided feature may lead to a significant change in some common kernel coding patterns. 

The feature, specifically, is the `cleanup` attribute, which is implemented by both [GCC](<https://gcc.gnu.org/onlinedocs/gcc/Common-Variable-Attributes.html#index-cleanup-variable-attribute>) and [Clang](<https://clang.llvm.org/docs/AttributeReference.html#cleanup>). It allows a variable to be declared using a syntax like: 

```
type my_var __attribute__((__cleanup__(cleanup_func)));
```

The extra attribute says that, when `my_var`, a variable of the given `type`, goes out of scope, a call should be made to: 

```
cleanup_func(&my_var);
```

This function, it is assumed, will do some sort of final cleanup on that variable before it disappears forever. As an example, one could declare a pointer (in the kernel) this way: 

```
void auto_kfree(void **p) { kfree(*p); }
    
       struct foo *foo_ptr __attribute__((__cleanup__(auto_kfree))) = NULL;
       /* ... */
       foo_ptr = kmalloc(sizeof(struct foo));
```

Thereafter, there is no need to worry about freeing the allocated memory; once `foo_ptr` goes out of scope, the compiler will ensure that it will be passed to a call to `kfree()`. It is no longer possible to leak this memory — at least, not without actively working at it. 

This attribute is not particularly new, but the kernel has never taken advantage of it. In late May, Peter Zijlstra decided to change that situation, posting [a patch set](<https://lwn.net/ml/linux-kernel/20230526150549.250372621@infradead.org/>) adding "lock and pointer guards" using that feature. A [second version](<https://lwn.net/ml/linux-kernel/20230526205204.861311518@infradead.org/>) followed shortly thereafter and resulted in quite a bit of discussion, with Linus Torvalds [encouraging](<https://lwn.net/ml/linux-kernel/CAHk-=wg2RHZKTN29Gr7MhgYfaNtzz58wry9jCNP75LAmQ9t8-A@mail.gmail.com/>) Zijlstra to generalize the work away from just protecting locks. The result was the [scope-based resource management](<https://lwn.net/ml/linux-kernel/20230612090713.652690195@infradead.org/>) patch set posted on June 12, which creates a new set of macros intended to make the use of the `cleanup` attribute easy. The 57-part patch set also converts a lot of code to use the new macros, giving an extensive set of examples of how they would change the look of the kernel code base. 

#### Cleanup functions in the kernel

The first step is to define a new macro, `__cleanup()`, which abbreviates the attribute syntax shown above. Then, a set of macros makes it possible to create and manage a self-freeing pointer: 

```
#define DEFINE_FREE(name, type, free) \
    	static inline void __free_##name(void *p) { type _T = *(type *)p; free;}
    
        #define __free(name)	__cleanup(__free_##name)
    
        #define no_free_ptr(p) \
    	({ __auto_type __ptr = (p); (p) = NULL; __ptr; })
    
        #define return_ptr(p)	return no_free_ptr(p)
```

The purpose of `DEFINE_FREE()` is to associate a cleanup function with a given type (though the "type" is really just a separate identifier than is not associated with any specific C type). So, for example, a free function can be set up with a declaration like: 

```
DEFINE_FREE(kfree, void *, if (_T) kfree(_T))
```

Within the macro, this declaration is creating a new function called `__free_kfree()` that makes a call to `kfree()` if the passed-in pointer is not `NULL`. Nobody will ever call that function directly, but the declaration makes it possible to write code like: 

```
struct obj *p __free(kfree) = kmalloc(...);
    
        if (!p)
            return NULL;
        if (!initialize_obj(p))
            return NULL;
        return_ptr(p);
```

The `__free()` attribute associates our cleanup function with the pointer `p`, ensuring that that `__free_kfree()` will be called when that pointer goes out of scope, regardless of how that happens. So, for example, the second `return` statement above will not leak the memory allocated for `p`, even though there is no explicit `kfree()` call. 

Sometimes, though, the automatic freeing isn't wanted; the case where everything goes as expected and a pointer to the allocated object should be returned to the caller is one example. The `return_ptr()` macro, designed for this case, defeats the automatic cleanup by copying the value of `p` to another variable, setting `p` to `NULL`, then returning the copied value. There are usually many ways in which things can go wrong and only one way where everything works, so arguably it makes more sense to annotate the successful case in this way. 

#### From cleanup functions to classes

Automatic cleanup functions are a start, but it turns out that there's more that can be done using this compiler feature. After some discussion, it was decided that the best name for a more general facility to handle the management of resources in the kernel was "class". So, the next step is to add "classes" to the C language as is used by the kernel: 

```
#define DEFINE_CLASS(name, type, exit, init, init_args...)		\
            typedef type class_##name##_t;					\
    	static inline void class_##name##_destructor(type *p)		\
    	    { type _T = *p; exit; }					\
    	static inline type class_##name##_constructor(init_args)	\
    	    { type t = init; return t; }
```

This macro creates a new "class" with the given `name`, encapsulating a value of the given `type`. The `exit()` function is a destructor for this class (the cleanup function, in the end), while `init()` is the constructor, which will receive `init_args` as parameters. The macro defines a type and a couple of new functions to handle the initialization and destruction tasks. 

The `CLASS()` macro can then be used to define a variable of this class: 

```
#define CLASS(name, var)						\
    	class_##name##_t var __cleanup(class_##name##_destructor) =	\
    		class_##name##_constructor
```

This macro is substituted with a declaration for a variable `var` that is initialized with a call to the constructor. Note that the result is an incomplete statement; the arguments to the constructor must be provided to complete the statement, as shown below. The use of the `__cleanup()` macro here ensures that the destructor for this class will be called when a variable of the class goes out of scope. 

One use of this macro, as shown in the patch set, is to bring some structure to the management of file references, which can be easy to leak. A new class, called `fdget`, is created that manages the acquisition and release of those references. 

```
DEFINE_CLASS(fdget, struct fd, fdput(_T), fdget(fd), int fd)
```

A constructor (named `class_fdget_constructor()`, but that name will never appear explicitly in the code) is created to initialize the class with a call to [`fdget()`](<https://elixir.bootlin.com/linux/v6.3.8/source/include/linux/file.h#L61>), with the integer `fd` as its parameter. This initialization creates a reference to the file that must, at some point be returned. The class definition also creates a destructor, which calls [`fdput()`](<https://elixir.bootlin.com/linux/v6.3.8/source/include/linux/file.h#L42>), that will be invoked by the compiler when a variable of this class goes out of scope. 

Code that wants to work with a file descriptor `fd` can make use of this class structure with a call like: 

```
CLASS(fdget, f)(fd);
```

This line declares a new variable, called `f`, of type [`struct fd`](<https://elixir.bootlin.com/linux/v6.3.8/source/include/linux/file.h#L35>), that is managed by the `fdget` class. 

Finally, there are macros to define classes related to locks: 

```
#define DEFINE_GUARD(name, type, lock, unlock) \
    	DEFINE_CLASS(name, type, unlock, ({ lock; _T; }), type _T)
        #define guard(name) \
    	CLASS(name, __UNIQUE_ID(guard))
```

`DEFINE_GUARD()` creates a class around a lock type. For example, it is used with mutexes with this declaration: 

```
DEFINE_GUARD(mutex, struct mutex *, mutex_lock(_T), mutex_unlock(_T)):
```

The `guard()` macro then creates an instance of this class, generating a unique name for it (which nobody will ever see or care about). An example of the usage of this infrastructure can be seen in [this patch](<https://lwn.net/ml/linux-kernel/20230612093537.833273038@infradead.org/>), where the line: 

```
mutex_lock(&uclamp_mutex);
```

is replaced with: 

```
guard(mutex)(&uclamp_mutex);
```

After that, the code that explicitly unlocks `uclamp_mutex` can be deleted — as can all of the error-handling code that ensures that the unlock call is made in every case. 

#### The guard-based future

The removal of the error-handling code in the above example is significant. A common pattern in the kernel is to perform cleanup at the end of a function, and to use `goto` statements to jump to an appropriate point in the cleanup code whenever something goes wrong. In pseudocode form: 

```
err = -EBUMMER;
        mutex_lock(&the_lock);
        if (!setup_first_thing())
           goto out;
        if (!setup_second_thing())
           goto out2;
        /* ... */
        out2:
            cleanup_first_thing();
        out:
            mutex_unlock(&the_lock);
            return err;
```

This is a relatively restrained use of `goto`, but it still adds up to vast numbers of `goto` statements in the kernel code and it is relatively easy to get wrong. Extensive adoption of this new mechanism would allow the above pattern to look more like this: 

```
guard(mutex)(&the_lock);
        CLASS(first_thing, first)(...);
        if (!first or !setup_second_thing())
            return -EBUMMER;
        return 0;
```

The code is more compact, and the opportunities for the introduction of resource-related bugs are reduced. 

There's more to these macros than has been discussed here, including a special variant for managing read-copy-update (RCU) critical sections. Curious readers can find the whole set in [this patch](<https://lwn.net/ml/linux-kernel/20230612093537.614161713@infradead.org/>). 

One potentially interesting side-change in the series is the [removal](<https://lwn.net/ml/linux-kernel/20230612093537.693926033@infradead.org/>) of the compiler warning for declarations after the first statement — a warning that has backed up the longstanding requirement in the kernel's coding style to avoid intermixing declarations and statements in that way. It simply was not possible to make these macros work without relaxing that rule. Torvalds [agreed](<https://lwn.net/ml/linux-kernel/CAHk-=wi-RyoUhbChiVaJZoZXheAwnJ7OO=Gxe85BkPAd93TwDA@mail.gmail.com/>) with this change, saying that perhaps the rule can be softened somewhat: 

> I think that particular straightjacket has been a good thing, but I also think that it's ok to just let it go as a hard rule, and just try to make it a coding style issue for the common case, but allow mixed declarations and code when it makes sense. 

The reaction to this work has been mostly positive; Torvalds seems to be happy with the general direction of this new mechanism and has [limited himself](<https://lwn.net/ml/linux-kernel/CAHk-=wj3jV+v63RA30eVpjHVyrGmmTF7d3ajdV=1wBU=5OHa=A@mail.gmail.com/>) to complaining about potential bugs in a couple of specific conversions and the length of the patch series in general. So it seems reasonably likely that something like this will find its way into a future kernel release. The result could be safer resource management in the kernel and a lot fewer `goto`s.
