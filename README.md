# JNI Bind 
  
`JNI Bind` is a new metaprogramming library that provides syntactic sugar for `C++` => Java/Kotlin.  It is header only, and can be easily included even without Bazel support.  It provides sophisticated type conversion with compile time validation of method calls and field accesses.

It requires clang enabled at C++17 or later.  It is compatible with Android.

## Table of Contents
- [About JNI Bind](#about)
- [Quickstart](#quickstart-without-bazel)
- [Usage](#usage)
  - [Classes](#classes)
  - [Local and Global Objects](#local-and-global-objects)
  - [Fields](#fields)
  - [Methods](#methods)
  - [Constructors](#constructors)
  - [Type Conversion Rules](#type-conversion-rules)
- [Advanced Usage](#advanced-usage)
  - [Forward Class Declarations](#forward-class-declarations)
  - [Multhreading](#multi-threading)
  - [Overloads](#overloads) 
  - [Class Loaders](#class-loaders)
  - [Arrays](#arrays)
- [License](#license)

<a name="about"></a>
## About JNI Bind
`JNI Bind` was originally developed at Google to simplify confusing verbose JNI boilerplate code. It has minimal overhead and provides extensive, intuitive type conversions.

**Many** features and optimisations are included:
  - **Object Management**
  - **Static signature generation and validation** (code calling invalid method names fail to build).
  - **Static cacheing of IDs** such as `jmethodID` and `jclass` with correct cacheing strategies for multi-threading.
  - **Classes** (including native construction and compile time argument validation).
  - **Classloaders** (native object construction, argument validation, object "buildability").
  - **Arrays** (inline object construction for method arguments, efficient pinning of existing spans).
  - **JVM Management** Power users who want to manage multiple JVM lifetimes in a single process.
  - And *much* more!

<a name="quickstart-without-bazel"></a>
## Quickstart *without Bazel*

If you want to jump right in, simply copy [jni_bind_release.h](jni_bind_release.h) into your own project.  The header itself is a automatically generated "flattened" version of the Bazel dependency set, so the documentation is a much simpler introduction to `JNI Bind` than attempting to read through it directly.

You are responsible for ensuring `#include <jni.h>` is successful.  E.g. given the `WORKSPACE` above this is a sufficient build target:

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

<a name="quickstart_with_bazel"></a>
## Quickstart *with Bazel*

If you're already using Bazel add the following to your WORKSPACE:

```starlark
http_archive(                                                                                        
  name = "jni-bind",                                                                                 
  urls = ["https://github.com/google/jni-bind/archive/refs/tags/Release-0.1.0-alpha.zip"],           
  strip_prefix = "jni-bind-Release-0.1.0-alpha",                                                     
)
```

<a name="usage"></a>
# Usage

<a name="jvm-lifecycle"></a>
## JVM Lifecycle

JNI Bind requires some minor bookkeeping in order to ensure acccess to a valid `JNIEnv*` as well cached `jMethodIDs`, `jclass`, etc.  The simplest way to do this is to include it in your `JNI_OnLoad` call.  Ensure this object outlives the surrounding `JNI` call or object's lifetime.

```cpp
JNIEXPORT jint JNICALL JNI_OnLoad(JavaVM* pjvm, void* reserved) {
  static std::unique_ptr<jni::JvmRef<jni::kDefaultJvm>> jvm {pjvm};
  return JNI_VERSION_1_6;
}
```
:warning: If you using another JNI Library initialise `JNI Bind` after. `JNI Bind` "attaches" the thread explicitly through JNI, and some libraries behaviour will be conditional on this being unset.  If JNI Bind discovers a thread has been attached previously it will not attempt to tear the thread down on teardown. 

<a name="classes"></a>
## Classes

Class definitions are the basic mechanism by which you interact with Java through JNI and resemble the following:

```cpp
static constexpr jni::class kClass{"com/full/class/name/JavaClassName", jni::Field..., jni::Method... };
```

Class definitions are static (the class names of any Java object is known in advance).  Instances of these classes are created at runtime using [`jni::LocalObject`](local_object.h) or [`jni::GlobalObject`](global_object.h).

<a name="local-and-global-objects"></a>
## Local and Global Objects

Local and global objects manage lifetimes of underlying `jobjects` using the normal RAII mechanism of C++. **`jni::LocalObject` always fall off scope at the conclusion of the surrounding JNI call and are valid only to a single thread, however `jni::GlobalObject` may be held indefinitely and are thread safe**.

`jni::LocalObject` is built by either wrapping constructing it from a `jobject` passed to native JNI from Java, or constructing a new object from native (see [Constructors](constructors)).

When `jni::LocalObject` or `jni::GlobalObject` falls off scope, it will unpin the underlying `jobject`, making it available for garbage collection by the JVM.  If you want to to prevent this call `Release()`. This is useful to return a `jobject` back from native, or to simply pass to another native component that isn't JNI Bind aware.

[Sample C++](javatests/com/jnibind/test/context_test_jni.cc), [Sample Java](javatests/com/jnibind/test/ContextTest.java)

<a name="fields"></a>
## Fields

A `jni::Field` is described as a field name and a zero value of the underlying type.

```cpp
static constexpr jni::Field kIntField{"intField", jint{} };
static constexpr jni::Field kFloatField{"floatField", jfloat{} };
static constexpr jni::Field kClassField{"kClassField", kClass };
static constexpr jni::Field kClassField2{"kClassField2", jni::Class{"com/project/kClass" };
```
If a class definition contains `jni::Fields` in its definition corresponding objects constructed will be imbued with `operator[]`. Using this operator with the corresponding field name will provide a proxy object with two methods `Get` and `Set`.  e.g.

```cpp
static constexpr jni::Class kClass { "kClassName", jni::Field{"field_name", jint{} };

jni::LocalObject<kClass> runtime_object{jobj};
runtime_object["field_name"].Set(5);
runtime_object["field_name"].Get();
```

Accessing and setting fields will follow the rules laid out in [Type Conversion Rules](#type-conversion-rules). *Accessing invalid field names won't compile, and `jfieldID`s are cached on your behalf.* 

<a name="method-definitions"></a>
## Methods

A `jni::Method` is described as a method name, a `jni::Return`, and an optional `jni::Params` containing a variadic pack of zero values of the desired type.

```cpp
static constexpr jni::Method kIntMethod{"intMethod", jni::Return{jint{}} };
static constexpr jni::Method kFloatMethod{"floatMethod", jni::Return{jfloat{}}, jni::Params{ jint{}, jfloat{} }};
static constexpr jni::Method kClassMethod{"kClassMethod", jni::Return{kClass} };
```

If a class definition contains `jni::Method`s in its definition corresponding objects constructed will be imbued with `operator()`. Using this operator with the corresponding method name will invoke the corresponding method.

```cpp
static constexpr jni::Class kClass { "com/project/kClass",
   jni::Method{"intMethod", jni::Return{jint{}}
};

jni::LocalObject<kClass> runtime_object{jobj};
int ret_val = runtime_object("intMethod");
```

Methods will follow the rules laid out in [Type Conversion Rules](#type-conversion-rules). *Invalid method names won't compile, and `jmethodID`s are cached on your behalf.* 

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

<a name="forward-class-declarations"></a>
## Forward Class Declarations

Sometimes you need to reference a class that doesn't yet have a corresponding JNI Bind definition (possibly due to a circular dependency or self reference). This can be obviated by defining the class inline.  E.g.
  
```cpp
using jni::Class;
using jni::Constructor;
using jni::Return;
using jni::Method;
using jni::Params;

static constexpr Class kClass {
  "com/project/kClass",
  Method{"returnsNothing", Return<void>{}, Params{Class{"kClass2"}},
  Method{"returnsKClass", Return{Class"com/project/kClass"}},           // self referential
  Method{"returnsKClass2", Return{Class"com/project/kClass2"}},          // undefined
};

static constexpr Class kClass2 {
  "com/project/kClass2",
  Constructor{ Class{"com/project/kClass2"} },  // self referential
  Method{"Foo", Return<void>{}}
};

LocalObject<kClass> obj1{};
obj1("returnsNothing", LocalObject<kClass2>{});          // correctly forces kClass2 for arg
LocalObject<kClass> obj2{ obj1("returnsKClass") };       // correctly forces kClass for return 
LocalObject shallow_obj{} = obj1("returnsKClass");       // returns unusable but name safe kClass
// shallow_obj("Foo");                                   // this won't compile as it's only a forward decl
LocalObject<kClass> rich_obj{ std::move(shallow_obj) };  // promotes the object to a usable version
LocalObject<kClass2> { obj1("returnsKClass2") };         // materialised from a temporary shallow rvalue
  
```
Note, if you use the output of a forward declaration, it will result in a *shallow* object. You can use this object in calls to other methods or constructors an they will validate as expected.
  
<a name="multi-threading"></a>
## Multhreading  
  
<a name="overloads"></a>
## Overloads

<a name="class-loaders"></a>
## Class Loaders

<a name="arrays"></a>
## Arrays 

<a name="license"></a>
## License
The `JNI Bind` library is licensed under the terms of the Apache license. See [LICENSE](LICENSE) for more information.
