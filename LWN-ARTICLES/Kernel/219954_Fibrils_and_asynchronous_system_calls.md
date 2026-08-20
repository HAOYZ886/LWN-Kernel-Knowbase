---
title: Fibrils and asynchronous system calls
url: https://lwn.net/Articles/219954/
date: "January 31, 2007"
category: Asynchronous IO; Fibrils
author: 
---

The kernel's support for asynchronous I/O is incomplete, and it always has been. While certain types of operations (direct filesystem I/O, for example) work well in an asynchronous mode, many others do not. Often implementing asynchronous operation is hard, and nobody has ever gotten around to making it work. In other cases, patches have been around for some time, but they have not made it into the mainline; AIO patches can be fairly intrusive and hard to merge. Regardless of the reason, things tend to move very slowly in the AIO area. 

Zach Brown has decided to stir things up by asking a basic question: could it be that the way the kernel implements AIO is all wrong? The current approach adds a fair amount of complexity, requiring explicit AIO handling in every subsystem which supports it. IOCB structures have to be passed around, and kernel code must always check whether it is supposed to block on a given operation or return one of two "it's in the works" codes. It would be much nicer if most kernel operations could simply be invoked asynchronously without having to clutter them up with explicit support. 

To that end, Zach has posted [a preliminary patch set](<http://lwn.net/Articles/219928/>) which simplifies asynchronous I/O support considerably, but doesn't stop there: it also makes any system call invokable in an asynchronous mode. The key is a new type of in-kernel lightweight thread known as a "fibril." 

A fibril is an execution thread which only runs in kernel space. A process can have any number of fibrils active, but only one of them can actually execute in the processor(s) at any given time. Fibrils have their own stack, but otherwise they share all of the resources of their parent process. They are kept in a linked list attached to the task structure. 

When a process makes an asynchronous system call, the kernel creates a new fibril and executes the call in that context. If the system call completes immediately, the fibril is destroyed and the result goes back to the calling process in the usual way. Should the fibril block, however, it gets queued and control returns to the submitting code, which can then return the "it's in progress" status code. The "main" process can then run in user space, submit more asynchronous operations, or do just about anything else. 

Sooner or later, the operation upon which the fibril blocked will complete. The wait queue entry structure has been extended to include information on which fibril was blocked; the wakeup code will find that fibril and make it runnable by adding it to a special "run queue" linked list in the parent task structure. The kernel will then schedule the fibril for execution, perhaps displacing the "main" process. That fibril might make some progress and block again, or it may complete its work. In the latter case, the final exit code is saved and the fibril is destroyed. 

By moving asynchronous operations into a separate thread, Zach's patch simplifies their implementation considerably - with few exceptions, kernel code need not be changed at all to support asynchronous calls. The creation of fibrils is intended to make it all happen quickly - fibrils are intended to be less costly than kernel threads or ordinary processes. Their one-at-a-time semantics help to minimize the concurrency issues which might otherwise come up. 

The user-space interface starts with a structure like this: 

```
struct asys_input {
    	int 		syscall_nr;
    	unsigned long	cookie;
    	unsigned long	nr_args;
    	unsigned long	*args;
        };
```

The application is expected to put the desired system call number in `syscall_nr`; the arguments to that system call are described by `args` and `nr_args`. The `cookie` value will be given back to the process when the operation completes. User space can create an array of these structures and pass them to: 

```
long asys_submit(struct asys_input *requests, unsigned long nr_requests);
```

The kernel will then start each of the requests in a fibril and return to user space. When the process develops an interest in the outcome of its requests, it uses this interface: 

```
struct asys_completion {
    	long 		return_code;
    	unsigned long	cookie;
        };
    
        long asys_await_completion(struct asys_completion *comp);
```

A call to `asys_await_completion()` will block until at least one asynchronous operation has completed, then return the result in the structure pointed to by `comp`. The `cookie` value given at submission time is returned as well. 

Your editor notes that the current `asys_await_completion()` implementation does not check to see if any asynchronous operations are outstanding; if none are, the call is liable to wait for a long time. There are a number of other issues with the patch set, all acknowledged by their author. For example, little thought has been given to how fibrils should respond to signals. Zach's purpose was not to present a completed work; instead, he wants to get the idea out there and see what people think of it. 

Linus [likes the idea](<https://lwn.net/Articles/219957/>): 

Yee-haa! [...] 

I heartily approve, although I only gave the actual patches a very cursory glance. I think the approach is the proper one, but the devil is in the details. It might be that the stack allocation overhead or some other subtle fundamental problem ends up making this impractical in the end, but I would _really_ like for this to basically go in. 

There are a lot of details - Linus noted that there is no limit on how many fibrils a process can create, for example - but this seems to be the way that he would like to see AIO implemented. He suggests that fibrils might be useful in the kevent code as well. 

On the other hand, Ingo Molnar [is opposed](<https://lwn.net/Articles/219958/>) to the fibril approach; his argument is long but worth reading. In Ingo's view, there are only two solutions to any operating system problem which are of interest: (1) the one which is easiest to program with, and (2) the one that performs the best. In the I/O space, he claims, the easiest approach is synchronous I/O calls and user-space processes. The fastest approach will be "a pure, minimal state machine" optimized for the specific task; his Tux web server is given as an example. 

According to Ingo, the fibril approach serves neither goal: 

Now where do all these LWP, fibre, firbril, micro-thread or N:M concepts fit? Most of the time they are just a /weakening/ of the #1 concept. And that's why they will lose out, because #1 is all about programmability and they don't offer anything new: because they cannot. Either you go for programmability or you go for performance. There is /no/ middle ground for us in the kernel! 

Ingo makes the claim that Linux is sufficiently fast at switching between ordinary processes that the advantages offered by fibrils are minimal at best, and not worth their cost. Anybody wanting performance will still have to face the full kernel AIO state machine. So, he says, there is no real advantage to fibrils at this time that are worth the cost of complicating the scheduler and moving away from the 1:1 thread model. 

These patches are in an early stage, and this story will clearly take some time to play out. Even if a consensus develops in favor of the fibril idea, the process of turning them into a proper, robust kernel feature could make them too expensive to be worthwhile. But it's an interesting idea which brings a much-needed fresh look at how the kernel does AIO; it's hard to complain too much about that.
