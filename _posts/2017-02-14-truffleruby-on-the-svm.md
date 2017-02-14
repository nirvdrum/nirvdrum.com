---
layout: post
title: TruffleRuby on the Substrate VM
author: Kevin Menard
draft: true
---

Introduction
------------

In the TruffleRuby [status report for 2016](http://lists.ruby-lang.org/pipermail/jruby/2017-January/000511.html), we indicated that we planned to address our startup time problem in the coming year.
We're well aware that one of the biggest challenges facing alternative Ruby implementations is being competitive with MRI on startup time and memory consumption.
While we have established a peak performance advantage over other implementations in many benchmarks, we haven't had a very impressive story to tell for short-lived applications such as one-off Ruby scripts and test suites.

I'm extremely pleased to say that we're making good on that promise.
Starting with GraalVM 0.20, we're now shipping a new virtual machine and ahead-of-time compiler that can produce static binaries of Truffle-based languages.

Glossary
--------

There's a lot of terminology involved in discussing TruffleRuby.
If you've been following our project status you might already be familiar with them, but I assume many people are not.
In order to make the remainder of this post easier to understand, I've provided a brief glossary below:

Graal
: An optimizing compiler for the JVM written in Java with hooks available to interact with it from Java.

Truffle
: A self-optimizing AST interpreter framework for Java. When paired with Graal it is able to perform partial evaluation on the interpreter and user program to produce tight machine code for the running application.

TruffleRuby
: An implementation of Ruby based on Truffle. Since it uses Truffle, the core of it is authored in Java, but much of the runtime is written in Ruby.

GraalVM
: A distribution containing a Graal-enabled JVM and runtimes for Truffle-based languages (currently TruffleRuby, Graal.js, and FastR) from Oracle Labs.

Performance
-----------

Isolating startup time from program execution can be a bit tricky.
Rather than getting mired in the details, I've taken the measurement<sup>1</sup> of an extremely simple program: `ruby -e 'p "Hello, world"'`.
If you want to follow along, simply [install GraalVM](https://github.com/graalvm/truffleruby/blob/master/doc/user/using-graalvm.md) and build the [TruffleRuby binary](https://github.com/graalvm/truffleruby/blob/master/doc/user/svm.md).

|                      | Real Time (s) | Max RSS (MB) |
|----------------------|:-------------:|-------------:|
| TruffleRuby SVM 0.20 | 0.24          | 128.6        |
| TruffleRuby JVM 0.20 | 3.43          | 439.6        |
| JRuby 9.1.7.0        | 1.55          | 200.4        |
| Rubinius 3.69        | 0.27          | 71.1         |
| MRI 2.4.0            | 0.05          | 8.8          |

Running TruffleRuby on the Substrate VM is 13 times faster than running on the JVM while only using 30% as much memory.
We still have a ways to go before we catch up to MRI's startup time, but TruffleRuby on the Substrate VM starts up faster than all other alternative Ruby implementations.
It's unlikely we'll ever approach MRI's memory consumption because we must retain runtime metadata for the Graal compiler, but we should be able to whittle it down further and run well on memory-constrained cloud servers.

Turning our attention to a more real world application, I ran the set of language specs from the [Ruby Spec Suite](https://github.com/ruby/spec).
These specs look and run very similarly to a typical application's test suite.
I removed JRuby and Rubinius from this evaluation because they don't pass the same set of specs that TruffleRuby & MRI do, confounding the results.

|                                  | Real Time (s) | Max RSS (MB) |
|----------------------------------|:-------------:|-------------:|
| TruffleRuby SVM 0.20             | 9.00          | 1,364.1      |
| TruffleRuby JVM<sup>2</sup> 0.20 | 68.38         | 560.8        |
| JRuby<sup>3</sup> 9.1.7.0        | 37.57         | 380.6        |
| Rubinius<sup>3</sup>  3.69       | 7.18          | 112.6        |
| MRI 2.4.0                        | 1.01          | 13.7         |

Test suites like this are generally hard on optimizing runtimes.
They always start in a cold state, they run for a short period of time, and in the case of JVM-based languages they incur a high degree of overhead when spawning new processes.
For this test suite, which has 2,121 specs and 3,824 assertions, TruffleRuby on the SVM is 6.6 times faster than TruffleRuby on the JVM, shaving almost a full minute off the test suite &mdash; a fairly substantial savings.
However, this is a relative performance reduction from what we saw with the simple startup test, suggesting we have additional opportunities to reduce total run time.

Unfortunately, this test suite illustrates a current memory problem when many objects are created.
The SVM uses a different garbage collector (GC) than the JVM and hasn't had the same amount of time to mature.
In particular, the SVM doesn't currently have support for [compressed oops](http://docs.oracle.com/javase/7/docs/technotes/guides/vm/performance-enhancements-7.html#compressedOop) which doubles the cost of object pointers in many applications.
This is really just a matter of time rather than an inherent limitation.
Additionally, we haven't tuned TruffleRuby for the SVM's GC.
We'll be investigating this result and will get it fixed for future releases.

We still have work to do before we catch up to MRI, but we're now in the same ballpark.
I think with TruffleRuby on the SVM we've moved from "absolutely not" to "not ideal, but acceptable" when deciding whether to use TruffleRuby for daily tasks.
I know that doesn't sound like a ringing endorsement, but it's a huge hurdle to clear.
I've seen a lot of interest in TruffleRuby that ends up being tempered by startup time concerns.
I think the release of TruffleRuby on the SVM demonstrates our approach to solving the problem is viable.

Keep in mind this is the same TruffleRuby codebase running on both virtual machines.
We essentially get this performance gain for free by implementing the language with Truffle!


More about the Substrate VM
---------------------------

The Substrate VM is a project under the GraalVM umbrella at Oracle Labs, headed by Christian Wimmer.
The basic idea behind the SubstrateVM is to provide a new distribution mechanism for languages offered in Truffle.
The Truffle DSL and framework are Java-based, which means languages wishing to make use of Truffle must also be authored in Java.
Authoring a language, such as Ruby, in a high level language like Java brings many advantages, such as excellent IDEs, refactoring capabilities, performance &amp; analysis tools, and a codebase that's easy to maintain.
However, it also brings with it disadvantages such as slower startup time, increased memory usage, and distribution difficulties.

The SubstrateVM aims to address the disadvantages while preserving the good stuff.
By stripping out the JDK classes not needed by a guest language, performing up-front optimizations during ahead-of-time compilation, and retaining the metadata necessary for Graal to perform online optimization of a guest program, SubstrateVM is able to produce a compact static binary with fast startup time and reduced memory overhead.
This is a powerful new addition to the Truffle toolchain.
In addition to an optimizing JIT, GC, instrumentation (profiler &amp; debugger), and built-in polyglot support, by implementing your language on top of Truffle you now get an ahead-of-time compiler.

There are trade-offs to targeting the Substrate VM.
In order to perform its static analysis, all code to be included in the binary must be statically reachable.
Consequently, reflection and dynamic class loading can't be used.
Additionally, arbitrary native code cannot be called.
The Substrate VM does ship with built-in support for [JNR POSIX](https://github.com/jnr/jnr-posix) to make native POSIX calls easier.
And, as the Substrate VM is still a young project, certain APIs from the JDK might not yet be supported.

For code you have control over, these restrictions might be inconvenient, but are generally manageable.
However, pulling in arbitrary 3rd party code can create headeaches since it's not quite as easy to avoid paths that don't comply with the SVM's restrictions.
The SVM does include a mechanism to replace a method implementation at runtime, much like aspect-oriented programming, but this should be used a last resort measure.


Conclusion
----------

The Substrate VM provides Truffle-based languages, such as TruffleRuby, with an incredible new way to deliver a language runtime.
If you've been following the TruffleRuby project, you likely know we've been talking about SVM for the past couple years as our solution to solving our startup time problem.
I'm excited to say it wasn't vaporware!
The results for TruffleRuby on SVM thus far are extremely promising and I think the release of SVM marks a new evolutionary phase of the TruffleRuby project.



Additional Resources
--------------------

If you're interested in learning more about the history of TruffleRuby or get a deeper look at its internals, please take a look at the [collection of resources](http://chrisseaton.com/rubytruffle/) Chris Seaton has assembled.
Truffle and Graal are active research projects with a rich set of publications <sub>([Truffle](https://github.com/graalvm/truffle/blob/master/docs/Publications.md), [Graal](https://github.com/graalvm/graal-core/blob/master/docs/Publications.md))</sub>.
If you're interested in implementing a language on Truffle, I suggest checking out the [SimpleLanguage implementation](https://github.com/graalvm/simplelanguage) and watching Christian Wimmer's [walkthrough of the code](https://www.youtube.com/watch?v=FJY96_6Y3a4).

<hr/>

<sup>1</sup>
<small>
  All measurements were taken on an i7-4930K running at 4.1 GHz and 48 GB RAM.
  The operating system was Ubuntu 16.04.1 with a 4.4 Linux kernel.
</small>

<sup>2</sup>
<small>
  Due to a bug in GraalVM 0.20, the Ruby Spec Suite language specs do not run with runtime compilation enabled.
  For this evaluation I ran with a stock JVM, while the startup tests report the JVM with Graal.
</small>

<sup>3</sup>
<small>
  Neither JRuby nor Rubinius pass 100% of the language specs from the Ruby Spec Suite.
  As a result, they error out on some specs that TruffleRuby and MRI pass.
  Since they're not executing the same code the recorded time and memory values shouldn't be taken as definitive.
</small>