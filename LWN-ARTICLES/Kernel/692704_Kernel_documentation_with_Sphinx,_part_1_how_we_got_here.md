---
title: "Kernel documentation with Sphinx, part 1: how we got here"
url: https://lwn.net/Articles/692704/
date: "July 6, 2016"
category: Documentation
author: "July 6, 2016 This article was contributed by Jani Nikula"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

July 6, 2016

This article was contributed by Jani Nikula

The last time LWN looked at [formatted kernel documentation](<https://lwn.net/Articles/671496/>) in January, it seemed like the merging of [AsciiDoc](<http://www.methods.co.nz/asciidoc/>) support for the kernel's structured source-code documentation ("kernel-doc") comments, was imminent. As Jonathan Corbet, in the capacity of the kernel documentation maintainer, wrote: ""A good-enough solution that exists now should not be held up overly long in the hopes that vague ideas for something else might turn into real, working code."" Sometimes, however, the threat that something not quite perfect might be merged is enough to motivate people to turn those vague ideas into something real.

In the end, [Sphinx](<http://www.sphinx-doc.org/>) and [reStructuredText](<http://docutils.sourceforge.net/rst.html>) are emerging as the future of Linux kernel documentation, with far more ambitious goals than the original AsciiDoc support patches ever had. With the bulk of the infrastructure work now merged to the `docs-next` branch headed for v4.8, it's a good time to reflect on how this came to happen and give an overview of the promising future of kernel documentation.

#### Background

The patches to support lightweight markup (initially using [Markdown](<https://daringfireball.net/projects/markdown/>), later AsciiDoc) in kernel-doc comments were borne out of a desire to write better documentation for the graphics subsystem. One of the goals was to enhance the in-source graphics subsystem internals documentation for two main reasons. First, if the documentation is next to the code it describes, the documentation has a better chance of being updated along with the code. Second, if the documentation can be written in plain text rather than DocBook XML, it's more likely to be written in the first place.

However, plain text proves to be just a little too plain when you venture beyond documenting functions and types, or if you want to generate pretty HTML or PDF documents out of it. Adding support for lightweight markup in the kernel-doc comments was the natural thing to do. However, bolting this to the existing DocBook toolchain turned out to be problematic.

As part of the documentation build process, the `scripts/kernel-doc` script extracts the structured comments and emits them in DocBook format. The `kernel-doc` script supports some structure but fairly little formatting. To fit into this scheme, the lightweight markup support patches caused `kernel-doc` to invoke an external conversion tool (initially `pandoc`, later `asciidoc`) on each documentation comment block to convert them from lightweight markup to DocBook. This was painfully slow.

Doing the conversion in `kernel-doc` kept the DocBook pipeline side of things mostly intact and oblivious to any markup, but it added another point of failure in the already long and fragile path from comments to HTML or PDF. Problems with markup and mismatches at each point of conversion made debugging challenging. The tools involved were not designed to work together and often disagreed about when and how markup should be applied.

It was clear that this was not the best solution, but at the time it worked and there was nothing else around.

#### AsciiDoc all-in, muddying the waters

Inspired by Jonathan's article and frustrated by the long documentation build times (we were testing the patches in the Intel graphics integration tree), I had the idea to make `kernel-doc` output AsciiDoc directly instead of DocBook. Converting the few structural features in the comments to AsciiDoc and just passing through the rest was trivial; `kernel-doc` already supported several output formats with reasonable abstractions. Like many ideas, this was the obvious thing to do—in retrospect. Suddenly, this opened the door to writing all of the high-level documents under `Documentation/DocBook` in AsciiDoc, embedding the documentation comments at that level, and getting rid of the DocBook template files altogether. This has massive benefits, and Jonathan soon followed up with a [proof-of-concept](<http://mid.gmane.org/1453764522-29030-1-git-send-email-corbet@lwn.net>) that did just that.

There was a little bit of excited buzz around this, with folks exploring, experimenting, and actually trying things out with document conversion. A number of conversations between interested developers at linux.conf.au seemed to further confirm that this was the path forward. But, just when it felt like people were settling on switching to doing everything in AsciiDoc, Jonathan [muddied the waters](<http://mid.gmane.org/20160213145317.247c63c7@lwn.net>) by taking a hard look at Sphinx as an alternative to AsciiDoc.

#### Sphinx vs. AsciiDoc

Sphinx is a documentation generator that uses reStructuredText as its markup language, extending and using [Docutils](<http://docutils.sourceforge.net/>) for parsing. Both Sphinx and Docutils were created in Python to document Python, but documenting C and C++ is also supported. Sphinx supports several output formats directly, such as HTML, LaTeX, and ePub, and supports PDF output via either LaTeX or the external [`rst2pdf`](<https://github.com/rst2pdf/rst2pdf>) tool.

The AsciiDoc format, on the other hand, is semantically equivalent to DocBook XML, with the DocBook constructs expressed in terms of lightweight markup. AsciiDoc is easier for humans to read and write than XML, but since it is designed to translate to DocBook, it fits nicely in front of an existing DocBook toolchain. The original Python AsciiDoc tool has been around for a long time, but has been superseded by a Ruby reimplementation called [Asciidoctor](<http://asciidoctor.org/>) in recent years. As far as the AsciiDoc markup goes, Asciidoctor was designed to be a drop-in replacement, but any extensions are implementation-specific due to the change in implementation language. Both tools support HTML and DocBook output natively; other output formats are generated from DocBook.

When comparing the markup formats for the purposes of kernel documentation, only the table support, which is much needed for the media subsystem documentation in particular, was clearly identified as being superior in AsciiDoc. Otherwise, the markup comparison was rather dispassionate; it really boiled down to the tools themselves and, to some extent, which languages the tools were written in. Indeed, the markups and tools were not independent choices. All the lightweight markups have their pros and cons.

Superficially, the implementation language of the tools shouldn't play any role in the decision. But it seemed that neither tool would work as-is, or at least we wouldn't be able to get their full potential without extending the tools ourselves. In the kernel tree, there are no tools written in Ruby, but there are plenty of tools written in Python. It was fairly easy to lean towards Sphinx in this regard.

If you are looking for flexibility, one great advantage of AsciiDoc is that it's so closely tied to DocBook. By switching to AsciiDoc, the kernel documentation could reuse the existing DocBook toolchain. The downside is that AsciiDoc would add another step in front of the already fragile DocBook toolchain. [Dan Allen of Asciidoctor said](<https://lkml.org/lkml/2016/3/8/184>): ""One of the key goals of the Asciidoctor project is to be able to directly produce a wide variety of outputs from the same source (without DocBook)."" However, this support isn't quite there yet.

The Asciidoctor project has a promising future. But Sphinx is stable, available now, and fits the needs of the kernel. [Grant Likely summed it up](<http://mid.gmane.org/CACxGe6ueYTEZjmVwV2P1JQea8b9Un5jLca6+MdUkAHOs2+jiMA@mail.gmail.com>) this way: ""Honestly, in the end I think we could make either tool do what is needed of it. However, my impression after trying to do a document that needs to have nice publishable output with both tools is that Sphinx is easier to work with, simpler to extend, better supported."" In the end, Jonathan's verdict was to go with Sphinx. The patches have been merged, and the first Sphinx-based documentation will appear in the 4.8 kernel. 

The [second and final part of this series](<https://lwn.net/Articles/692705/>) will look into how the kernel's new Sphinx-based toolchain works and how to write documentation using it.
