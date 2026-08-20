---
title: "Filesystem notification, part 2: A deeper investigation of inotify"
url: https://lwn.net/Articles/605128/
date: "July 14, 2014"
category: "Development model-User-space ABI; Inotify"
author: "July 14, 2014 This article was contributed by Michael Kerrisk . Filesystem notification"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

July 14, 2014

This article was contributed by [Michael Kerrisk](<http://man7.org/>).

* * *

[Filesystem notification](<https://lwn.net/Articles/605313/>)

In the [first article](<https://lwn.net/Articles/604686/>) in this series, we briefly looked at the original Linux filesystem notification API, dnotify, and noted a number of its limitations. We then turned our attention to its successor, inotify, and saw how the design of the newer API addressed various problems with the dnotify API while providing a number of other benefits as well. At first glance, inotify seems to provide a complete solution for the task of creating an application that reliably monitors the state of a filesystem. However, we are about to see that this isn't quite the case. 

So, now we dig deeper into the use of inotify, looking at how the API can be used to create an application that monitors the dynamic state of a directory tree. Our goals are, on the one hand, to demonstrate how inotify can be used to robustly achieve this task and, on the other hand, to discover some of the challenges inherent in the task, and thereby discover some of the limitations of inotify. 

Our application, [`inotify_dtree.c`](<http://man7.org/tlpi/code/online/dist/inotify/inotify_dtree.c.html>), serves as a proof-of-concept, rather than performing any practical task. It is invoked with a command of the following form: 

```
./inotify_dtree <directory> <directory>...
```

The goal of the application is to robustly and accurately maintain an internal representation ("a cache") of the dynamically changing set of subdirectories under each of the directories named in its command line. This is somewhat similar to the task of programs such as GUI file managers, which must present the user with an accurate representation of (some part of) a filesystem tree. 

In addition to caching the state of the directory trees, the program also provides a command-line interface that allows the user to do such things as dumping the contents of the cache and triggering a consistency check of the cache against the current state of the directory trees. 

In order to keep the program to a manageable size, we make a number of simplifications: 

  * The application caches only the subdirectories under a directory tree. Other types of files are ignored. Caching other file types is reasonably straightforward, but we focus on monitoring subdirectories because they are the most challenging part of the problem. 
  * For each directory, the only information we cache is the directory name and the corresponding watch descriptor. In a real-world application, we might want to cache other information as well, such as file ownership, permissions, and last modification time. 
  * The cache data structure is simply a list (a dynamically allocated array of structures that is resized as needed). The list is searched linearly by pathname or watch descriptor. This allows a simple implementation, at the expense of efficiency. In a real-world application, we might instead elect to use a tree data structure whose form matches the directory tree. 

Even with these simplifications, we will see that programming with inotify presents a number of challenges. 

#### Challenge 1: recursively monitoring a directory tree

Inotify does not perform recursive monitoring of directories. If we monitor the directory `mydir`, then we will receive events for the directory itself and for its immediate children, but not for children of subdirectories. 

Therefore, in order to monitor an entire directory tree, we must create a watch for each subdirectory in the tree. This requires a recursive process whereby, for each directory, we create a watch and scan for subdirectories that should also be watched. The fun starts when we consider some of the possible race conditions. For example, suppose that we handle the tasks in the following order: 

  1. We scan `mydir`, adding watches for the subdirectories that we find. 
  2. We add a watch for `mydir`. 

Suppose that between these two steps, a new subdirectory, `mydir/new`, is created (or a directory is renamed into `mydir`). No watch will be created for that directory, since, on the one hand, it was created after the scan of subdirectories, and, on the other hand, it will not generate an event for inotify, because it was created before a watch was added for the directory `mydir`. 

The solution is to ensure that we perform the above steps in the reverse order, first creating a watch for the parent directory, and then scanning for subdirectories. If a new subdirectory is created between these two steps, then it might both generate an inotify event and be caught by the scanning step, which may result in a watch being created twice for the subdirectory. However, with careful programming, this is harmless, since repeated `inotify_add_watch()` calls for the same filesystem object will generate the same watch descriptor number. 

There are other race conditions to consider as well. For example, during the scanning step we may find a subdirectory and attempt to add a watch for it, only for the `inotify_add_watch()` call to fail because the directory has already been deleted or renamed. The correct approach in such cases is to silently ignore the error. 

The standard [`nftw()`](<http://man7.org/linux/man-pages/man3/nftw.3.html>) library function can assist with the task of creating a watch list for an entire directory tree. `nftw()` performs a traversal (preorder by default) of all of the files in a directory, calling a user-defined function once for each file. Among the arguments supplied to that function are the pathname of the file being visited and a [`stat`](<http://man7.org/linux/man-pages/man2/stat.2.html>) structure containing attributes of that file, including the file type. 

Here's an edited version of our function that is called by `nftw()` as it recursively traverses a directory tree that we want to add to the cache: 

```
static int
       traverseTree(const char *pathname, const struct stat *sb, int tflag,
                    struct FTW *ftwbuf)
       {
           int wd, slot, flags;
       
           if (! S_ISDIR(sb->st_mode))
               return 0;               /* Ignore nondirectory files */
       
           flags = IN_CREATE | IN_MOVED_FROM | IN_MOVED_TO | IN_DELETE_SELF;
           if (isRootDirPath(pathname))
               flags |= IN_MOVE_SELF;
       
           wd = inotify_add_watch(ifd, pathname, flags | IN_ONLYDIR);
```

A few points in the above code warrant explanation. The `S_ISDIR()` check allows us to skip non-directory files. We initialize the `inotify_add_watch()` `flags` argument with the set of flags that are relevant for monitoring the creation, renaming, and deletion of directories. 

Our `isRootDirPath()` function returns true if the given path is one of those supplied on the command line. For these directories, we are interested in events that rename the directories themselves, so we include the `IN_MOVE_SELF` flag. When an `IN_MOVE_SELF` event is processed, the program ceases monitoring the corresponding directory and all of its subdirectories. 

The `IN_ONLYDIR` flag causes `inotify_add_watch()` to check that the monitored file is a directory. Although we already checked (via the `S_ISDIR()` macro) that `pathname` is a directory, there is a chance that in the meantime the directory was deleted and a regular file with the same name was created. In other words, `IN_ONLYDIR` provides us with a race-free way of ensuring that a monitored object is a directory. 

The remainder of the function handles the consequences of the various race conditions described earlier, and then caches the new watch descriptor and pathname: 

```
if (wd == -1) {
       
               /* By the time we come to create a watch, the directory might
                  already have been deleted or renamed, in which case we'll get
                  an ENOENT error. In that case, we log the error, but
                  carry on execution. Other errors are unexpected, and if we
                  hit them, we give up. */
       
               logMessage(VB_BASIC, "inotify_add_watch: %s: %s\n",
                       pathname, strerror(errno));
               if (errno == ENOENT)
                   return 0;
               else
                   exit(EXIT_FAILURE);
           }
       
           if (findWatch(wd) > 0) {
       
               /* This watch descriptor is already in the cache;
                  nothing more to do. */
       
               logMessage(VB_BASIC, "WD %d already in cache (%s)\n", wd, pathname);
               return 0;
           }
       
           slot = addWatchToCache(wd, pathname);
       
           return 0;
       }
```

#### Challenge 2: handling overflow events

Queueing inotify events until they are read requires kernel memory. Therefore, the kernel imposes a per-queue limit on the number of events that can be queued. (The various inotify limits are exposed, and can, with privilege, be modified via `/proc` files, as described in the [inotify(7) man page](<http://man7.org/linux/man-pages/man7/inotify.7.html>).) When this queue limit is reached, the kernel adds an inotify overflow event (`IN_Q_OVERFLOW`) to the event queue and then _discards_ further events until the application has drained some events from the queue. 

This behavior has a number of implications. First of all, it means that when an overflow situation occurs, the application loses information about filesystem events. Or, to put things another way, inotify can't be used to produce a completely accurate log of filesystem activity. 

Overflow events also have implications for applications such as our example application. Once an overflow occurs, the cache state in our application is no longer synchronized with the filesystem state. Furthermore, even if we drain the inotify event queue so that future events can be queued, we are likely to encounter problems because some of those events will likely be inconsistent with the now-out-of-sync cache. In this circumstance, there is only one thing to do: close the inotify file descriptor, discard the existing cache, open a new inotify file descriptor and then repopulate the cache and re-create all of our watch descriptors by rescanning the monitored directories. This may take some time, if we are monitoring a large number of directories. These steps are handled in the [`reinitialize()` function](<http://man7.org/tlpi/code/online/dist/inotify/inotify_dtree.c.html#reinitialize>) of our example application. 

Although the inotify queue limit can be raised in order to make overflow events less likely, there nevertheless always remains a risk that the limit will be hit in an application. Therefore, all inotify applications must be written to properly handle overflow events. 

In passing, it's worth noting that unexpected corner cases or program bugs also may cause an application cache to become inconsistent with the state of the filesystem. A robust application should allow for this possibility: if such inconsistencies are detected, it is probably necessary to take the same steps as for the queue overflow case. 

#### Challenge 3: handling rename events

As noted in our previous article, inotify provides much better support than dnotify for monitoring rename events. When a filesystem object is renamed, two events are generated: an `IN_MOVED_FROM` event for the source directory from which the file is moved, and an `IN_MOVED_TO` for the target directory to which the file is moved. Those events are only generated for directories that are being watched, so an application will only get both if it is monitoring both the source and destination. The `name` fields of these two events provide the old and new names of the file. The two events will have the same (unique) value in their `cookie` field, which provides an application with the means to connect them. 

![\[Directory tree example for inotify\]](https://static.lwn.net/images/2014/inotify_dir_example.png) Rename events present a number of challenges. One of these is that, depending how we cache pathnames, a rename event may affect multiple items in the cache. For example, in our application, each item in the cache consists of a watch descriptor plus the complete pathname string corresponding to that watch descriptor. Suppose that we had a subtree containing the directories shown in the diagram to the right. 

If the directory `abc` was renamed to `def` under directory `xyz`, then three pathname strings inside our cache would need to be modified (so, for example, `mydir/abc/man` would become `mydir/xyz/def/man`). 

More sophisticated cache designs may mitigate or eliminate this problem. For example, if the cache employs a tree data structure that mirrors the structure of the directory tree, and each node contains just the filename (rather than the full pathname) of the corresponding filesystem object, then only a single cache object would be affected by a rename (it would be relocated inside the tree structure, and its pathname string would be modified). 

Rename events present other, more serious challenges, however. First, it's important to note that the `IN_MOVED_FROM` and `IN_MOVED_TO` events are generated only if, respectively, the source and target directories are each in the set being monitored by our inotify file descriptor. If only the target directory is in the set, then we will receive only an `IN_MOVED_TO` event. This can be dealt with straightforwardly: it is the same as the case of (recursively) adding a complete subtree to the set of watch subtrees. 

On the other hand, if only the source directory is in the monitored set, then only an `IN_MOVED_FROM` event is generated. Notionally, this can be treated like a deletion event: for all of the monitored objects in the renamed subtree, we destroy each of the corresponding objects in our cache and destroy each of the watch descriptors using `inotify_rm_watch()`. 

However, there is a problem: when we receive an `IN_MOVED_FROM` event, we do not yet know if there will be a following `IN_MOVED_TO` event. In other words, we do not yet know whether to treat this as a deletion event or as a "true rename" event. And it gets worse. If there are multiple processes generating events in the monitored trees, then there is no guarantee that, for a true rename event, the `IN_MOVED_FROM` and `IN_MOVED_TO` events will be returned as consecutive items in the buffer of events returned when reading from the inotify file descriptor; other events may be returned between the pair. The upshot is that when we encounter an `IN_MOVED_FROM` event, we need to do some measure of forward searching to see if there is a corresponding `IN_MOVED_TO` event. 

At this point, one might ask: rather than going to the effort of detecting whether there is an event pair, why not simply always treat `IN_MOVED_FROM` as a deletion event and `IN_MOVED_TO` as a creation event? This approach simplifies the programming in some respects, but it has some costs. If this is a true rename event, then we will waste effort in deleting items from our cache and destroying watch descriptors only to immediately repopulate the cache with the same filesystem objects and re-create a new set of watch descriptors. If the subtree that is being renamed is large, this may be rather expensive. Furthermore, when the watches are re-created they will use different watch descriptors. This means that there may be a series of events ahead in the inotify queue that contain watch descriptors that no longer exist. The simplest thing to do with these events is to discard them. 

Because of the problems described in the previous paragraph, it is usually worth going to some effort to detect whether there is an `IN_MOVED_TO` that matches the `IN_MOVED_FROM`. The only question is: how much effort do we make? It turns out that even on a busy filesystem where multiple processes are generating events, the two events are _almost_ always consecutive. So, a simple approach when encountering `IN_MOVED_FROM` is to check whether the very next event is `IN_MOVED_TO`, in which case this can be treated as a true rename, otherwise the `IN_MOVED_FROM` can be treated as a deletion event. Occasionally, we will fail to detect true renames because the events are not consecutive, in which case we fall back to the situation described in the preceding paragraph. 

However, even when performing this simplified detection of `IN_MOVED_FROM`+`IN_MOVED_TO` pairs, some care is required. A `read()` from an inotify descriptor will (given a sufficiently large buffer) return multiple events if they are available. This means that, generally, if there is a true rename operation, the `IN_MOVED_FROM` and `IN_MOVED_TO` events will be read in the same buffer. However, it may happen that only the `IN_MOVED_FROM` can fit at the end of the buffer returned by one `read()` and the `IN_MOVED_TO` event is fetched by the next `read()`. The application should deal with this possibility. 

It is also possible, of course, that there is no following `IN_MOVED_TO`, and indeed there might not (for the moment) be any more events at all. Furthermore, from the point of view of user space, the `IN_MOVED_FROM`+`IN_MOVED_TO` pair that is generated by a rename is not inserted atomically into the event queue: this means that having read an `IN_MOVED_FROM` from the queue, the following `IN_MOVED_TO` may not yet be available if we try to immediately fetch it with a nonblocking `read()`. Therefore, the second `read()` that tries to fetch the possibly following `IN_MOVED_TO` event must be performed with a (small) timeout. 

The [`processInotifyEvents()` function](<http://man7.org/tlpi/code/online/dist/inotify/inotify_dtree.c.html#processInotifyEvents>) in our example application provides one example of how to do this. It employs a 2-millisecond timeout for the second `read()`, which was found to be sufficient to catch around 99.8% of the true renames even when the monitored directory tree was subject to a high level of rename activity. 

For the purposes of testing, I created a test program, [`rand_dtree.c`](<http://man7.org/tlpi/code/online/dist/inotify/rand_dtree.c.html>), that randomly performs either subdirectory creations, subdirectory deletions, or subdirectory renames at a specified location. During testing, I simultaneously ran ten instances of the program in rename mode against a directory tree that contained approximately 200 subdirectories. 

A similar timeout technique is used in the [`inotify-kernel.c`](<https://git.gnome.org/browse/glib/tree/gio/inotify/inotify-kernel.c>) source file of the [GIO](<https://developer.gnome.org/gio/stable/>) library in [GLib](<https://wiki.gnome.org/Projects/GLib>), where a 0.5 millisecond timeout is used. In my tests, this was sufficient to detect true renames with only 95% accuracy. 

#### Challenge 4: using pathnames for notifications

When an event is generated for an object inside a monitored directory, inotify produces an event containing the name of the file. This is more information than given to us by dnotify. However, notification via pathname has some difficulties. The problem is that pathnames exist independently of filesystem objects. (One must also bear in mind that a filesystem object may have multiple pathnames, since a file can have multiple hard links.) Thus, by the time we read a notification that contains a filename, that filename may already have been deleted or renamed. Applications must be designed to handle this possibility.

![\[Two files that link to the same inode\]](https://static.lwn.net/images/2014/inotify_links.png)

Notification via pathname also produces some other oddities. Suppose that a filesystem object has two links, one inside a directory (`mydir/abc`) that we are monitoring for all possible events via inotify, and another link inside a directory (`mydir/xyz`) that we are not monitoring, as shown in the diagram to the right. 

If the inode 5139 is opened via the link `mydir/abc/x1`, then an event will be generated. On the other hand, if it is opened via `mydir/xyz/x2`, no event is generated. More generally, inotify events are generated for files only when they occur via pathnames that are in the monitored set. This behavior is not the consequence of a kernel limitation, but rather is a limitation of the notification method. In some circumstances, notifying the application about an event that occurred via one pathname using a different pathname would be confusing. For example, suppose that the pathname `mydir/xyz/x2` was deleted, how should an `IN_DELETE` event for `mydir/abc/x1` be interpreted? A similar question can be asked about rename events. To avoid these sorts of confusions, inotify only notifies events that occur via pathnames in the monitored set. 

#### Other limitations of the inotify API

In addition to the various challenges dealt with by our example application, there are several other limitations of the inotify API that an application may encounter: 

  * Event notifications do not include information about the process that generated the event. Nevertheless, it would sometimes be useful to have the process ID and user ID of the triggering process. One particularly notable case is that if a monitoring application itself touches files inside the monitored directories, then it may also generate events, but it has no way (for example, a PID) to distinguish those events from events generated by other processes. 
  * Inotify does not provide any gatekeeping functionality. That is to say, inotify only informs us about filesystem activity; it provides no way to block filesystem actions by other processes. This type of functionality is needed by antivirus software and some types of user-space file servers, for example. 
  * Inotify reports only events that a user-space program triggers through the filesystem API. This constitutes a fairly serious limitation of the API. For example, it means that inotify does not inform us of events on monitored objects via a remote filesystem (e.g. NFS) operation. Likewise, no events are generated for virtual filesystems such as `/proc`. Furthermore, events are not generated for file accesses and modifications that may occur via file mappings created using [`mmap()`](<http://man7.org/linux/man-pages/man2/mmap.2.html>). To discover changes that occur via these mechanisms, an application must revert to polling the filesystem using `stat()` and `readdir()`. 

Inotify improves on dnotify in many respects. However, a number of limitations of the API mean that reliably monitoring filesystem events still presents quite a challenge. In the next article in this series, we will consider the fanotify API, which, although more restricted than inotify in its range of applications, addresses some of the limitations of inotify.
