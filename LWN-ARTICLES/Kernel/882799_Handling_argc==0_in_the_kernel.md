---
title: "Handling argc==0 in the kernel"
url: https://lwn.net/Articles/882799/
date: "January 28, 2022"
category: "Security-Vulnerabilities; System calls-execve"
author: "By Jonathan Corbet January 28, 2022"
---

By **Jonathan Corbet**  
January 28, 2022

By now, most readers are likely to be familiar with the [Polkit vulnerability](<https://lwn.net/Articles/882609/>) known as CVE-2021-4034. The fix for Polkit is relatively straightforward and is being rolled out across the net. The root of this problem, though, lies in a misunderstanding about how programs are run on Unix-like systems. This problem is highly likely to exist in other programs, so it would be nice to find a more general solution. The best place to address this issue may be in the kernel, but properly working around this misunderstanding without causing regressions is not an easy task. 

#### I'd like to have an argument, please

Most developers are familiar with the prototype of the main program as expressed in the C language: 

```
int main(int argc, char *argv[], char *envp[]);
```

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

The program is invoked with its command-line arguments in `argv` and the environment in `envp`; both are pointers to null-terminated arrays of `char *` strings. The number of non-null entries in `argv` is stored in `argc`. This API is a user-space creation, though; what happens when the kernel first runs a program is a little bit different: on Linux, that program is passed a single pointer to the `argv` array. The `envp` array begins immediately after the `NULL` value that terminates `argv`. Thus, in a C program, the following statement will be true on entry to `main()`: 

```
envp == argv + argc + 1
```

By convention, `argv[0]` is the name of the program that is being executed, and many programs rely on that convention. As it happens, though, this convention is exactly that: a convention, but not a guarantee. The actual contents of `argv` are entirely under the control of whoever calls [`execve()`](<https://man7.org/linux/man-pages/man2/execve.2.html>) to run the program in the first place, and that caller is not required to put the program name in `argv[0]`. 

Indeed, the caller is not required to provide `argv[0]` at all. If the `argv` array passed to `execve()` is empty (or the `argv` pointer is simply `NULL`), the first pointer in the new program's `argv` array will be `NULL`, and the `envp` array will start immediately thereafter. Unfortunately, Polkit (or, more specifically, the setuid `pkexec` utility) "knew" that `argv[0]` would always be present, so it performed its argument processing by iterating over the `argv` array starting at `argv[1]`. If there are no arguments at all, `argv[1]` is the same as `envp`, so `pkexec` was iterating through its environment variables instead. Throw in some in-place argument modification (`pkexec` overwrites its `argv` array), and `pkexec` could be convinced to rewrite its environment variables, thus bypassing the sanitizing of those variables done for setuid programs. At that point, the game was over. 

This problem is not new, and neither is awareness of it. Ryan Mallon [wrote about it in 2013](<https://ryiron.wordpress.com/2013/12/16/argv-silliness/>), noting that ""it does allow for some amusing behaviour from setuid binaries""; he also evidently [sent a Polkit patch](<https://twitter.com/ryiron/status/1486207465918468097>) to address it, but that patch was never applied. Even further back, in 2007, Michael Kerrisk [reported the kernel's behavior as a bug](<https://bugzilla.kernel.org/show_bug.cgi?id=8408>), but the report was closed with little discussion. So the problem persisted, culminating in the vulnerability administrators are scrambling to patch now. 

#### Toward a more general fix

Fixing this issue is a simple matter of making `pkexec` check that `argc` is at least one. But there are surely other programs out there containing similar assumptions. Given the strength of the `argv[0]` convention, it is natural to ask whether it makes sense to allow programs to be run with an empty `argv` array at all. Perhaps it doesn't, but the current API has a lot of history and cannot be changed without a lot of thought. 

Ariadne Conill [started the linux-kernel discussion](<https://lwn.net/ml/linux-kernel/20220126043947.10058-1-ariadne@dereferenced.org/>) with a patch that would simply disallow calls to `execve()` without at least one `argv` entry. Offending callers would get an `EFAULT` return value instead. This would solve the problem by providing a guarantee that `argv` would not be empty, but at the potential cost of introducing problems of its own. One is that, as Kees Cook [discovered](<https://lwn.net/ml/linux-kernel/39480927-B17F-4573-B335-7FCFD81AB997@chromium.org/>), there is actually a fair amount of code out there that calls `execve()` with an empty `argv` array. Conill [wrote those off](<https://lwn.net/ml/linux-kernel/44b4472d-1d50-c43f-dbb1-953532339fb4@dereferenced.org/>) as ""lazily-written test cases which should be fixed"", but regressions in lazily-written test cases are still regressions. Also, as [Heikki Kallasjoki](<https://lwn.net/ml/linux-kernel/YfE%2FowUY+gVnn2b%2F@selene.zem.fi/>) and [Rich Felker](<https://lwn.net/ml/linux-kernel/20220126132729.GA7942@brightrain.aerifal.cx/>) both pointed out, an empty `argv` array is actually allowed by the POSIX standard. 

Felker also suggested an alternative with less potential for regressions: only enforce a non-empty `argv` at privilege boundaries — when `execve()` is being called to run a setuid program, in other words. Cook [said](<https://lwn.net/ml/linux-kernel/202201261210.E0E7EB83@keescook/>) that he would rather avoid taking the privilege boundary into account if possible, though. He proposed [a different solution](<https://lwn.net/ml/linux-kernel/20220126175747.3270945-1-keescook@chromium.org/>) to the problem: inject an extra null pointer at the end of an empty `argv` array so that even code that tries to skip `argv[0]` will notice that there is nothing there. This solution will not work either, as it turns out: the ABI promise is that `envp` starts immediately after `argv`, and the extra `NULL` breaks that promise. There are evidently programs that rely on that layout and would break if it were changed. 

Yet another approach, first [suggested](<https://lwn.net/ml/linux-kernel/877damwi2u.fsf@email.froward.int.ebiederm.org/>) by Eric Biederman, would be to replace an empty `argv` with one containing a single pointer to an empty string. This proposal had some support (though nobody has implemented it as of this writing), but also provoked some worries of its own. Perhaps there are programs out there that will respond badly to an empty-string argument, or which do something special when `argc` is zero. Changing the number of arguments passed to the program run by `execve()` just looks like it could create surprises. 

Cook eventually [summarized the situation](<https://lwn.net/ml/linux-kernel/202201261440.0C13601104@keescook/>) this way: 

> Given the code we've found that depends on NULL argv, I think we likely can't make the change outright, so we're down this weird rabbit hole of trying to reject what we can and create work-around behaviors for the cases that currently exist. 

That notwithstanding, he then went on to express a preference for the initial change (simply disallowing a zero-argument `execve()` call anyway, albeit with an `EINVAL` return rather than `EFAULT`), with a suggestion to fix a [Valgrind](<https://valgrind.org/>) test that is already known to break with that restriction. Conill responded [with a new version of the original patch](<https://lwn.net/ml/linux-kernel/20220127000724.15106-1-ariadne@dereferenced.org/>); this time it emits a warning before failing an empty-`argv` call to `execve()` with `EINVAL`. Cook [acked the patch](<https://lwn.net/ml/linux-kernel/202201262119.105FA8BCA9@keescook/>), saying: ""Let's do it and see what breaks""; Biederman [concurred](<https://lwn.net/ml/linux-kernel/87r18tt952.fsf@email.froward.int.ebiederm.org/>): ""Especially since you are signing up to help fix the tests"". 

That is where the discussion stands as of this writing, but it is far from clear that this is how the problem will eventually be addressed. This patch has, crucially, not yet survived its first encounter with Linus Torvalds, who may take a dim view of its potential for regressions. It _is_ an ABI change, after all, and there may well be code out there that responds badly to it, though the fact that BSD systems already prohibit an empty `argv` will, with luck, have already shaken out most of those. Should unfortunate regressions arise anyway, a different solution will almost certainly need to be found.
