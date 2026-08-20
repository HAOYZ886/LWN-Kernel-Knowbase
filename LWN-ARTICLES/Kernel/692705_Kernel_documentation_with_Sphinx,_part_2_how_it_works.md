---
title: "Kernel documentation with Sphinx, part 2: how it works"
url: https://lwn.net/Articles/692705/
date: "July 13, 2016"
category: Documentation
author: "July 13, 2016 This article was contributed by Jani Nikula"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

July 13, 2016

This article was contributed by Jani Nikula

The kernel's documentation tree is going through a fundamental transition toward the use of [Sphinx](<http://www.sphinx-doc.org/>) and [reStructuredText](<http://docutils.sourceforge.net/rst.html>) for the production of formatted documents. The [first article in this series](<https://lwn.net/Articles/692704/>) discussed the path the development community took as it made the decision to go with Sphinx. This article, which concludes the series, covers the mechanics of the new documentation system and how to add to it. 

From the casual developer's perspective, building the documentation hasn't changed much. In the 4.8 kernel and beyond, the usual "`make htmldocs`" and "`make pdfdocs`" commands will invoke both Sphinx to build the documentation written in reStructuredText and the old toolchain to build documentation still in DocBook format. One will need to have Sphinx installed, obviously. For prettier HTML, the Read the Docs Sphinx theme (`sphinx_rtd_theme`) will be used if available. For PDF output, the [rst2pdf](<http://rst2pdf.ralsina.me/stories/index.html>) package is also needed. All of them are readily available in stable distributions.

The documentation build for Sphinx uses a dedicated `Documentation/Makefile.sphinx`, with `Documentation/conf.py` for configuration. The generated files are placed under `Documentation/output` in format-specific subdirectories. Currently, there is not much documentation that is actually built from reStructuredText, but the graphics documentation as well as documentation about the Sphinx-based system itself will be ready in time for v4.8. Over time, the plan is that all DocBook documents will be converted to reStructuredText, and we can finally say goodbye to DocBook.

From the perspective of the build system, Sphinx is pleasantly simple compared to the DocBook toolchain. It handles dependencies within documents by itself, storing intermediate data in the output directory. This allows the build system to work without knowledge of how the input and output files map to each other.

#### Writing documentation

Adding new documentation to the Sphinx build can be as simple as following these steps:

  1. Add a new reStructuredText file somewhere under `Documentation` with a `.rst` extension.
  2. Refer to it from the main index file `Documentation/index.rst`.

For now, converting existing plain-text and DocBook files to reStructuredText is more likely to happen than adding new files altogether. Because the current plain-text files don't follow any markup, they need to be manually converted; happily, by design, plain text is not too far from lightweight markup. We expect that some of the thousands of plain-text files will be converted to reStructuredText over time, but there is no real pressure to do so, and not everything needs to be part of the documentation build. 

The DocBook conversion is more interesting. There's a "cheesy conversion script" from Jonathan Corbet in `Documentation/sphinx/tmplcvt` that uses `pandoc` with some `sed` pre- and post-processing. Markus Heiser has been working on some more advanced conversion scripts. The DocBook templates should be converted primarily by their authors or maintainers to ensure they remain sensible and no errors creep in while converting. The conversion is a one-time effort anyway, so after a point, polishing the scripts is wasted effort. ([Here's a sample](<http://kernel.readthedocs.io/en/sphinx-samples/>) of the results of some of the DocBook files converted using the cheesy script, with no manual editing on top.)

Once converted, the DocBook templates are to be placed alongside other documentation under `Documentation` instead of in a silo under `Documentation/DocBook`. That directory, along with the entire DocBook toolchain, is slated to be removed once all the documents therein have been converted. Even developers who couldn't care less about producing pretty documents can benefit from converting the DocBook templates to reStructuredText because grepping and reading reStructuredText is much easier than the angle-bracketed mess that is DocBook.

Eventually we'll need to have more structure than just shoving everything directly in the main index. In particular, the PDF output needs to be split into several documents. This can be done using a configuration option in `Documentation/conf.py` as more documents are added. For starters, however, keeping things simple seems like the way to go.

#### Formatted kernel-doc comments

When building documentation using Sphinx, the kernel-doc comments are now treated as reStructuredText. Some hiccups will inevitably follow, as the comments were not written with reStructuredText in mind, but mostly it just works.

The `kernel-doc` script parses the formatted comments at the high level (function and structure names, parameter and member descriptions, and so on), generates appropriate Sphinx [C Domain](<http://www.sphinx-doc.org/en/stable/domains.html>) anchors for them, filters the comments for highlights and cross-references, and otherwise passes the rest through as-is. The filters convert `function_name()` and references to structure types (using the `&struct struct_name` convention) to proper C Domain cross-references, and there are other highlights as well.

A dedicated Sphinx directive extension incorporates kernel-doc comments from source files into the document. Internally, the extension invokes `kernel-doc` to do the job and informs Sphinx about the document dependencies on source files. The extension makes it possible to include kernel-doc comments with any reStructuredText file under `Documentation` with no special handling or dependency tracking in the makefiles.

For example, to include the documentation for all the functions exported using `EXPORT_SYMBOL()` from `bitmap.c`, you'd write the following:

```
.. kernel-doc:: lib/bitmap.c
           :export:
```

To include [an overview documentation section from `intel_audio.c`](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/drivers/gpu/drm/i915/intel_audio.c#n33>):

```
.. kernel-doc:: drivers/gpu/drm/i915/intel_audio.c
           :doc: High Definition Audio over HDMI and Display Port
```

The `DOC:` title given in the source code acts as an identifier for the section. There are also ways to include documentation for specific functions or types.

Daniel Vetter's contributions enable the kernel-doc extension to feed the source code file and line number of each documentation comment to Sphinx to enhance diagnostic messages on reStructuredText errors. This will come in handy when fixing the hiccups mentioned earlier.

#### Future work

There has been some talk (and even [code](<https://lwn.net/Articles/692710/>) from Markus) to convert the `kernel-doc` script from Perl to Python and perhaps to run it directly in the Sphinx extension. It is not clear, however, whether it's worth converting a homebrew C parser with two decades of field testing from one language to another just for the sake of it. Perhaps a compiler plugin would be a better idea.

As noted earlier, the media documentation in particular needs better syntax for tables. To this end, Markus has written [a Sphinx extension to support row and column spans](<https://lwn.net/Articles/692711/>), among other things, in tables. This work, too, looks set to go into 4.8; it is a dependency for converting the media documents.

But, on a positive note, most of the work discussed in this article has been merged. We'll be seeing more documentation patches that convert files to reStructuredText, as well as fixing and improving kernel-doc comments in source. Hopefully the changes will improve the state of the kernel documentation as a whole, and will move us one step closer to the documentation maintainer's vision as expressed during [a linux.conf.au talk](<https://youtu.be/gsJXf6oSbAE?t=37m>), "If we do this, we end up with, some years from now, this beautiful, integrated documentation tree, that covers things in a comprehensive way, where you can find what you want, looks pretty when you look at it. It's a nice vision, I hear angels singing when I think about it and so on, it's where I want to go."

[_Jani Nikula is employed by Intel to work on Linux graphics, and is also the author of most of the Sphinx work, with contributions from Daniel Vetter and Jonathan Corbet._]
