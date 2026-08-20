---
title: "A kernel unit-testing framework"
url: https://lwn.net/Articles/780985/
date: "March 1, 2019"
category: "Development tools-Testing"
author: "By Jonathan Corbet March 1, 2019"
---

By **Jonathan Corbet**  
March 1, 2019

For much of its history, the kernel has had little in the way of formal testing infrastructure. It is not entirely an exaggeration to say that testing is what the kernel community kept users around for. Over the years, though, that situation has improved; internal features like kselftest and services like the 0day testing system have increased our test coverage considerably. The story is unlikely to end there, though; the next addition to the kernel's testing arsenal may be a unit-testing framework called [KUnit](<https://lwn.net/ml/linux-kernel/20190214213729.21702-1-brendanhiggins@google.com/>). 

The KUnit patches, currently in their fourth revision, have been developed by Brendan Higgins at Google. The intent is to enable the easy and rapid testing of kernel components in isolation — unit testing, in other words. That distinguishes KUnit from kernel's [kselftest framework](<https://www.kernel.org/doc/html/latest/dev-tools/kselftest.html>) in a couple of significant ways. Kselftest is intended to verify that a given feature works in a running kernel; the tests run in user space and exercise the kernel that the system booted. They thus can be thought of as a sort of end-to-end test, ensuring that specific parts of the entire system are behaving as expected. These tests are important to have, but they do not necessarily test specific kernel subsystems in isolation from all of the others, and they require actually booting the kernel to be tested. 

KUnit, instead, is designed to run more focused tests, and they run inside the kernel itself. To make this easy to do in any setting, the framework makes use of user-mode Linux (UML) to actually run the tests. That may come as a surprise to those who think of UML as a dusty relic from before the kernel had proper virtualization support (its [home page](<http://user-mode-linux.sourceforge.net/>) is hosted on SourceForge and offers a bleeding-edge 2.6.24 kernel for download), but UML has been maintained over the years. It makes a good platform for something like KUnit without rebooting the host system or needing to set up virtualization. 

Using KUnit is a matter of writing a set of test cases to exercise the code in question and check the results. Each test case is a function with this signature: 

```
void test_case(struct kunit *test);
```

A test case function that returns normally is deemed to have succeeded; a failure can be indicated by a call to `KUNIT_FAIL()`: 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

```
void always_fails(struct kunit *test)
        {
            KUNIT_FAIL(test, "I'm so bad I always fail");
        }
```

One could thus write a test case with a bunch of `if` statements and `KUNIT_FAIL()` calls but, naturally, a set of helper macros exists to reduce the amount of boilerplate code required. For example: 

```
KUNIT_EXPECT_EQ(test, v1, v2);
```

will test `v1` and `v2` for equality and complain loudly if the test fails. A module testing low-level string handling might feature calls like: 

```
KUNIT_EXPECT_EQ(test, strcmp("foo", "foo"), 0);
        KUNIT_EXPECT_NE(test, strcmp("foo", "bar"), 0);
        /* ... */
```

As one would expect, there are a number of these macros for different kinds of tests; see [this page](<https://google.github.io/kunit-docs/third_party/kernel/docs/api/test.html>) for the full set. Note that a test case will continue after one of these "expectations" fails; that may not be desirable if a particular failure means that the remaining tests cannot be performed. For such cases, there is a mirror set of macros with `ASSERT` instead of `EXPECT` in their names; if an assertion fails, the rest of the test case (but not any other test cases) will be aborted. 

Once a set of test cases are written, they should be gathered together into an array with code like: 

```
static struct kunit_case test_cases[] = {
            KUNIT_CASE(my_case_1),
    	KUNIT_CASE(my_case_2),
    	{},
        };
```

This array is then packaged into a `kunit_module` structure this way: 

```
static struct kunit_module test_module = {
            .name = "My fabulous test module",
    	.init = my_test_init,
    	.exit = my_test_exit,
    	.test_cases = test_cases,
        };
        module_test(test_module);
```

The `init()` and `exit()` functions can be provided if they are needed to set up and clean up after the test cases. The resulting source file implements a special type of kernel module; the next step is to add it to the `Kconfig` file: 

```
config MY_TEST_MODULE
            bool "This is my test module and nobody else's"
    	depends on KUNIT
```

and to add an appropriate line to the makefile as well. Then, running the `kunit.py` program that comes with KUnit will build a UML kernel and boot it to run any tests that are enabled in the current kernel configuration. 

For more information, see [the detailed documentation](<https://google.github.io/kunit-docs/third_party/kernel/docs/>) written by Higgins. There is also [an extended example](<https://lwn.net/ml/linux-kernel/20190214213729.21702-16-brendanhiggins@google.com/>) provided with the patch set in the form of the conversion of the existing device-tree unit tests to the KUnit framework. There have been some comments on the details of how test cases are written, but the code would appear to be getting closer to ready for merging into the mainline. Then the kernel will have another tool in its testing toolbox. That is just the beginning of course; then somebody has to actually write the tests to go with it.
