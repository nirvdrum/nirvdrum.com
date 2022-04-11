---
layout: post
draft: true
title: Embedding Truffle Languages
author: Kevin Menard
date: 2022-04-11 15:21 -0400
---
Introduction
------------

The past several years of my career have been spent predominately working on [TruffleRuby](https://www.graalvm.org/ruby/), an implemention of the Ruby programming language that can achieve impressive execution speed thanks to the [Truffle](https://www.graalvm.org/22.0/graalvm-as-a-platform/language-implementation-framework/) language implementation framework and the [Graal](https://github.com/oracle/graal/tree/master/compiler) JIT compiler.
Taken together, these three technologies form part of the GraalVM distribution.
The full distribution includes implementations of other languages (JavaScript, Python, R, and Java), an interpreter for LLVM bitcode (Sulong), tooling such as a profiler and debugger for both host and guest code (VisualVM), tooling to visualize decisions made by the JIT compiler (IGV), and the ability to generate native binaries of Java applications (including any of the listed language interpreters) via Native Image.
There's more to GraalVM as well, which makes defining it and discovering all of its capabilities difficult.
In this article, I'd like to focus on two pieces of GraalVM functionality: 1) programmatically driving any of the Truffle-based languages from Java; and 2) using Native Image to generate native shared libraries to support loading code into other applications.


Native Image Overview
---------------------

GraalVM's [Native Image](https://www.graalvm.org/22.0/reference-manual/native-image/) tool can build native executables and native shared libraries from Java code.
By default, these binaries will have a dependency on your system's libc and implementations, but you can instruct Native Image to statically link in libc and zlib libraries if you have them, leaving you with a binary that has no external dependencies.
In effect, you can use Java just as you would any other ahead-of-time (AOT) compiled language.
In contrast to C/C++, Rust, or other similar systems languages, you still have access to the Java VM facilities such as Java's IO abstraction and garbage collection (GC).
However, the VM facilities are not provided by HotSpot, but rather a new VM written specifically for Native Image binaries called SubstrateVM.

As with most technology decisions, there's a trade-off: Native Image binaries start considerably faster than running an application in the JVM, but they forego the ability to JIT the application<a href="#footnote_1"><sup>1</sup></a>.
Additionally, the SubstrateVM garbage collector is not quite as polished as HotSpot one.
That doesn't mean that there is no optimization at all.
The Native Image compilation process will run AOT optimization passes as it builds the image.
The enterprise version of GraalVM also supports profile-guided optimization (PGO) to help Native Image make compilation decisions that are favorable to the profiled application.
Native Image binaries also make distribution easier, since you don't need to have a JVM available in your target environment.
While Native Image binaries may not be the best option for long-running server applications, they open up the ability to run Java applications in environments that the language was previously ill-suited towards, such as Functions as a Service (FaaS), which need to start up quickly and are ephemeral.
TruffleRuby ships as a native image so it can 

In order to build a binary of a native application while still supporting broad use of the JDK, the Native Image performs an extensive closed world analysis to figure out exactly what classes and methods your application uses and only compiles those into the application.
Notably, this means your application can't generally use reflection (including JNI) or dynamic classloading because that data just won't exist in the resulting binary.
If you can constrain your usage of either to a fixed set of items, you can provide a configuration file to the Native Image compiler so it knows to compile in extra classes or additional information needed for reflection/JNI.


Embedding Truffle Languages
---------------------------

At their core, Truffle language interpreters are just Java applications.
Certainly, the language implementations also use non-Java code (e.g., a substantial portion of TruffleRuby is written in Ruby and some parts in C), but they all can be loaded and invoked in Java using the [Truffle Polyglot API](https://www.graalvm.org/22.0/reference-manual/polyglot-programming/).
By using _libjvm_ and the Java Native Interface (JNI) Invocation API, we can load a copy of the JVM up into a non-Java application and execute code in a Truffle language via the Truffle Polyglot API.
But, loading an entire copy of the JVM up is rather slow and memory intensive.

As Java applications, a Truffle interpreter can be compiled via Native Image<a href="#footnote_2"><sup>2</sup></a>.
Moreover, Native Image can link the entirety of a Truffle interpreter into the resulting binary (executable or shared library).
Following this approach, we can generate a library to run our Ruby code that starts quickly, uses less RAM than _libjvm_, requires less disk space than JVM distribution, _and_ have an integrated JIT to optimize our code running in the interpreter.


Native Image Playground
-----------------------

The GraalVM distribution ships with a dizzying amount of functionality.
Most of it is very well documented, but some of it is either lacking or simply assumes the reader has more information than this post will assume.
To help illustrate some of the techniques described here, I've pulled together a project called the [Native Image Playground](https://github.com/nirvdrum/native-image-playground) which has several examples of using Native Image to build standalone executables and shared libraries, for loading a Truffle interpreter into another process, and for executing multiple Truffle languages (e.g., Ruby and JavaScript) within the same process.
I will refer to examples from Native Image Playground in this article.
If you wish to run the code on your own, please follow the steps outlined in the project's [README](https://github.com/nirvdrum/native-image-playground/blob/main/README.md) to ensure you have all the necessary prerequisites.

Many of the examples in the Native Image Playground compute the [Haversine distance](https://en.wikipedia.org/wiki/Haversine_formula): a way to measure geographic distance on the surface of a sphere.
This algorithm was chosen because it was the same one used by Chris Seaton in his [Top 10 Things To Do With Graal](https://chrisseaton.com/truffleruby/tenthings/) post, which is a spiritual predecessor to this piece.
The algorithm is implemented with simple arithmetic and trigonometry, so we can easily evaluate the generated machine code for the operation.
As a caveat though, the algorithm implementation was taken from the [Apache SIS](https://sis.apache.org/) project and was found to be incorrect.
Since the purpose of this post isn't to be a reference for geopspatial operations, I've pushed ahead with the incorrect algorithm in order to retain parity with Chris's earlier post and because the correct implementation is more involved, complicating our performance analysis.


Calling Methods From a Native Image Shared Library
--------------------------------------------------

As of GraalVM 22.0.0.2, there are two primary mechanisms for calling a method embedded in a Native Image shared library: the [Native Image C API](https://www.graalvm.org/22.0/reference-manual/native-image/C-API/) and the [JNI Invocation API](https://docs.oracle.com/en/java/javase/17/docs/specs/jni/invocation.html).
The Native Image C API is somewhat begrudgingly supported and likely to be removed in the not too distant future.
It's an extra API specific to Native Image that the GraalVM would like to remove in favor of the more standard JNI Invocation API.
In a Native Image binary, JNI is retargeted to work with [GraalVM Isolates](https://medium.com/graalvm/isolates-and-compressed-references-more-flexible-and-efficient-memory-management-for-graalvm-a044cc50b67e), the mechanism by which GraalVM supports multiple, isolated execution environments within the same process.
However, JNI performance within a Native Image is limited pending the merge of [Project Panama](https://openjdk.java.net/projects/panama/) to the JDK.
As a result, we have two methods for calling natively compiled Java methods from a library where neither can be fully endorsed at the moment.

### Native Image C API

Don't be put off by the name "Native Image C API".
While GraalVM makes it easy to use C to call into Native Image shared libraries, the name is more of an indication as to how the functions will be exported from the library.
You can use this API in any language with the ability to call foreign functions (e.g., using the FFI or Fiddle libraries in Ruby).

By default, nothing is exported from your shared library other than a function named `main` should you have a `public static void main` method somewhere in your Java code.
Otherwise, to export a Java method you must do the following:

* Declare the method as `static`
* Make the first argument an `org.graalvm.nativeimage.IsolateThread`
* Restrict your parameter and return types to primitive types or a type from the `org.graalvm.nativeimage.c.type` package
* Annotate the method with the `org.graalvm.nativeimage.c.function.CEntryPoint` annotation

If you look at the various `org.graalvm.nativeimage` sub-packages, you'll find some code for handling additional cases that we are not going to do so here, such as mapping Java interfaces to C structs.
For the Haversine distance calculations, all parameters will be doubles and the return value will be a double as well, so we won't need any of the additional functionality that Native Image makes available.

Taking the NativeLibrary example from the Native Image Playground project, we have the following:

```java
@CEntryPoint(name = "distance")
public static double distance(IsolateThread thread,
        double a_lat, double a_long,
        double b_lat, double b_long) {
    return DistanceUtils.getHaversineDistance(a_lat, a_long, b_lat, b_long);
}
```
<div class="caption"><caption>Example 1: Haversine distance in Java exposed as C function in Native Image shared library.</caption></div>

The `name` attribute in the `@CEntryPoint` annotation may be omitted, but the default name is constructed from the class and method names along with randomly generated number to ensure uniqueness.
Naturally, since the methods are being exposed in a shared library, they must have unique names.
If you give two exposed methods the same name, the Native Image compiler will fail with a message such as:

```
duplicate symbol '_distance' in:
    libnative-library-runner.o
ld: 1 duplicate symbol for architecture x86_64
```
<div class="caption"><caption>Example 2: Error message building Native Image shared library with duplicate exposed function names.</caption></div>

When you build the binary, Native Image will also generate some C header files for you.
If working with C or C++, you can use these header files directly.
For other languages, you can use the function declarations in the headers to set up your foreign call bindings.
The code found in Example 1 will result in the following function declaration:

```c
double distance(graal_isolatethread_t*, double, double, double, double);
```
<div class="caption"><caption>Example 3: Function declaration for the Haversince distance method exposed in the Native Image shared library.</caption></div>

As you can see, the Java `double` type is mapped to the C `double` type.
The Java `IsolateThread` type is mapped to a `graal_isolatethread_t*` in C.

#### Working with Isolates

Every function you would like to expose in a Native Image shared library using `@CEntryPoint` must have an `IsolateThread` as its first parameter and every call to that method through the shared library must supply a Graal Isolate pointer as its first argument.
Looking at the code in Example 1, the `distance` method doesn't do anything with the Isolate parameter.
The actual usage of the Isolate handle is managed by Native Image in the generated binary.

Along with the header file generated with all of the function declarations for exposed methods in the shared library, Native Image also generates a _graal_isolate.h_ file with type definitions and function declarations for working with the Native Image C API.

The naming here might be a bit confusing.
There are Graal Isolates and Graal Isolate Threads.
When calling a function exposed in Native Image shared library, you must actually supply a pointer to an Isolate Thread and all Isolate Threads must be attached to an Isolate.
Creating an Isolate will implicitly create a primary Isolate Thread and that is what the sample projects in Native Image Playground use (i.e., none of the sample projects dig into multi-threading).
All Graal Isolates and Isolate Threads must be torn down when you're done with them; tearing down the Isolate will also teardown the primary Isolate Thread.

Another way of working with Isolates is to expose your own functions in the shared library by using `@CEntryPoint` built-ins.
The Native Image Playground samples do not make extensive use of this form of resource management, but some do for completeness.
To expose these methods, you would use something like the following:

```java
@CEntryPoint(builtin = CEntryPoint.Builtin.CREATE_ISOLATE, name = "create_isolate")
static native IsolateThread createIsolate();

@CEntryPoint(builtin = CEntryPoint.Builtin.TEAR_DOWN_ISOLATE, name = "tear_down_isolate")
static native int tearDownIsolate(IsolateThread thread);
```
<div class="caption"><caption>Example 4: Using @CEntryPoint built-ins to expose Graal Isolate resource management methods in the Native Image shared library with custom names.</caption></div>


### Java Native Interface (JNI) Invocation API

The preferred mechanism for invoking code in a Native Image shared library is to use the Java Native Interface (JNI) Invocation API &mdash; a standard JDK API for starting and programmatically controlling a JVM from another process.
Usage of JNI Invocation API might seem a bit odd, given a defining feature of Native Image binaries is that they do not include the JVM.
Native Image binaries do include a VM though to handle things like GC and thread scheduling.
This alternative VM, called the [Substrate VM](https://www.graalvm.org/22.0/reference-manual/native-image/), reimplements the JNI Invocation API to create Graal Isolates and Isolate Threads and adjusts the rest of the API so that JNI calls bind to the appropriate Isolate Thread _(see the earlier discussion on [Graal Isolates](#working-with-isolates) if you're unsure what that means)_.

By using the JNI Invocation API, you don't need to learn a new Native Image-specific way to write code that drives a Java process.
However, much of JNI is essentially runtime reflection and Native Image does not allow arbitrary reflection.
In order to use JNI with a Native Image binary, you need to supply a [JNI configuration file](https://www.graalvm.org/22.0/reference-manual/native-image/JNI/) to the `native-image` command when build your image.
Manually creating that file is tedious and error-prone.
To simplify the process, I recommend using a [tracing agent](https://www.graalvm.org/22.0/reference-manual/native-image/Agent/) provided by GraalVM, which will record all JNI calls made at runtime and dump them out to a file.
To do so, you'll need to temporarily swap your application over to using _libjvm_, which will allow general JNI calls.
I found it easiest to set the `JAVA_TOOL_OPTIONS` environment variable, that way I wouldn't have to customize the `java` command in Maven.
Using the _jni-libjvm-polyglot_ example from the Native Image Playground, we have:

```bash
$ mvn -P jni-libjvm-polyglot -D skipTests=true clean package
$ export JAVA_TOOL_OPTIONS="-agentlib:native-image-agent=config-output-dir=$PWD/target-jni-libjvm/config-output-dir-{pid}-{datetime}/"
$ ./target-jni-libjvm/jni-runner js 51.507222 -0.1275 40.7127 -74.0059
```
<div class="caption"><caption>Example 5: Enable the Native Image tracing agent to record JNI calls.</caption></div>

In this example, we really didn't need to embed the PID or timestamp into the generated directory, but it's generally useful if you have multiple Java processes running since they'll all share the environment variable and thus would all dump to their output to the same directory.
If we take a look at that directory, we'll see the agent generated several files for us:

```bash
$ ls target-jni-libjvm/config-output-dir-40562-20220329T191144Z/

jni-config.json                 proxy-config.json               resource-config.json
predefined-classes-config.json  reflect-config.json             serialization-config.json
```
<div class="caption"><caption>Example 6: Configuration files generated by the Native Image tracing agent.</caption></div>

The _jni-config.json_ file is the one of interest.
We can pass that file to the `native-image` command using the `-H:JNIConfigurationFiles` option.
The _jni-native_ profile from the Native Image Playground does precisely that.
Both the _jni-libjvm-polyglot_ and _jni-native_ Maven profiles from the Native Image Playground use the the same exact C++ code launcher application to calculate the Haversine distance using a Truffle language through its Java polyglot API.
That's the primary draw of using the JNI Invocation API with Native Image; you don't need to learn a new non-standard API and your code will work without modification as you switch between _libjvm_ and the Native Image shared library.

### Benchmarks

When starting this project, I was only aware of the Native Image C API, so that's what I started with.
Between documentation, GitHub issues, and discussions with others on the [GraalVM Slack](https://graalvm.slack.com/join/shared_invite), I learned about the JNI support in Native Image.
But, I was also told that JNI calls would have higher overhead than using the Native Image C API until [Project Panama](https://openjdk.java.net/projects/panama/) is finished.
This presented a conflict because ultimately I'm investigating ways to embed languages like TruffleRuby into other applications.
The choice between fast &amp; deprecated (Native Image C API) and slower &amp; but API-stable (JNI Invocation API) is not the sort of trade-off I really wanted to make.
I haven't been actively tracking Project Panama, but it's not in Java 17 and GraalVM only uses Java LTS releases.
The next planned LTS release will be Java 21 and that's targeted for Sept. 2023 &mdash; too far out to wait for this application.

While I spoken with people that experienced significant slowdowns in trying to migrate from the Native Image C API to the JNI Invocation API, I couldn't find any numbers supporting the claims.
Thus, the final aspect of the Native Image Playground is to benchmark different different options for executing code in Truffle languages embedded in a process.
Whether using the Native Image C API or the JNI Invocation API, there are several different ways to call into a Truffle language, so the benchmarks include multiple approaches with each of the Native Image shared library APIs.

I want to reiterate that the focus of these benchmarks is on Truffle language performance.
While Truffle interpreters are written in Java and compile the same as any other Java method would, Native Image does some extra work to record Graal graphs for Truffle interpreters so those interpreters can be JIT compiled.
In contrast, a trade-off when using Native Image is that there is no JIT for arbitrary Java methods.
The GraalVM team is working on a Java interpreter called [Espresso](https://www.graalvm.org/22.0/reference-manual/java-on-truffle/interoperability/) that will allow Java methods to JIT in Native Image binaries by running the bytecode through the Espresso interpreter, but I did not consider it for any part of the Native Image Playground.
The reason I'm calling this out specifically is because I'm not measuring the call overhead of Java methods being run in a Native Image binary.
Certainly, I need to make some Java calls to use the Truffle polyglot API, but what I'm really concerned with is the performance of executing guest code in a Truffle interpreter.

#### Methodology

For benchmarking, I'm using Google's [benchmark](https://github.com/google/benchmark) library in a launcher written in C++.
I.e., the benchmark harness is not a Native Image binary.
The benchmarks were run on a Ryzen 3700X system with 64 GB ECC RAM running Ubuntu 21.10 (kernel 5.15.0-6-generic) and with CPU frequency scaling disabled.
To help avoid differences related to benchmark execution order, benchmark results were collected using the `--benchmark_enable_random_interleaving=true` option from the Google Benchmark library.

Much like the examples used to demonstrate the various ways to embed Truffle languaages, the benchmarks all compute the Haversine distance.
There is a Haversine implementation in C++ intended to be something of a control value.
Likewise, there's an Haversine implementation in Java to establish the baseline for methods compiled by Native Image.
From there, all of the other benchmarks call into a Truffle language to calculate the Haversine distance.

Software versions:

  * LLVM: 13.0.0
  * C++ Compiler: clang++ (Ubuntu clang version 13.0.0-2)
  * Google Benchmark: 705202d22a10154ebc8bf0975e94e1a93f08ce98
  * GraalVM: 22.0.0.2 (based on Java 17.0.2) - Community Edition

#### Classifications

##### Baseline

The benchmark runner includes an implementation of the Haversine distance written in C++ using trigonometric functions from cmath/math.h from the C/C++ standard library as shipped with LLVM.
The Haversine distance implementation is a direct port of the Apache SIS implementation used in Java.
While I don't doubt the algorithm could be tweaked more manually, the implementation is quite straightforward and compact.
A large component of these benchmarks is to see how well a compiler is able to optimize code.
Since the Native Image builder will perform optimizations when building its binary, the benchmark runner, including the C++ Haversine implementation, is compiled with the `-O3` optimization flag.

```
---------------------------------------------------------------------------------------------
Benchmark                                                   Time             CPU   Iterations
---------------------------------------------------------------------------------------------
C++                                                      50.8 ns         50.7 ns     13773699
```
<div class="caption"><caption>Figure 1: Benchmark baseline number.</caption></div>

##### Native Image C API

There are three types of benchmarks run with the Native Image C API:

  1. A pure Java implementation of the Haversine distance
  2. A hard-coded Ruby implementation of Haversine distance
  3. A generic executor of Truffle language code, which is supplied with implementations of the Haversine distance in Ruby and JavaScript.

I chose these three to help establish where overhead may lie.
I expect the pure Java version to optimize the best during AOT compilation; however, it will not JIT compile.

The hard-coded Ruby implementation uses the Truffle polyglot API to execute a predetermined code fragment with TruffleRuby.
The code fragment can be parsed ahead-of-time and the resulting function object stored directly into the Native Image shared library, avoiding the need to parse the code at runtime.
Since the requirements of tha code fragment are known ahead of time, the implementation can make the exact Truffle polyglot calls needed to execute the Ruby Haversine code.
While somewhat limited, this example is illustrative of how you might embed a language like Ruby within a process to run specific workloads.

The final benchmark also makes use of the Truffle polyglot API, but it accepts arbitrary code fragements.
Well, in this case, it's not exactly arbitrary.
The benchmark harness does provide the code fragment to execute and does so for both JavaScript and Ruby implementations of the Haversine distance algorithm.
However, I took something of a short-cut for the execution of the code.
The Truffle polyglot API types all results as `org.graalvm.polyglot.Value`, but this is not a type that Native Image can work with out-of-the-box in a `@CEntryPoint` method.
As a simplification, the exposed method always returns `double`.
Likewise, the four `double` coordinates would need to be packed into an `Object[]` in order to make the execution method truly arbitrary, but our exposed function takes four `double` arguments and passes them through to the Truffle polyglot call.
I went with this short-cut as a matter of practicality: 1) I wanted to see what the overhead would be of executing essentially the same code, but being evaluated at runtime rather than being baked into the Native Image library; and 2) the effort required to make this approach truly general is so involved that I can't believe anyone would do it in practice.<a href="#footnote_3"><sup>3</sup></a>

###### Results

The results of the two simple `@CEntryPoint` methods &mdash; numbers 1 and 2 from the previous section's benchmark description &mdash; are available in Fig. 2.

```
---------------------------------------------------------------------------------------------
Benchmark                                                   Time             CPU   Iterations
---------------------------------------------------------------------------------------------
C++                                                      50.8 ns         50.7 ns     13773699

# Benchmark 1
@CEntryPoint: Java                                        120 ns          120 ns      5795027

# Benchmark 2
@CEntryPoint: Ruby                                        155 ns          155 ns      3312553
```
<div class="caption"><caption>Figure 2: Benchmark results for methods exposed via @CEntryPoint (Native Image C API).</caption></div>

The Java Haversine implementation executes in 120 ns, compared to the 51 ns of the C++ implementation, taking 2.4x times as long to execute.
Interpreting that result requires contextualizing your application.
On the one hand, if performance is your ultimate goal, the C++ implementation is twice as fast as the Java one compiled with Native Image.
On the other hand, the Native Image implementation includes a fully functional virtual machine with memory safety, garbage collection, platform API abstraction, and so on.
If the overall performance is within your application's performance target, Native Image can be a compelling option for generating native binaries while benefiting from the Java ecosystem.
Either way, this example is not telling the whole story and results should not be extrapolated.
There isn't much going on in terms of memory allocation, I/O, multi-threading, or even functionality like virtual calls and templating/generics.
I'd encourage you to run your own benchmarks flexing the functionality your application would require.

With all of that said, from here on out I'm going to use the Java implementation of the Haversine algorithm as the baseline.
The differences between Java and Truffle languages are smaller than the differences between C++ and Java and that detail would be difficult to see if the ensuing analysis focused on Truffle languages vs C++.

Fig. 3 adds in the results from the third benchmark outlined in the previous section.
Results are provided for both TruffleRuby and Graal.js in order to provide Truffle polyglot API results that don't overfit to a particular language.
To help give a better sense of the benefits of Truffle caching, multiple results are presented:

<ol type="a">
  <li>No caching at all (try-with-resources pattern; context is local and code fragments always parsed at runtime)</li>
  <li>Truffle context re-used, but code fragments always parsed at runtime</li>
  <li>Truffle context re-used and evaluated code fragments parsed (thread-safe)</li>
  <li>Truffle context re-used and evaluated code fragments parsed (thread-unsafe)</li>
</ol>

```
---------------------------------------------------------------------------------------------
Benchmark                                                   Time             CPU   Iterations
---------------------------------------------------------------------------------------------
# Benchmark 1
@CEntryPoint: Java                                        120 ns          120 ns      5795027

# Benchmark 2
@CEntryPoint: Ruby                                        155 ns          155 ns      3312553

# Benchmark 3a
@CEntryPoint: Polyglot (JS)                           1351136 ns      1350841 ns          473
@CEntryPoint: Polyglot (Ruby)                        70609504 ns     70523737 ns           10

# Benchmark 3b
@CEntryPoint: Polyglot (JS) - No Parse Cache             4058 ns         4055 ns       125009
@CEntryPoint: Polyglot (Ruby) - No Parse Cache           3461 ns         3458 ns       163807

# Benchmark 3c
@CEntryPoint: Polyglot (JS) - Safe Parse Cache           2144 ns         2142 ns       300847
@CEntryPoint: Polyglot (Ruby) - Safe Parse Cache         1926 ns         1926 ns       326447

# Benchmark 3d
@CEntryPoint: Polyglot (JS) - Unsafe Parse Cache         1497 ns         1496 ns       392821
@CEntryPoint: Polyglot (Ruby) - Unsafe Parse Cache       1339 ns         1338 ns       429389
```
<div class="caption"><caption>Figure 3: Benchmark results for Truffle polyglot methods exposed via @CEntryPoint (Native Image C API).</caption></div>

The case where the Ruby code can be parsed ahead of time (benchmark _2_) is not quite as fast Java, but probably much closer than people would think, taking 1.3x as long to execute.
The two `@CEntryPoint` methods that take an arbitrary language and code fragment to execute fare much worse, taking 10.8x as long to execute in the unsafe Ruby version and 15.8x as long in the thread-safe implementation.
While a fair bit of the 
For this particular case, the Ruby code is outperforming the JavaScript implementation.

Benchmarks _3a_ - _3d_ execute an exposed Native Image function that takes both a Truffle language identifier and a code fragment to execute.
These values are supplied at runtime, so there's no ability to parse them ahead-of-time and snapshot them into the image as benchmark _2_ could.
The results for benchmark _3a_ clearly illustrate why re-using a Truffle context is important.
Without reusing a Truffle context, each Truffle language must bootstrap itself each time a context is used<a href="#footnote_4"><sup>4</sup></a>.
Ruby has a much larger core library than JavaScript does, so there is a lot more to do to bootstrap the interpreter, leading TruffleRuby to have a per-call execution time that's an order of magnitude slower than Graal.js.
Moreover, by starting from scratch each time, Graal is never able to JIT the Truffle language code.

Benchmarks _3b_ - _3d_ all reuse a Truffle context throughout the life of their benchmarks.
Benchmark _3b_ parses the Haversine distance code fragment each time the benchmark runs.
Benchmark _3c_ uses a thread-safe parse cache, parsing the Haversine distance code fragment once and reusing the resulting Truffle function object for subsequent calls.
Benchmark _3d_ does essentially the same things as _3c_, but does away with the overhead of a `ConcurrentHashMap`.
Of the three, I think the thread-safe parse cache (_3c_) is the one to go with.
I included results with no parse cache (_3a_) and no thread-safety (_3d_) in order to see what the overhead is for using a thread-safe cache.
Once the code is bootstrapped and JIT'd, the performance difference between the Ruby and JavaScript closes considerably, with TruffleRuby eking out Graal.js by a small amount.


##### JNI Invocation API

In order to easily compare results between the Native Image C API and the JNI Invocation API, the same workloads were tested with each API.
With the JNI Invocation API, we have considerably more control over re-using the Graal Isolate, Truffle context, and parsed guest language functions.
Consequently, the JNI benchmarks do not explore different Truffle caching strategies as we did with the Native Image C API benchmarks; we just use the most straightforward implementation, which happens to be the most performant.
<!--
The benchmarks take advantage of the more granular controls afforded to it so we can see the realistic trade-off between the two APIs.
The goal isn't necessarily to measure the call overhead, which would be better accomplished by using the same parse cache strategy for both APIs.
-->

###### Results

The results of the two simple JNI methods &mdash; numbers 1 and 2 from the previous section's benchmark description &mdash; are available in Fig. 4.

```
---------------------------------------------------------------------------------------------
Benchmark                                                   Time             CPU   Iterations
---------------------------------------------------------------------------------------------
C++                                                      50.8 ns         50.7 ns     13773699

# Benchmark 1
JNI: Java                                                 136 ns          136 ns      4961384

# Benchmark 2
JNI: Ruby                                                 155 ns          155 ns      3244096
```
<div class="caption"><caption>Figure 4: Benchmark results for methods exposed invoked via JNI Invocation API.</caption></div>

As with the Native Image C API results, we start by looking at the performance the Java implementation of the Haversine distance algorithm.
At 136 ns, the Java implementation takes 2.7x as long to execute as the C++ implementation (51 ns).
As noted in the Native Image C API benchmark results, there's a natural trade-off between executing the C++ version and Java version, as the latter has a supporting virtual machine.
It's important to know that there is a performance difference and what that is, but that should be evaluated in context of your functional requirements.
The Haversine distance algorithm is a computation-heavy benchmark; you should establish your own representative benchmarks if you're trying to decide between a systems language and Native Image for a particular task.

Having established the performance difference between the C++ Haversine implementation and the Java implementation compiled with Native Image, I'll be using the Java implementation as the baseline for the Truffle benchmarks.
Ultimately, I'm interested in embedding Truffle languages into another process.
While the C++ implementation helps establish an ideal performance target, I plan on using Native Image so and so seeing performance of Truffle languages relative to Java, with everything compiled in the Native Image binary, is a more useful comparison.

```
---------------------------------------------------------------------------------------------
Benchmark                                                   Time             CPU   Iterations
---------------------------------------------------------------------------------------------
C++                                                      50.8 ns         50.7 ns     13773699

# Benchmark 1
JNI: Java                                                 136 ns          136 ns      4961384

# Benchmark 2
JNI: Ruby                                                 155 ns          155 ns      3244096

# Benchmark 3
JNI: Polyglot (JS)                                        548 ns          548 ns      1268457
JNI: Polyglot (Ruby)                                      311 ns          311 ns      2091522
```
<div class="caption"><caption>Figure 5: Benchmark results for Truffle polyglot methods invoked via JNI Invocation API.</caption></div>

As with the `@CEntryPoint` approach, the case where the Ruby code can be parsed and snapshotted into the Native Image binary (benchmark _2_) is nearly as fast (1.1x) as the Java implementation.
The Truffle polyglot calls (benchmark _3_), which take both a Truffle language ID and a code fragment to evaluate at runtime performed, naturally were slower.
The Truffle polyglot call with the Ruby implementation of the Haversine distance took 2x as long to execute as the version where the Ruby code could be compiled at Native Image build time and 2.3x as long as the Java implementation.
The JavaScript implementation was considerably slower than the Ruby one when using Truffle polyglot calls.

##### Overall

The previous sections looked at the performance of different ways to execute code compiled into a Native Image shared library from an external process, with an emphasis on executing code in a Truffle interpreter.
There are two primary invocation APIs for exposing Java methods and making them accessible via C calling conventions: the Native Image C API and the Java Native Interface Invocation API.
Thus far we've looked at the relative performance difference of executing methods written in C++, Java, Ruby, and JavaScript in each of these invocation APIs.
In Fig. 6, we can now see how the invocation APIs perform relative to each other.

```
---------------------------------------------------------------------------------------------
Benchmark                                                   Time             CPU   Iterations
---------------------------------------------------------------------------------------------
C++                                                      50.8 ns         50.7 ns     13773699

# Benchmark 1
@CEntryPoint: Java                                        120 ns          120 ns      5795027
JNI: Java                                                 136 ns          136 ns      4961384

# Benchmark 2
@CEntryPoint: Ruby                                        155 ns          155 ns      3312553
JNI: Ruby                                                 155 ns          155 ns      3244096

# Benchmark 3
@CEntryPoint: Polyglot (JS) - Safe Parse Cache           2144 ns         2142 ns       300847
@CEntryPoint: Polyglot (Ruby) - Safe Parse Cache         1926 ns         1926 ns       326447
JNI: Polyglot (JS)                                        548 ns          548 ns      1268457
JNI: Polyglot (Ruby)                                      311 ns          311 ns      2091522
```
<div class="caption"><caption>Figure 6: Benchmark results for Truffle polyglot methods invoked via both the Native Image C API and the JNI Invocation API.</caption></div>

As I mentioned much earlier in this post, a refrain I've heard repeatedly in the GraalVM community is that the JNI Invocation API is prohibitively slow for calling exposed functions in a Native Image shared library.
I was keen to embed a Truffle language into another process, but which API to use wasn't clear.
While the JNI Invocation API is the GraalVM team's stated path forward, I also need to solve today's problems with the tooling I have at hand, not some hypothetically better version to come.

With that said, the only benchmark where the Native Image C API comes out on top is the one with the Java implementation of the Haversine distance algorithm.
This benchmark evaluated a computation-heavy method written entirely in Java.
I did not run this code with the Espresso Truffle implementation of Java, so there was no JIT compilation of the code.
With this benchmark, we see that there is ~13% overhead in using the JNI Invocation API (take that with a grain of salt: I don't have error bounds for these values, although I did run multiple times and results were fairly consistent).
Based on what I had been reading, I was expecting a much larger performance difference.
I'm sure for some applications, > 10% performance hit across the board is untenable.
And to be fair, that performance gap may be larger with functions that take more arguments or varargs or have non-primitive parameters or any of the variations in making a foreign call that are *not* benchmarked here.

When calling Truffle languages, however, the performance numbers start flipping the other way.
In the case where the Ruby code could be parsed and snapshotted into the image, there was essentially no performance difference at all between the Native Image C API and the JNI Invocation API.
Once we started executing code using the Truffle polyglot API with values that were supplied at runtime, the JNI Invocation API handily outperformed the Native Image C API.
The latter's approach to calling into the TruffleRuby interpreter via the Truffle polyglot API took 6x as long to execute as the JNI Invocation API.

<!--
If calling a function in a Native Image shared library incurs more overhead with JNI than the Native Image C API, that overhead is negated by the overall performance gain of working with a Truffle language via JNI.
In the benchmarks where the Truffle language code to be executed is embedded directly in the compiled Java method, the JNI implementation as fast as or faster than the Native Image C API implementation.
When calling into the Native Image library using the Truffle polyglot API, allowing for flexible execution of arbitrary guest language code, the JNI approach is an order of magnitude faster.
-->

If your goal is to embed Truffle languages into another process, using the JNI Invocation API is the clear winner.
While the API is more difficult to work with, it is also considerably more flexible.
That is ultimately why it outperforms the Native Image C API implementations using the Truffle polyglot API.
In order to execute Truffle code within a Native Imaged shared library, the caller must construct a Graal Isolate and a Truffle context.
Creating a Graal Isolate was covered elsewhere in this post and both the Native Image C API and JNI Invocation API have mechanisms for managing Graal Isolates.
Working with the Truffle context is more complicated, however.

Many of the examples using the Truffle polyglot API use a try-with-resources statement to create a Truffle context, which is then used for initializing a Truffle language engine and ultimately executing a fragment of code.
While you can use this pattern in any `@CEntryPoint` function, you will lose any profiling information or compiled code when the context is closed.
If you're executing long-running code, that may be fine.
However, if you're looking to embed a Truffle language to execute small, short-lived snippets of code and call that `@CEntryPoint` method repeatedly (e.g., as part of a web request), performance will suffer greatly by not sharing the context across all invocations.
Sharing the context so you can parse the code snippet once and execute that repeatedly is the way to go, but that resource management is rather complicated.

When you write a `@CEntryPoint` using try-with-resources for managing the Truffle context, you have a completely self-contained function you can call from C.
When you want to share the context, you now need a mechanism for creating the context, passing it to all of your `@CEntryPoint` methods using the Truffle polyglot API, and a way to clean up the context when you're done with it.
In contrast to managing Graal Isolates, the Native Image C API does not provide a way to manage Truffle contexts with special API methods.
Instead, you would need a way to create the Truffle context in Java, return it in a call to `@CEntryPoint` method, then have the ability to pass it to other `@CEntryPoint` calls.
This is where things get very complicated very quickly.

The functions that Native Image can expose in a shared library only support a narrow range of types for parameters and return types.
You might think that you could pass arbitrary Java objects around as `void *`, treating them as opaque values to be passed around.
However, Java is a garbage collected language and that presents problems.
If the GC were to free an object you still have a pointer to, you would have a use-after-free problem if you ever used that pointer again.
An equally bad situation is if the GC moves the object, since there would be no way to update the calling process.
To help in this situation, you can _pin_ an object with the Native Image API, which will prevent the object from GC consideration.
However, this should be done sparingly and is intended for ensuring an object doesn't in a very narrow window; long-term storage of an object's address is not recommended as it will have an impact on GC.
Moreover, you won't find documentation on pinning objects.
You will be decidedly in unsupported territory.

Another way to share a Truffle context across multiple calls of `@CEntryPoint` is to store the context in a static field.
Doing so, however, introduces thread safety issues.
This is the approach I took for the benchmarks since I was unable to get object pinning to work correctly.
For the polyglot cases, the caller supplies both the Truffle language identifier and the code to evaluate.
Since parsing the code on each call would incur overhead, I set up a parsed code cache, keyed by the language ID and code.
The code fragments for the Haversine distance were all implemented as anonymous functions, so the cache also ensures that we create an anonymous function once and call it repeatedly so that it can JIT.<a href="#footnote_5"><sup>5</sup></a>

The parse cache, while effective at helping the code fragment JIT, incurs overhead that adversely impacts the benchmark.
To ensure thread-safety, I used a `ConcurrentHashMap` for the parse cache.
I think this accurately reflects what a conscientious developer would need to do to support multiple calls with multiple code fragments.
To further assess that impact, I replaced the `ConcurrentHashMap` with a single static field.
For this benchmark, the cache does not support executing different code fragments and only uses a simple `null` check to determine whether to parse again &mdash; this almost certainly will result in multiple callers writing to the cache, but since the code fragment is idempotent, the wasted computation is acceptable and the cache will stabilize during benchmark warm-up.


Conclusion
----------

This turned out to be a much bigger project than I anticipated.
Unfortunately, the various mechanisms for exposing Java methods in a Native Image shared library and then calling into those methods were not easy to discover.
I found after the fact that some things that I thought were not documented actually were, but I didn't know which set of keywords to use or where in the GraalVM reference manual to find them.
Other things I needed to discover by digging into the Native Image source code.
And to the best of my knowledge, no one has published comprehensive benchmarks on either the Native Image C API or the JNI Invocation API within a Native Image binary.

Consequently, this blog post has turned out to be much larger than I anticipated.
I wanted to document how to use both invocation APIs for calling into a Native Image shared library.
Moreover, I wanted to show how to work effectively with Truffle languages embedded in other applications.
And to top it all off, I wanted to show the performance trade-offs between both the invocation APIs and the various ways of calling into a Truffle interpreter.

The [Native Image Playground](https://github.com/nirvdrum/native-image-playground) is home to self-contained examples that demonstrate everything discussed in this post, including the benchmarks.
Please see the "Native Image Overview" section for more details on how to work with the project.

Based on my evaluation, I'll be using the JNI Invocation API for all of my Truffle embedded work.
It's the API that the GraalVM team has signaled would be the future for foreign calls into a Native Image shared library and it's the fastest invocation API for calling into Truffle interpreters.
Unfortunately, working with the Truffle polyglot API with JNI is very awkward.
It requires building an extra configuration file to supply to the `native-image` compiler at image build time.
That extra configuration file is a reflection of all the class and method mapping you'll need to do in order to make a Truffle call with JNI, which requires learning how to map Java types to their string-based type signatures.
It's incredibly easy to make silent errors with JNI because Java exceptions don't bubble up.
Instead, you need to make a call to check if an exception was raised and then decide what to do with it.
As is typical in C error handling, you need not do this error-checking, but if you don't and there is an error, you'll probably just get a `nullptr` somewhere with very little help in finding the cause.

I think all of this could be addressed with a C-based wrapper API that takes care of the underlying JNI calls.
Until then, please look at the _jni-native_ project in the Native Image Playground for an example how to call into a Truffle language via JNI and check out the "Java Native Interface (JNI) Invocation API" section for details on how to use the GraalVM Tracing Agent to help generate the JNI configuration file that the `native-image` binary requires.


<hr />

<a name="footnote_1"></a>
<sup>1</sup>
<small>
Native Image is unable to JIT Java code because there is no Java bytecode to profile in the compiled binary.
However, Truffle-based languages can JIT because Graal compiler graphs corresponding to the language's Truffle interpreter are compiled into the image.
Getting a bit meta, there's an implementation of a Java bytecode interpreter in Truffle called [Espresson](https://www.graalvm.org/22.0/reference-manual/java-on-truffle/) in development.
Since Espresso uses a Truffle interpreter, it will be able to JIT and it is expected that will be the way forward for JITting Java applications in a Native Image binary.
</small>

<a name="footnote_2"></a>
<sup>2</sup>
<small>
While a Truffle interpreter can be compiled to a native binary with Native Image, an application using a Truffle language cannot as of yet.
E.g., you cannot write a CLI application in Ruby and compile the application into a native executable.
Instead, you'd use a compiled interpreter such as TruffleRuby to load and run your script.
</small>

<a name="footnote_3"></a>
<sup>3</sup>
<small>
For an approximation of the effort involved, please look at the [_native-polyglot_ launcher](https://github.com/nirvdrum/native-image-playground/blob/15042578609535aa37ca68be6c051be23b6c8066/src/main/c/native-polyglot/native-polyglot.c) implementation in the Native Image Playground.
The _native-polyglot_ example uses the Native Image Polyglot API &mdash; a wrapper around the Native Image C API used for Truffle polyglot calls in C.
This approach to embedding Truffle languages in another process is not benchmarked because it's so similar to the Native Image C API.
Additionally, the Native Image Polyglot API is deprecated and should not be used for new objects.
It exists in the Native Image Playground solely for completeness.
</small>

<a name="footnote_4"></a>
<sup>4</sup>
<small>
Truffle languages can pre-bootstrap an interpreter and snapshot that into the Native Image.
The Truffle languages from the GraalVM team do precisely that to varying degrees.
Whatever can't be snapshotted during the Native Image building process must be executed at run time.
</small>

<a name="footnote_5"></a>
<sup>5</sup>
<small>
As a practical matter, anonymous functions allows us to evaluate the same code snippet multiple times without state conflicts.
In a shared context, all executed code fragments share the same global state.
If you were to parse and run the JavaScript snippet `const EARTH_RADIUS = 6371` each time you called the `@CEntryPoint` function with a shared context, you would get an error about attempting to redefine a constant.
There are ways to work around this, of course.
In Ruby, you can check if a constant is already defined before defining it in your code snippet.
In our embedded examples, we could make multiple calls to the Truffle polyglot API, ensuring function definitions are only executed once, while function calls may happen in th benchmark loop.
Using anonymous functions allowed for flexibility in how the code is evaluated; it works fine with or without a parse cache and is wholly contained (i.e., it does not require two separate calls to define and then call a function).
</small>