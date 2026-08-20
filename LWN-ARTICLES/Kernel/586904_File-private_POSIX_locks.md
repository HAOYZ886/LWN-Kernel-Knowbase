---
title: "File-private POSIX locks"
url: https://lwn.net/Articles/586904/
date: "February 19, 2014"
category: "Filesystems-POSIX locks"
author: "February 19, 2014 This article was contributed by Jeffrey T. Layton"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

February 19, 2014

This article was contributed by Jeffrey T. Layton

File locks are heavily used by applications such as databases and file servers. Whenever you have multiple programs accessing files at the same time there is always the potential for data corruption or other bugs unless that access is carefully synchronized. File locks solve that problem, but the existing implementation can be difficult to use, especially for multi-threaded programs. File-private POSIX locks are an attempt to take elements of both BSD-style and POSIX locks and combine them into a more threading-friendly file locking API. 

Multiple writers attempting to change a file at the same time can clobber each other's changes. In addition, an update to a file may need to be done in more than one place. If another thread of execution sees only part of the update, it may trigger bugs in the program. 

File locks are generally available in two flavors: read (also known as shared) locks and write (also known as exclusive) locks. Multiple read locks can be given out for a portion of a file, but only one write lock can be handed out at any given time, and only if no other read or write lock for that region has been set. While file locks on some operating systems are mandatory, on Unix-like systems locking is generally advisory. Advisory locks are like stoplights -- they only work if everyone pays attention to them. 

One of the primary mechanisms to handle file locking is the one specified by the POSIX standard. POSIX defines a file-locking standard that allows the ability to lock arbitrary byte ranges in a file for read or write. Unfortunately, they have a couple of serious problems that make them unsuitable for use by modern applications. 

#### The problems with POSIX locking

Whenever a program attempts to acquire a lock, that lock is either granted or denied based on whether there is already a conflicting lock set over the given range. If there is no conflicting lock present, the lock will be granted. If there is then it will be denied. 

Classic POSIX lock requests from the same process never conflict with one another. When a request for a lock comes in that would conflict with an existing lock that the process set previously, the kernel treats it as a request to modify the existing lock. Thus, classic POSIX locks are useless for synchronization between threads within the same process. Given the prevalence of threaded applications in modern computing, this renders POSIX locks fairly useless as a synchronization mechanism. 

More troublingly, the standard states that all locks held by a process are dropped any time the process closes _any_ file descriptor that corresponds to the locked file, even if those locks were made using a still-open file descriptor. It is this detail that catches most programmers by surprise as it requires that a program take extra care not to close a file descriptor until it is certain that locks held on that file are able to be dropped. 

That's not always a simple question to answer. If a program opens two different links of a hardlinked file, takes a lock on one file descriptor and then closes the other, that lock is implicitly dropped even though the file descriptor on which the lock was originally acquired remains open. 

This is a particular problem for applications that use complex libraries that do file access. It's common to have a library routine that opens a file, reads or writes to it, and then closes it again, without the calling application ever being aware that has occurred. If the application happens to be holding a lock on the file when that occurs, it can lose that lock without ever being aware of it. That sort of behavior can lead to silent data corruption, and loss of developer sanity. Jeremy Allison has an [excellent writeup of this problem](<http://www.samba.org/samba/news/articles/low_point/tale_two_stds_os2.html>) and of how such a broken standard came into being (see the section entitled "First Implementation Past the Post"). 

There is however, another competing (or complementary) file locking standard that has its roots in BSD Unix. These locks (which are manipulated via the `flock()` system call) have more sane semantics. Whereas POSIX locks are owned by the process, BSD locks are owned by the open file. If a process opens a file twice and tries to set exclusive locks on both, the second one will be denied. Thus, BSD locks are usable as a synchronization mechanism between threads as long as each thread has a separate opened file. Note that cloning a file descriptor with `dup()` is not sufficient since that simply takes a reference to the same opened file. 

Also, BSD locks are only released when the last reference to the open file on which they were acquired is closed. Thus if a program opens a file, takes a lock on it and then uses `dup()` to duplicate the file descriptor, the lock will only be released automatically when both file descriptors are closed. 

The only real problem with BSD locks is that they are whole-file locks. POSIX locks, on the other hand can operate on arbitrary byte ranges within a file. While whole-file locks are useful (and indeed, many applications just lock entire files even with POSIX locks), they are not sufficient for many cases. Applications such as databases need granular locking in order to allow for better parallelism. 

#### File-private POSIX locks

I will assert that what is needed is a new type of lock that is a hybrid of the two -- a byte-range lock that has BSD-like semantics for inheritance across `fork()` and on `close()`. Furthermore, since there is a large legacy codebase of programs that use "classic" POSIX locks, these new locks need to be aware of the classic locks so that programs using the new locks will interoperate correctly with those applications. 

Classic POSIX locks are manipulated using a set of command values passed to the `fcntl()` system call: 

  * `F_GETLK` \- test whether a lock is able to be applied 
  * `F_SETLK` \- attempt to set a lock. Return error if unable to do so 
  * `F_SETLKW` \- attempt to set a lock and block until able to do so 

These commands are accompanied by a pointer to a binary `struct flock` argument that looks something like this: 

```
struct flock {
            short int l_type;   /* Type of lock: F_RDLCK, F_WRLCK, or F_UNLCK.  */
            short int l_whence; /* Where `l_start' is relative to (like `lseek').  */
            off_t l_start;      /* Offset where the lock begins.  */
            off_t l_len;        /* Size of the locked area; zero means until EOF.  */
            pid_t l_pid;        /* Process holding the lock. (F_GETLK only) */
        };
```

Similarly, file-private POSIX locks are manipulated with a similar set of commands, this time appended with 'P': 

  * `F_GETLKP` \- test whether a lock is able to be applied 
  * `F_SETLKP` \- attempt to set a file-private lock 
  * `F_SETLKPW` \- attempt to set a file-private lock and block until able to do so 

The new commands should look very familiar to those used to working with classic POSIX locks and they take the same `struct flock` argument. The only real difference between file-private and classic POSIX locks is their "ownership". Classic POSIX locks are owned by the process whereas file-private POSIX locks are owned by the opened file. 

#### Using file-private POSIX locks

It is currently necessary to define the `_GNU_SOURCE` preprocessor macro in order to get the new command definitions as file-private locks are not yet part of POSIX. Using file-private locks is very similar to using classic POSIX locks. In many cases, one can simply replace the command value with the file-private equivalent. There are subtle differences, however. 

Since one of the most troublesome aspects of classic POSIX locks is their behavior on close, there should be no surprise that file-private locks behave differently. File-private locks are only released automatically when the last reference to the open file is released. 

It's tempting to then consider file-private locks to be "owned" by the file descriptor, but that's not technically true. If a file descriptor is cloned via `dup()`, the kernel will simply take an extra reference to the open file and assign it to a new slot in the open file descriptor table. File-private locks set on a cloned file descriptor will not conflict with locks set on the original file descriptor. The kernel will treat such a lock request as a request to modify the existing lock. Furthermore, file-private locks set using either file descriptor would only be released automatically once both file descriptors are closed, though one can always release a lock manually with an `F_UNLCK` request. 

Interaction across `fork()` is very similar. When `fork()` is called, the kernel takes an extra reference to each open file and assigns it to the same slot in the new process's file descriptor table. Locks set by either process on the same open file would not conflict with one another, and would only be automatically released once both processes have closed it. 

Classic and file-private locks will always conflict with one another, even when used in the same process and/or on the same file descriptor. I don't expect that many programs will mix the two, but given the pain that undefined behaviors can cause I think it's prudent to declare that explicitly. 

#### Whither `F_GETLK`?

`F_GETLK` would probably have been better named `F_TESTLK`. While it does technically fetch the existing status of a locked range, its real purpose is to allow one to test whether a given lock request could be set without actually setting it. If there happens to be a conflicting lock already set within that range, the kernel will overwrite the `struct flock` with information about that lock and set `l_pid` to the value of the process that owns that lock. 

The `l_pid` field is a bit of a dilemma for file-private locks. File-private locks are not owned by processes. A file descriptor could have been inherited across a `fork()`, so the `l_pid` value is somewhat meaningless if the conflicting lock is a file-private one. Still, when a program using classic POSIX locks calls `F_GETLK`, we do need to put something in the `l_pid` field. That something is `-1`. 

This precedent comes from BSD. On Linux, POSIX and BSD locks operate in a completely different namespace. On BSD, however, they operate in the same namespace and, thus, will conflict with each other. If a program holds a BSD lock on a file, and another does a `F_GETLK` request against it, the BSD kernel will set the `l_pid` to `-1`. Since portable programs already need to contend with such behavior, using the same behavior for file-private locks seems like a reasonable choice. 

#### Using file-private locks with threads

It's common for modern applications to use a threading model instead of forking to create a new thread of execution. This is problematic with classic POSIX locks. They are associated with a process, so locks acquired by threads within the same process cannot conflict. 

With file-private locks however we can circumvent that restriction by giving each thread its own open file. Here's an example (note that I've left out proper error handling for the sake of brevity): 

```
#define _GNU_SOURCE
        #include <stdio.h>
        #include <sys/types.h>
        #include <sys/stat.h>
        #include <unistd.h>
        #include <fcntl.h>
        #include <pthread.h>
    
        #define FILENAME    "/tmp/foo"
        #define NUM_THREADS 3
        #define ITERATIONS  5
    
        void *
        thread_start(void *arg)
        {
            int i, fd, len;
            long tid = (long)arg;
            char buf[256];
            struct flock lck = {
                .l_whence = SEEK_SET,
                .l_start  = 0,
                .l_len    = 1,
            };
    
            fd = open(FILENAME, O_RDWR|O_CREAT, 0666);
    
            for (i = 0; i < ITERATIONS; i++) {
                lck.l_type = F_WRLCK;
                fcntl(fd, F_SETLKPW, &lck);
    
                len = sprintf(buf, "%d: tid=%ld fd=%d\n", i, tid, fd);
    
                lseek(fd, 0, SEEK_END);
                write(fd, buf, len);
                fsync(fd);
    
                lck.l_type = F_UNLCK;
                fcntl(fd, F_SETLKP, &lck);
    
                usleep(1);
            } 
            pthread_exit(NULL);
        }
    
        int
        main(int argc, char **argv)
        {
            long i;
            pthread_t threads[NUM_THREADS];
    
            truncate(FILENAME, 0);
    
            for (i = 0; i < NUM_THREADS; i++)
                pthread_create(&threads[i], NULL, thread_start, (void *)i);
    
            pthread_exit(NULL);
            return 0;
        }
```

This example spawns three threads and has each do five iterations of appending to a file. Access to that file is serialized via file-private locks. If we compile and run the above program, we end up with `/tmp/foo` that has 15 lines in it. 

If we, however, were to replace the `F_SETLKP` and `F_SETLKPW` commands with their classic POSIX lock equivalents, the locking essentially becomes a noop since it is all done within the context of the same process. That leads to data corruption (missing lines) as some threads race in and overwrite the data from others. 

#### Conclusion

File-private locks can solve many of the problems experienced with classic POSIX locks, but programmers intending to use them should take heed of the differences. 

Developers from several projects including Samba, NFS Ganesha, SQLite, and OpenJDK have expressed interest in using file-private locks since they help simplify the code for many of their use cases, and help eliminate data corruption issues that can occur when files are closed. 

The kernel patchset is available in the linux-kernel [ mailing list posting](<http://thread.gmane.org/gmane.network.samba.internals/74464>), or via the [linux-next branch of my git tree](<http://git.samba.org/?p=jlayton/linux.git;a=shortlog;h=refs/heads/linux-next>). I plan to keep that branch updated with the latest version until it gets merged into mainline kernels. The kernel patches are currently being pulled into the linux-next tree as well. Anyone using linux-next kernels can use these now. There is also a (fairly trivial) [GNU C library (glibc) patchset](<http://git.samba.org/?p=jlayton/glibc.git;a=shortlog;h=refs/heads/file-private>) which implements the definitions needed to access these locks. 

I'm currently aiming to have the kernel patches merged into mainline in v3.15, and glibc patches to add the new command definitions along with an update to the glibc manual should hopefully be merged soon afterward. Assuming that these patches are merged, I also intend to submit an update to the POSIX specification to make these a formal part of POSIX. I have opened a [request](<http://austingroupbugs.net/view.php?id=768>) to have it considered. There have already been some helpful suggestions and The Austin Group (who oversees POSIX) seems receptive to the general idea. 

Hopefully other operating systems will follow suit and implement these as well so that programmers dealing with those platforms can reap the same benefits.
