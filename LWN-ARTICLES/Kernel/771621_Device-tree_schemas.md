---
title: "Device-tree schemas"
url: https://lwn.net/Articles/771621/
date: "November 14, 2018"
category: Device tree
author: "November 14, 2018 This article was contributed by Marta Rybczyńska"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

November 14, 2018

This article was contributed by Marta Rybczyńska

Device trees have become ubiquitous in recent years as a way of describing the hardware layout of non-discoverable systems, such as many ARM-based devices. The device-tree _bindings_ define how a particular piece of hardware is described in a device tree. Drivers then implement those bindings. The device-tree documentation shows how to use the bindings to describe systems: which properties are available and which values they may have. In theory, the bindings, drivers and documentation should be consistent with each other. In practice, they are often not consistent and, even when they are, using those bindings correctly in actual device trees is not a trivial task. As a result, developers have been considering formal validation for device-tree files for years. Recently, Rob Herring [proposed](<https://lwn.net/ml/linux-kernel/20181005165848.3474-1-robh@kernel.org/#t>) a move to a more structured documentation format for device-tree bindings using [JSON Schema](<http://json-schema.org/>) to allow automated validation.

Device-tree documentation today is free-form text with some defined structure and optional examples (like the generic GPIO clock multiplexer documentation in [`gpio-mux-clock.txt`](<https://elixir.bootlin.com/linux/latest/source/Documentation/devicetree/bindings/clock/gpio-mux-clock.txt>)). For new bindings, the review process is entirely manual and depends on the reviewers to find typos and errors that an automated system might be expected to catch. No tool exists to check whether any given device-tree file conforms to the binding documentation. In addition, the bindings documentation files sometimes either duplicate information that is also contained elsewhere or are missing information that is necessary to validate a device-tree file.

Numerous proposals have been made in the past to address the validation of device trees. One went as far as [using YAML as a source format](<https://lwn.net/Articles/730217/>) for device-tree files. Herring does not go that far; instead he proposes to convert only the documentation files, using JSON Schema for the schema vocabulary, while leaving the device-tree format itself unchanged. He [explained the choice](<https://lwn.net/ml/linux-kernel/20181005165848.3474-1-robh@kernel.org/>) in the submission cover letter: ""the language has a defined specification, maps well to DT data, and there are numerous existing tools which can be leveraged"". He prefers to use a YAML subset because it is generally considered more human-readable and allows certain additions, including comments. This solution also takes advantage of existing technology and libraries and avoids inventing a new language. The goal was to allow validating device-tree files at build time and verifying the correctness of the documentation. In addition, error and warning messages can be made more meaningful.

#### Documentation format and validation process

The device-tree documentation in the new format becomes a structured file, a schema. It is written in YAML using a JSON-compatible format. A schema should include the information necessary to validate a device-tree file. The schema file has a number of sections (mandatory and optional), including: 

  * `$id` -- gives a unique schema identifier 
  * `$schema` -- gives the identifier of the meta-schema this schema follows 
  * `title` -- provides the schema title 
  * `description` -- includes a multi-line description of the binding 
  * `maintainers` -- a list of email addresses of all maintainers 
  * `properties` -- the sections with the dictionary of property descriptions (this is a big part, of course) 
  * `requires` -- lists the mandatory properties 
  * `examples` -- that may include examples using the DTS language 

The patch-set documentation includes [an annotated example](<https://lwn.net/ml/linux-kernel/20181005165848.3474-9-robh@kernel.org/>). A simple file could look like: 

```
$id: "http://devicetree.org/schemas/bindings/vendor/someexample.yaml#"
        $schema: "http://devicetree.org/meta-schemas/core.yaml#"
    
        title: Documentation example
    
        maintainers:
            - Our Maintainer <example@example.com>
    
        description:  |
            Multi-line description is to be added here.
    
        properties:
            # Here we define the compatible property with one possible string
            compatible:
                items:
                    - const: vendor,my-clk
            reg:
                maxItems: 1
    
        required:
            - compatible
```

The schema files must follow a number of rules and can be validated using so called meta-schemas that are also implemented in YAML. The meta-schemas are provided with the tools and documentation writers are not expected to modify them.

#### Current status and further plans

The submitted series adds build support, documentation, and converts some bindings in the ARM tree. An additional tool, `doc2yaml`, exists in [Herring's tree](<https://git.kernel.org/pub/scm/linux/kernel/git/robh/linux.git/log/?h=yaml-bindings>), but has not been submitted; the script can be used for a preliminary conversion of a device-tree binding file in the current text format to the YAML one.

The meta-schemas, schemas, and validation tools are hosted in a [separate repository](<https://github.com/robherring/yaml-bindings>) for now. The repository includes a number of scripts (implemented in Python 3 with [ruamel.yaml](<https://yaml.readthedocs.io/en/latest/>) and JSON Schema). There are three that are expected to be used the most. `dt-doc-validate` can help validating a schema. `dt-mk-schema` creates a single schema file from the provided schema files and generic schemas from the repository; the use of such processed schema is expected to speed up device-tree validation. Finally, `dt-validate` takes the YAML device trees and validates them against the schema.

With the patch set, the kernel build system gains two new targets: `dt_binding_check` to check the device-tree binding schema documents, and `dtbs_check`, which checks device-tree files against the binding schema. They are using the new functionality of the device-tree compiler (`dtc`) present in 4.20: YAML output. The exact format is intended for validation purposes only and may change in the future.

During the conversion effort, Herring refactored some of the documentation. In the process he moved the bindings to have one binding per file and moved miscellaneous bindings (that are used for multiple SoCs) to separate files. The [plan is to merge the core changes](<https://lwn.net/ml/linux-kernel/CAL_Jsq+8EwYzBmzSWL0q0YB3rP-xdrZreRnt1Rj-h=D8OTNVwA@mail.gmail.com/>) directly and then use specific trees for conversions of the bindings documentation. Future plans include conversion of more documentation files, validation of the examples against the schema, allowing validation of selected targets, and more control over which schemas are used for validation (so that it takes less time).

The patch set has received some comments, some with direct approval, others asking for clarifications. Many questions concerned the details of the schema file syntax. The overall push to more structured documentation of device-tree bindings seems to be uncontested. It looks likely that the tools to support the new format will be merged soon. The conversion itself might require more work, however. A simple attempt to convert random documentation files from the mainline kernel shows that many properties will require manual description. Setting this new format as a rule for new submissions would certainly help to make the format widespread in the months to come.

More information is available in the slides from a couple of recent Linaro Connect talks on this subject; they give examples of the device-tree schema and their usage: [presentation [PDF]](<https://www.slideshare.net/linaroorg/hkg18120-devicetree-schema-documentation-and-validation>) and [BoF slides [PDF]](<https://s3.amazonaws.com/connect.linaro.org/yvr18/presentations/yvr18-404.pdf>).
