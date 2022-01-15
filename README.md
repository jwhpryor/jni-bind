# JNI Bind 
  
`JNI Bind` is a new metaprogramming library that provides syntactic sugar for `C++` => `Java/Kotlin`.  It is header only, and can be easily included even without Bazel support.

It requires the clang enabled at C++17 or later.  It is compatible with Android.

## Table of Contents
- [About JNI Bind](#about)
- [Quickstart](#quickstart-without-bazel)
- [Usage](#usage)
  - [Class Definitions](#class-definitions)
  - [Methods](#methods)
  - 
- [Advanced Usage](#advanced-usage)
  - [Multhreading](#multi-threading) 
  - [Class Loaders](#class-loaders)
  - [Arrays](#arrays)
- [License](#license)

<a name="about"></a>
## About JNI Bind
`JNI Bind` was originally developed at Google for use in various Android system services to simplify overly complicated JNI code.  `JNI Bind` handles boilerplate code with very little overhead and frequently can result in performance wins.

**Many** features and optimisations are included:
  - **Object Management** Rich expression for local and global objects.
  - **Static signature generation and validation** (code calling invalid method names fail to build).
  - **Static cacheing of IDs** such as `jmethodID` and `jclass` with correct cacheing strategies for multi-threading.
  - **Classes** (including native construction and compile time argument validation).
  - **Classloaders** (native object construction, argument validation, object "buildability").
  - **Arrays** (inline object construction for method arguments, efficient pinning of existing spans).
  - **JVM Management** Power users who want to manage multiple JVM lifetimes in a single process.
  - And *much* more!

<a name="quickstart_without_bazel"></a>
## Quickstart *without Bazel*

If you want to jump right in, simply copy [jni_bind_release.h](jni_bind_release.h) into your own project.  The header itself is a automatically generated "flattened" version of the Bazel dependency set, so the documentation is a much simpler introduction to `JNI Bind` than attempting to read through it directly.

You are responsible for ensuring `#include <jni.h>` is successful.  Given the WORKSPACE the following is sufficient:

```
cc_library(                                                                                          
    name = "Foo",                                                                                    
    hdrs = [                                                                                         
        "jni_bind_release.h",                                                                        
    ],                                                                                               
    srcs = [
        "Foo.cc",
        "@local_jdk//:jni_header",
        "@local_jdk//:jni_md_header-linux",                                                          
    ],                                                                                               
    includes = [                                                                                     
        "external/local_jdk/include",                                                                
        "external/local_jdk/include/linux",                                                          
    ],                                                                                               
)
```

<a name="quickstart_with_bazel"></a>
## Quickstart *with Bazel*

If you're already using Bazel add the following to your WORKSPACE:

```
http_archive(                                                                                        
  name = "jni-bind",                                                                                 
  urls = ["https://github.com/google/jni-bind/archive/refs/tags/Release-0.1.0-alpha.zip"],           
  strip_prefix = "jni-bind-Release-0.1.0-alpha",                                                     
)
```

<a name="Usage"></a>
# Usage

<a name="jvm_lifecycle"></a>
## JVM Lifecycle

-- Jvm 

<a name="class_definitions"></a>
## Class Definitions

Classes are...

<a name="method_definitions"></a>
## Method Definitions


<a name="license"></a>
## License
The `JNI Bind` library is licensed under the terms of the Apache license. See [LICENSE](LICENSE) for more information.
