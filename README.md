# JNI Bind

`JNI Bind` is a new metaprogramming library that provides syntactic sugar for `C++` => `Java/Kotlin`.  It is header only and provides sophisticated type conversion with compile time validation of method calls and field accesses.

It requires clang enabled at C++17 or later, is compatible with Android, and is unit / E2E tested on `x86`/`ARM` toolchains.

**Many** features and optimisations are included:
- **Object Management**
- **Compile time method name and argument checking**
- **Static cacheing of IDs** *multi-threading support and compile time signature generation.*
- **Classes** *native construction, argument validation, lifetime support, method and field support.*
- **Classloaders** *native construction, object "buildability".*
- **JVM Management** *single-process multiple JVM lifecycles.*
- **Strings** *easy syntax for inline argument construction.*
- **Arrays** *inline object construction for method arguments, efficient pinning of existing spans.*
- And *much* more!

## Table of Contents
- [Quick Intro](#quick-intro)
- [Installation](#installation-without-bazel)
- [Usage](#usage)
  - [Classes](#classes)
  - [Local and Global Objects](#local-and-global-objects)
  - [Fields](#fields)
  - [Methods](#methods)
  - [Constructors](#constructors)
  - [Type Conversion Rules](#type-conversion-rules)
- [Advanced Usage](#advanced-usage)
  - [Strings](#strings)
  - [Forward Class Declarations](#forward-class-declarations)
  - [Multhreading](#multi-threading)
  - [Overloads](#overloads)
  - [Class Loaders](#class-loaders)
  - [Arrays](#arrays)
- [Upcoming Features](#upcoming-features)
- [License](#license)

<a name="quick-intro"></a>
## Quick Intro

**JNI is notoriously difficult to use, and numerous libraries exist to try to reduce this complexity**. These libraries usually require code generation (or extensive macro use) which leads to brittle implementation, indexes poorly, and still requires domain expertise in JNI (making code difficult to maintain).

**`JNI Bind` is header only (no auto-generation), and it generates robust, easily maintained, and expressive code.** It obeys the regular RAII idioms of C++17 and can help separate JNI symbols in compilation. Classes are provided in `static constexpr` definitions which can be shared across different implementations enabling code re-use.

This is a sample  Java class and it's corresponding`JNI Bind` class definition:

```java
package com.project;
public class Clazz {
 int Foo(float f, String s) {...}
}
```

```cpp
#include "jni_bind_release.h"

static constexpr jni::class kClass {
  "com/project/clazz", jni::Method { "Foo", jni::Return<jint>{}, jni::Params<jfloat, jstring>{}},

jni::LocalObject<kClass> obj { jobject_to_wrap };
obj("Foo", 1.5f, "argString");
// obj("Bar", 1.5, "argString");  // won't compile (good).
```

There are [sample tests](javatests/com/jnibind/test/) which can be a good way to see some example code.  Consider starting with with [context_test_jni](javatests/com/jnibind/test/context_test_jni.cc), [object_test_helper_jni.h](/javatests/com/jnibind/test/object_test_helper_jni.h) and [ContextTest.java](javatests/com/jnibind/test/ContextTest.java).

<a name="installation-without-bazel"></a>
## Installation *without Bazel*

If you want to jump right in, copy [jni_bind_release.h](jni_bind_release.h) into your own project.  The header itself is an automatically generated "flattened" version of the Bazel dependency set, so this documentation is a simpler introduction to `JNI Bind` than reading the header directly.

You are responsible for ensuring `#include <jni.h>` compiles when you include `JNI Bind`.

<a name="installation-with-bazel"></a>
## Installation *with Bazel*

If you're already using Bazel add the following to your WORKSPACE:

```starlark
http_archive(
  name = "jni-bind",
  urls = ["https://github.com/google/jni-bind/archive/refs/tags/Release-0.1.0-alpha.zip"],
  strip_prefix = "jni-bind-Release-0.1.0-alpha",
)
```

Then include `@jni-bind//:jni_bind` (not `:jni_bind_release`) and `#include "jni_bind.h"`.

JNI is sometimes difficult to get working in Bazel, so if you don't have an environment already also add this to your `WORKSPACE`, and follow the pattern of the BUILD target below.

```starlark
# Rules Jvm.
RULES_JVM_EXTERNAL_TAG = "4.2"
RULES_JVM_EXTERNAL_SHA = "cd1a77b7b02e8e008439ca76fd34f5b07aecb8c752961f9640dea15e9e5ba1ca"

http_archive(
    name = "rules_jvm_external",
    strip_prefix = "rules_jvm_external-%s" % RULES_JVM_EXTERNAL_TAG,
    sha256 = RULES_JVM_EXTERNAL_SHA,
    url = "https://github.com/bazelbuild/rules_jvm_external/archive/%s.zip" % RULES_JVM_EXTERNAL_TAG,
)
```

```starlark
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
There are easy to lift samples in [javatests/com/jnibind/test/](javatests/com/jnibind/test/). If you want try building these samples (or to copy the BUILD configuration) you can clone *this* repo.

```bash
cd /tmp
git clone https://github.com/google/jni-bind.git
cd jni-bind
bazel test  --cxxopt='-std=c++17' --repo_env=CC=clang ...
```

<a name="usage"></a>
# Usage

<a name="jvm-lifecycle"></a>
## JVM Lifecycle
**JNI Bind requires some minor bookkeeping in order to ensure acccess to a valid `JNIEnv*`**. To do this, create a [`jni::JvmRef`](implementation/jvm_ref.h) whose lifetime extends past any JNI Bind call (*a function local static is a reasonable way to do this, although sanitizers will flag this as a memory leak, so all tests explicitly manage the lifetime of the `jni::jvmRef`).

The simplest way to from the `JavaVM*` in your `JNI_OnLoad` call. **This object's lifetime must outlive all JNI calls.**

```cpp
JNIEXPORT jint JNICALL JNI_OnLoad(JavaVM* pjvm, void* reserved) {
  static std::unique_ptr<jni::JvmRef<jni::kDefaultJvm>> jvm {pjvm};
  return JNI_VERSION_1_6;
}
```
:warning: If you using another JNI Library initialise `JNI Bind` after. `JNI Bind` "attaches" the thread explicitly through JNI, and some libraries behaviour will be conditional on this being unset.  If JNI Bind discovers a thread has been attached previously it will not attempt to tear the thread down on teardown.

You can also build a `jni::jvmRef` from any `JNIEnv*`.

<a name="classes"></a>
## Classes

Class definitions are the basic mechanism by which you interact with Java through JNI:

```cpp
static constexpr jni::class kClass{"com/full/class/name/JavaClassName", jni::Field..., jni::Method... };
```

jni::Class definitions are static (the class names of any Java object is known in advance).  Instances of these classes are created at runtime using [`jni::LocalObject`](implementation/local_object.h) or [`jni::GlobalObject`](implementation/global_object.h).

<a name="local-and-global-objects"></a>
## Local and Global Objects

Local and global objects manage lifetimes of underlying `jobjects` using the normal RAII mechanism of C++. **`jni::LocalObject` always fall off scope at the conclusion of the surrounding JNI call and are valid only to a single thread, however `jni::GlobalObject` may be held indefinitely and are thread safe**.

`jni::LocalObject` is built by either wrapping constructing it from a `jobject` passed to native JNI from Java, or constructing a new object from native (see [Constructors](constructors)).

When `jni::LocalObject` or `jni::GlobalObject` falls off scope, it will unpin the underlying `jobject`, making it available for garbage collection by the JVM.  If you want to to prevent this call `Release()`. This is useful to return a `jobject` back from native, or to simply pass to another native component that isn't JNI Bind aware.  Calling methods for a released object is undefined.

When possible try to avoid using raw `jobject`. Managing lifetimes with regular JNI is difficult, e.g. `jobject` can mean either local or global object (the former will be automatically unpinned at the end of the JNI call, but the latter won't and must be deleted _exactly_ once).

[Sample C++](javatests/com/jnibind/test/context_test_jni.cc), [Sample Java](javatests/com/jnibind/test/ContextTest.java)

<a name="method-definitions"></a>
## Methods

**A `jni::Method` is described as a method name, a `jni::Return`, and an optional `jni::Params` containing a variadic pack of zero values of the desired type.** If a class definition contains `jni::Method`s in its definition then corresponding objects constructed will be imbued with `operator()`. Using this operator with the corresponding method name will invoke the corresponding method.

```cpp
static constexpr jni::Class kClass {
   "com/project/kClass",
   jni::Method{"intMethod", jni::Return{jint{}},
   jni::Method {"floatMethod", jni::Return{jfloat{}}, jni::Params{ jint{}, jfloat{} }},
   jni::Method {"classMethod", jni::Return{kClassFromOtherHeader} }
};

jni::LocalObject<kClass> runtime_object{jobj};
int int_val = runtime_object("intMethod");
float float_val = runtime_object("floatMethod");
jni::LocalObject<kClassFromOtherHeader> class_val { runtime_object("classMethod") };
```

Methods will follow the rules laid out in [Type Conversion Rules](#type-conversion-rules). *Invalid method names won't compile, and `jmethodID`s are cached on your behalf. Method lookups are compile time, there is no hash lookup cost.*

[Sample C++](javatests/com/jnibind/test/method_test_jni.cc), [Sample Java](javatests/com/jnibind/test/MethodTest.java)

<a name="fields"></a>
## Fields

**A `jni::Field` is described as a field name and a zero value of the underlying type.** If a class definition contains `jni::Fields` in its definition then corresponding objects constructed will be imbued with `operator[]`. Using this operator with the corresponding field name will provide a proxy object with two methods `Get` and `Set`.

```cpp
static constexpr jni::Class kClass {
  "kClassName",
  jni::Field {"intField", jint{} },
};

jni::LocalObject<kClass> runtime_object{jobj};
runtime_object["intField"].Set(5);
runtime_object["intField"].Get(); // = 5;
```

Accessing and setting fields will follow the rules laid out in [Type Conversion Rules](#type-conversion-rules). *Accessing invalid field names won't compile, and `jfieldID`s are cached on your behalf.*

*Fields are currently missing class support which will be added soon.

[Sample C++](javatests/com/jnibind/test/field_test_jni.cc), [Sample Java](javatests/com/jnibind/test/FieldTest.java)

<a name="constructors"></a>
## Constructors

If you want to create a new Java object from native code, you can define a `jni::Constructor`cpp, or use the default constructor. **If you omit a constructor the default constructor is called, but if any are specified you must explicitly define a no argument constructor.**

```cpp
static constexpr jni::Class kSomeClass{...};

static constexpr jni::Class kClass {
   "com/google/Class",
   jni::Constructor{jint{}, kSomeClass},
   jni::Constructor{jint{}},
   jni::Constructor{},
};

jni::LocalObject<kClass> obj1{123, jni::LocalObject<kSomeClass>{}};
jni::LocalObject<kClass> obj2{123};
jni::LocalObject<kClass> obj3{};  // Compiles only because of jni::Constructor{}.
```

Constructors follow the arguments rules laid out in [Type Conversion Rules](#type-conversion-rules).

[Sample C++](javatests/com/jnibind/test/object_test_helper_jni.h).

<a name="type-conversion-rules"></a>
## Type Conversion Rules

All JNI types have corresponding `JNI Bind` types. These types are used differently depending on their context.  Sometimes multiple types are valid as an argument, and you can use them interchangeably, however *types are strictly enforced, so you can't pass arguments that might otherwise implictly cast*.

| JNI C API    | Jni Bind Declaration         | Types valid when used as arg                      | Return Type                                    |
| ------------ | ---------------------------- | ------------------------------------------------- | ---------------------------------------------- |
| void         |                              |                                                   |                                                |
| jboolean     | jboolean                     | jboolean, bool                                    | jint                                           |
| jint         | jint                         | jint, int                                         | jint                                           |
| jfloat       | jfloat, float                | jfloat                                            | jfloat                                         |
| jdouble      | jdouble, double              | jdouble                                           | jdouble                                        |
| jlong        | jlong, long                  | jlong                                             | jlong                                          |
| jobject      | jni::Class kClass            | jobject, jni::LocalObject, jni::GlobalObject[^1]  | jni::LocalObject<kClass>                       |
| jstring      | jstring                      | jstring, jni::LocalString, jni::GlobalString,     | jni::LocalString                               |
|              |                              | char*, std::string                                |                                                |
| jarray       | jni::Array<T>                | jarray, jni::LocalArray, , jlongArray             | jni::LocalArray<T>                             |
|              |                              | jbooleanArray, jbyteArray, jcharArray,            |                                                |
|              |                              | jfloatArray, jintArray                            |                                                |
| jobjectarray | jni::Array<jobject, kClass>  | jarray, jni::LocalArray, , jlongArray             | jni::LocalArray<T>                             |

More conversions will be added later and this table will be updated as they are. In particular `wchar` views into jstrings will offer a zero copy view into java strings.  Also, `jarray` will support `std::span` views for its underlying types.

[^1]:Note, if you pass a `jni::LocalObject` or `jni::GlobalObject`as an rvalue, it will release the underlying jobject (mimicking the same rules used for [const& in C++](https://en.cppreference.com/w/cpp/language/lifetime)).

<a name="advanced-usage"></a>
# Advanced Usage

<a name="strings"></a>
## Strings

Strings are slightly atypical as they are a regular Java class ([java/lang/String](https://docs.oracle.com/javase/7/docs/api/java/lang/String.html)) but have a separate JNI type, `jstring`. They therefore have two separate `JNI Bind` types: `jni::LocalString` and `jni::GlobalString`.

Unlike `jni::LocalObject`, you must explicitly pin the underlying string data to access it (arrays also follow this paradigm). To "view" into the string, you must call `Pin()` which returns a `jni::UtfStringView`.  You can call `ToString()` to get a `std::string_view` into the string.

```cpp
jni::LocalString new_string{"TestString"};             // builds a new java/lang/String instance.
jni::UtfStringView utf_string_view = new_string.Pin();
std::string_view jni_string_view = utf_string_view.ToString();
```

`jni::UtfStringView `will immediately pin memory associated with the jstring, and release on leaving scope. This will *always* make an expensive copy, as strings are natively represented in Java as Unicode (C++20 will offer a compatible `std::string_view` but C++17 does not).

[Sample C++](javatests/com/jnibind/test/string_test_jni.cc), [Sample Java](javatests/com/jnibind/test/StringTest.java)

<a name="forward-class-declarations"></a>
## Forward Class Declarations

Sometimes you need to reference a class that doesn't yet have a corresponding JNI Bind definition (possibly due to a circular dependency or self reference). This can be obviated by defining the class inline.

```cpp
using jni::Class;
using jni::Constructor;
using jni::Return;
using jni::Method;
using jni::Params;

static constexpr Class kClass {
  "com/project/kClass",
  Method{"returnsNothing", Return<void>{}, Params{Class{"kClass2"}},
  Method{"returnsKClass", Return{Class{"com/project/kClass"}},           // self referential
  Method{"returnsKClass2", Return{Class{"com/project/kClass2"}},         // undefined
};

static constexpr Class kClass2 {
  "com/project/kClass2",
  Constructor{ Class{"com/project/kClass2"} },  // self referential
  Method{"Foo", Return<void>{}}
};

LocalObject<kClass> obj1{};
obj1("returnsNothing", LocalObject<kClass2>{});        // correctly forces kClass2 for arg
LocalObject<kClass> obj2{ obj1("returnsKClass") };     // correctly forces kClass for return
LocalObject shallow_obj{} = obj1("returnsKClass");     // returns unusable but name safe kClass
// shallow_obj("Foo");                                 // this won't compile as it's only a forward decl
LocalObject<kClass> rich_obj{std::move(shallow_obj)};  // promotes the object to a usable version
LocalObject<kClass2> { obj1("returnsKClass2") };       // materialised from a temporary shallow rvalue

```
Note, if you use the output of a forward declaration, it will result in a *shallow* object. You can use this object in calls to other methods or constructors and they will validate as expected.

Sample: [proxy_test.cc](implementation/proxy_test.cc).

<a name="multi-threading"></a>
## Multhreading

When using multi-threaded code in native, it's important to remember the difference between maintaining thread safety in your native code vs your Java code. `JNI Bind` *only* handles thread safety of your native handles and class ids.  E.g. you might safely pass a `jobject` from one native thread to another (i.e. you managed to get the handle correctly to the other thread), however, your Java code may not be expecting calls from multiple threads.

**To pass a jobject from one thread to another you must use `jni::GlobalObject`** (using `jni::LocalObject` is undefined).

Upon spinning a new native thread (that isn't the main thread), you must declare a `jni::ThreadGuard` to explicitly announce to JNI the existence of this thread.  It's permissible to to have nested `jni::ThreadGuard`s.

Sample [jvm_test.cc](implementation/jvm_test.cc).

<a name="overloads"></a>
## Overloads

Methods can be overloaded just like in regular Java by declaring a name, and then a variadic pack of [`jni::Overload`](implementation/method.h). Overloaded methods are invoked like regular methods. `JNI Bind` will correctly differentiate between functions that differ only by type, including functions that take different class types.

```cpp
static constexpr Class kClass{
    "com/google/SupportsStrings",
    Method{
        "Foo",
        Overload{jni::Return<void>{}, Params{jint{}}},
        Overload{jni::Return<void>{}, Params{jstring{}}},
        Overload{jni::Return<void>{}, Params{jstring{}, jstring{}}},
    }
};

LocalObject<kClass> obj{};
obj("Foo", 1);
obj("Foo", "arg");
obj("Foo", "arg", jstring{nullptr});
```

Sample [method_test_jni.cc](javatests/com/jnibind/test/method_test_jni.cc), [MethodTest.java](javatests/com/jnibind/test/MethodTest.java).

<a name="class-loaders"></a>
## Class Loaders

*Documentation coming soon!*

<a name="arrays"></a>
## Arrays

*Documentation coming soon!*

<a name="upcoming-features"></a>
## Upcoming Features

Feature requests are welcome! Some upcoming features are:
- Statics
- Tighter array type validation (arrays are overly permissive for arguments)
- Better error messages
- Link time symbol validation in Bazel
- Unit Testing Support (enabling unit testing of JNI interfaces)
- Callback syntactic sugar
- Per JNI call lambda invocations (e.g. per JNI call logging, perf tracing)
- Auto generated interfaces (with pre-loaded scrapes of Java libraries)
- And more!

<a name="license"></a>
## License
The `JNI Bind` library is licensed under the terms of the Apache license. See [LICENSE](LICENSE) for more information.
