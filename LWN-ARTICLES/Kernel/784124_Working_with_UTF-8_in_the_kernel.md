---
title: "Working with UTF-8 in the kernel"
url: https://lwn.net/Articles/784124/
date: "March 28, 2019"
category: Character encoding
author: "By Jonathan Corbet March 28, 2019"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
March 28, 2019

In the real world, text is expressed in many languages using a wide variety of character sets; those character sets can be encoded in a lot of different ways. In the kernel, life has always been simpler; file names and other string data are just opaque streams of bytes. In the few cases where the kernel must interpret text, nothing more than ASCII is required. The [proposed addition of case-insensitive file-name lookups](<https://lwn.net/Articles/784041/>) to the ext4 filesystem changes things, though; now some kernel code must deal with the full complexity of Unicode. A look at the API being provided to handle encodings illustrates nicely just how complicated this task is. 

The Unicode standard, of course, defines "code points"; to a first approximation, each code point represents a specific character in a specific language group. How those code points are represented in a stream of bytes — the encoding — is a separate question. Dealing with encodings has challenges of its own, but over the years the [UTF-8](<https://en.wikipedia.org/wiki/UTF-8>) encoding has emerged as the preferred way of representing code points in many settings. UTF-8 has the advantages of representing the entire Unicode space while being compatible with ASCII — a valid ASCII string is also valid UTF-8. The developers implementing case independence in the kernel decided to limit it to the UTF-8 encoding, presumably in the hope of solving the problem without going entirely insane. 

The API that resulted has two layers: a relatively straightforward set of higher-level operations and the primitives that are used to implement them. We'll start at the top and work our way down. 

#### The high-level UTF-8 API

At a high level, the operations that will be needed can be described fairly simply: validate a string, normalize a string, and compare two strings (perhaps with case folding). There is, though, a catch: the Unicode standard comes in multiple versions ([version 12.0.0](<https://unicode.org/versions/Unicode12.0.0/>) was released in early March), and each version is different. The [normalization](<https://en.wikipedia.org/wiki/Unicode_equivalence#Normalization>) and case-folding rules can change between versions, and not all code points exist in all versions. So, before any of the other operations can be used, a "map" must be loaded for the Unicode version of interest: 

```
struct unicode_map *utf8_load(const char *version);
```

The given `version` number can be `NULL`, in which case the latest supported version will be used and a warning will be emitted. In the ext4 implementation, the Unicode version used with any given filesystem is stored in the superblock. The latest version can be explicitly requested by obtaining its name from `utf8version_latest()`, which takes no parameters. The return value from `utf8_load()` is a map pointer that can be used with other operations, or an error-pointer value if something goes wrong. The returned pointer should be freed with `utf8_unload()` when it is no longer needed. 

UTF-8 strings are represented in this interface using the [`qstr` structure](<https://elixir.bootlin.com/linux/v5.0/source/include/linux/dcache.h#L40>) defined in `<linux/dcache.h>`. That reveals an apparent assumption that the use of this API will be limited to filesystem code; that is true for now, but could change in the future. 

The single-string operations provided are: 

```
int utf8_validate(const struct unicode_map *um, const struct qstr *str);
        int utf8_normalize(const struct unicode_map *um, const struct qstr *str,
    		       unsigned char *dest, size_t dlen);
        int utf8_casefold(const struct unicode_map *um, const struct qstr *str,
    		      unsigned char *dest, size_t dlen);
```

All of the functions require the map pointer (`um`) and the string to be operated upon (`str`). `utf_validate()` returns zero if `str` is a valid UTF-8 string, non-zero otherwise. A call to `utf8_normalize()` will store a normalized version of `str` in `dest` and return the length of the result; `utf8_casefold()` does case folding as well as normalization. Both functions will return `-EINVAL` if the input string is invalid or if the result would be longer than `dlen`. 

Comparisons are done with: 

```
int utf8_strncmp(const struct unicode_map *um,
    		     const struct qstr *s1, const struct qstr *s2);
        int utf8_strncasecmp(const struct unicode_map *um,
    		     const struct qstr *s1, const struct qstr *s2);
```

Both functions will compare the normalized versions of `s1` and `s2`; `utf8_strncasecmp()` will do a case-independent comparison. The return value is zero if the strings are the same, one if they differ, and `-EINVAL` for errors. These functions only test for equality; there is no "greater than" or "less than" testing. 

#### Moving down

Normalization and case folding require the kernel to gain detailed knowledge of the entire Unicode code point space. There are a lot of rules, and there are multiple ways of representing many code points. The good news is that these rules are [packaged](<https://www.unicode.org/reports/tr44/#UCD_Files>), in machine-readable form, with the Unicode standard itself. The bad news is that they take up several megabytes of space. 

The UTF-8 patches incorporate these rules by processing the provided files into a data structure in a C header file. A fair amount of space is then regained by removing the information for decomposing Hangul (Korean) code points into their base components, since this is a task that can be done algorithmically as well. There is still a lot of data that has to go into kernel space, though, and it's naturally different for each version of Unicode. 

The first step for code wanting to use the lower-level API is to get a pointer to this database for the Unicode version in use. That is done with one of: 

```
struct utf8data *utf8nfdi(unsigned int maxage);
        struct utf8data *utf8nfdicf(unsigned int maxage);
```

Here, `maxage` is the version number of interest, encoded in an integer form from the major, minor, and revision numbers using the `UNICODE_AGE()` macro. If only normalization is needed, `utf8nfdi()` should be called; use `utf8nfdicf()` if case folding is also required. The return value will be an opaque pointer, or `NULL` if the given version cannot be supported. 

Next, a cursor should be set up to track progress working through the string of interest: 

```
int utf8cursor(struct utf8cursor *cursor, const struct utf8data *data,
    	           const char *s);
        int utf8ncursor(struct utf8cursor *cursor, const struct utf8data *data,
    		    const char *s, size_t len);
```

The `cursor` structure must be provided by the caller, but is otherwise opaque; `data` is the database pointer obtained above. If the length of the string (in bytes) is known, `utf8ncursor()` should be used; `utf8cursor()` can be used when the length is not known but the string is null-terminated. These functions return zero on success, nonzero otherwise. 

Working through the string is then accomplished by repeated calls to: 

```
int utf8byte(struct utf8cursor *u8c);
```

This function will return the next byte in the normalized (and possibly case-folded) string, or zero at the end. UTF-8-encoded code points can take more than one byte, of course, so individual bytes do not, on their own, represent code points. Due to decomposition, the return string may be longer than the one passed in. 

As an example of how these pieces fit together, here is the full implementation of `utf8_strncasecmp()`: 

```
int utf8_strncasecmp(const struct unicode_map *um,
    		         const struct qstr *s1, const struct qstr *s2)
        {
    	const struct utf8data *data = utf8nfdicf(um->version);
    	struct utf8cursor cur1, cur2;
    	int c1, c2;
    
    	if (utf8ncursor(&cur1, data, s1->name, s1->len) < 0)
    	    return -EINVAL;
    
    	if (utf8ncursor(&cur2, data, s2->name, s2->len) < 0)
    	    return -EINVAL;
    
    	do {
    	    c1 = utf8byte(&cur1);
    	    c2 = utf8byte(&cur2);
    
    	    if (c1 < 0 || c2 < 0)
    		return -EINVAL;
    	    if (c1 != c2)
    		return 1;
    	} while (c1);
    
    	return 0;
        }
```

There are other functions in the low-level API for testing validity, getting the length of strings, and so on, but the above captures the core of it. Those interested in the details can find them in [this patch](<https://lwn.net/ml/linux-fsdevel/20190318202745.5200-4-krisman@collabora.com/>). 

That is quite a bit of complexity when one considers that it is all there just to compare strings; we are now far removed from the simple string functions found in Kernighan & Ritchie. But that, it seems, is the world that we live in now. At least we get some nice emoji for all of that complexity 👍.
