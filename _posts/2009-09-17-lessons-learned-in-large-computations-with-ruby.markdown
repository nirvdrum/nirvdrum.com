---
layout: post
title: Lessons Learned in Large Computations with Ruby
author: Kevin Menard
---

Introduction
------------

This is the follow-up post to my [GitHub Contest Recap](http://nirvdrum.com/2009/09/03/github-contest-recap.html) post that I promised.  As mentioned, I submitted two entries for the GitHub contest, starting with Ruby and then rewriting in Java.  This post summarizes why I ultimately dropped Ruby in favor of Java for this particular task.  I apologize for its length, but it is divided into discrete sections and can be read in chunks without great loss of continuity.


Things that Worked Well with Ruby
---------------------------------

### Very quick and easy text processing ###

Overall, Ruby excelled at the tasks I knew it would excel at.  Principally, I was able to write fairly clean code that produced results in short order.  Its text processing capabilities are solid and were a big boon in processing the data files supplied as part of the contest.  Likewise, generating a results file was a trivial matter.

### Class re-opening ###

I generally shy away from monkey-patching unless absolutely necessary, but having the ability to do it easily is always nice.  I found myself working around limitations in [AI4R](http://ai4r.rubyforge.org/), a Ruby-based artificial intelligence library, and adding basic statistics functions to Array.  Being able to call `[x, y, z].mean` or `[t, u, v].sum` was an extremely concise way to represent terms in some of my equations.


Things that Did Not Work Well with Ruby
---------------------------------------

I had thought that I understood Ruby fairly well, but this project taught me otherwise.  Much like Java and the JVM, there are a lot of subtleties in Ruby I was either blissfully unaware of or haven't had to worry about in any great detail.  From the seemingly inane, such as the three different methods of object equality, to the surprising, such as the lack of a Float::Infinity constant. 

### Marshalling circular relationships ###

One of my earliest setbacks was related to marshalling.  Given that I was dealing with processed datasets and a large number of objects, I thought marshalling the data to and from disk would be an appropriate thing to do in order to save start-up time.  I had written my code such that a `Watcher` class and a `Repository` class maintained bi-directional references to one another.  By using a set for the associations, I avoided creating infinite loops when establishing the relationships.  However, the marshalling library does not know how to deal with circular references.  Thus, an attempt to marshal resulted in an infinite loop.  This seems quite odd to me as the problem of persisting a graph of objects is not intractable and thus I consider the limitation to be a bug in the marshalling library.

To be fair, Ruby does provide means of manually controlling marshalling, but I did not pursue this path.  Ultimately, I was using Ruby because it was supposed to make my development faster.  Getting drawn into the nuances of marshalling was something I didn't have time for and had no interest in doing.


### Creating large number of objects ###

An early design decision I made was to keep as much data in memory as possible.  This was because I was planning on running a large number of computations and wanted data access to be as quick as possible; faulting in from disk or DB would have been too slow.  A consequence of that decision was that in my very first pass at the program I had 750,000 objects in memory.  I never grew significantly beyond that because I managed to make the garbage collector in MRI segfault several times.  For about a week I tried to clean up my memory space: I dropped the bi-directional relationships between a watcher and a repository; I removed memoized calculations; and I did away with local copies of data at both the method and instance levels in favor of effectively global data.  Essentially, I tossed away any legibility my code once had.  In the end, however, the program was able to run without crashing the garbage collector, albeit several multitudes slower.  This prompted me to attempt profiling the application.

### Profiling with perftools.rb ###

My first attempt at profiling was with [perftools.rb](http://github.com/tmm1/perftools.rb), which uses Google's profiling tools.  I had [read good things](http://www.igvita.com/2009/06/13/profiling-ruby-with-googles-perftools/) about it from people that I generally hold in high regard.  Alas, I was unable to run it.  On both my MacOS X 10.5.7 laptop using prefixed Gentoo's Ruby (1.8.7p174) and Ubuntu 9.04's Ruby (1.8.7p72) the profiler segfaulted almost immediately.  I believe this had to do with the large object graph, but the error message wasn't terribly helpful and I had little interest in analyzing the coredump.


### Profiling with ruby-prof ###

Having failed to accomplish anything with profile.rb, I fell back to the venerable ruby-prof.  This profiler worked, but was extremely slow.  After several hours of running, I finally relented and sent it a SIGTERM.  I was pleasantly surprised to see that it maintained intermediary stats and output them upon application exit.  Unfortunately, it hadn't moved beyond my initial data loading code, so the reported information was of limited value.

### Memoizing at class method level ###

At one point my back-of-the-napkin calculation was that I was performing 13 MM point comparisons in [my instance space](http://nirvdrum.com/2009/09/03/github-contest-recap.html), many of them duplicating internal computations.  This was an easy target for optimization just through caching, so I decided to try out the memoize gem.  The general lack of code examples for it leads me to believe that it is not in widespread use.  All examples I found, including the one in "The Ruby Way," use the library outside of any class structure.  After searching news groups for a while and studying the source, I managed to get memoization going for instance methods on an object.  However, I was unable to figure out how to get the library to work on class methods without code modification; another irritating setback.

### Memcache only storing 1MB files ###

Having had issues with the MRI garbage collector and hoping to save costly calculations between data runs, I turned to the quintessential memory caching tool, memcached.  I was running version 1.4.0, which sports a new binary protocol for compressed communication.  Although I was connecting to localhost, I wanted to reduce latency as much as possible.  Unfortunately, the memcache-client library had not yet been updated to use the new protocol, forcing me to use the older, less efficient protocol.  

As it turned out, fretting over the application protocol was not time well spent; unbeknownst to me, memcache has a 1 MB limit on the size of an object being stored and I was storing rather large hashes.  Although the limit can be configured at compilation time, preliminary research indicated this was not a recommended practice.  Even if it worked, the memcache-client lib has the 1 MB limit hardcoded to avoid network traffic for large data, so the gem would also have to be patched and recompiled.


Alternative Ruby Implementations
--------------------------------

Throughout the course my Ruby implementation I used three of the major Ruby implementations.  I began with MRI (1.8.7p174) and moved to YARV (1.9.1p129) and ended up with JRuby 1.3.1 (1.8.6p287 compatible).  Most of the discussion up until now has focused on my use of MRI.  The following sections describe my experiences with alternative Ruby implementations.

### YARV ###

I was excited at the prospect of being able to use Ruby 1.9 for a project.  My contest entry had a small number of dependencies to cope with so I thought my potentials for failure were limited.  Generally, things seemed to work well with YARV.  I definitely saw a speed improvement over MRI.  However, the ruby debug gem has not been updated for 1.9.  Not having a debugger is not acceptable for development, so I had to abandon this path.

### JRuby ###

After having experienced issues with MRI & Ruby 1.9, JRuby ended up being my runtime of choice.  It was remarkably faster than MRI and did not suffer from the garbage collection issues.  Likewise, it had the full debugger support that Ruby 1.9 lacked.  However, there were several issues I ran into with JRuby.

#### Breaking the debugger with optimizations turned on ####

In order to maximize execution speed, I used the [--fast flag](http://kenai.com/projects/jruby/pages/PerformanceTuning#Using_JRuby_s_Fast_Mode), which performs some bytecode optimizations especially suited for `Fixnum` operations at the cost of compatibility with Ruby's stacktrace format.  Generally, this was acceptable because I could largely deduce what a problem was by unraveling the stacktrace by inspection.  The problem is that the Ruby debugger apparently relies quite heavily on its stacktrace format.  I was astonished by this finding, but I was unable to use the debugger until the `--fast` flag was removed, forcing me to have to bounce from one to the other.

#### Lack of profiling tools ####

Discovering the source of massive heap use was difficult due to the lack of profiling tools for JRuby.  None of the standard Ruby ones seemed to work at all.  I tried to use [YourKit for Java](http://yourkit.com/), which technically worked, but the class names did not match the Ruby ones and I think I was profiling more JRuby proper rather than my application.  Unable to make much use of the results, I abandoned the profiling effort and resorted to manual code analysis, something humans are notoriously bad at.

#### Debugger issues ####

I wrote most of my Ruby implementation using the [RubyMine](http://www.jetbrains.com/ruby/index.html) IDE.  RubyMine has an integrated graphical debugger, which worked fairly well.  I found that often, however, the debugger would detach itself from the process forcing me to start debugging from scratch.  It appears that this may be a problem with the ruby-debug-ide gem.  I noticed that the issue occurred more frequently with the 1.5 beta of RubyMine than it did with the stable 1.1.1 release, so I suspect it may have to do with RubyMine itself to some degree.  I also found it happened much more frequently with JRuby than it did with MRI.


The Switch to Java
------------------

Unhappy with the progress I had made with my Ruby-based contest entry, I made the decision to port the whole project over to Java.  It was something I was extremely remiss to do, but ultimately I wanted to see the performance impact would be.

### Static type checking ###

Dealing with generics in Java is an abomination, but static type checking was of enormous use when porting the Ruby code over to Java.  One of the biggest headaches I had in Ruby was related to my used of the property named "id."  When I had a rich object model, it wasn't a problem, but when I had to decompose that model I ran into all sorts problems because every object in Ruby has a default "id" property.  It became hard to keep track of what were domain model objects and what were simple Strings.  When porting over to Java, the distinction was quite clear and the IDE let me know immediately if I was using the wrong type.  Granted it was more verbose, but when running an application with long runtime characteristics, relying on runtime type checking can be a brutal cycle of run-wait-fix-repeat.

### Fast execution ###

My initial data load went down from about 2.5 minutes in JRuby (the fastest Ruby implementation I tried) to about 8 seconds in Java.  The gain was so great that I was convinced I had made a mistake somewhere.  Fortunately, I had a test suite to help indicate otherwise.

My best explanation for the difference in execution times is that Java must handle object creation much faster than JRuby can, despite both of them running on the JVM.  All this particular section of code did was iterate through the data files and create my watcher and repository representations.  That is to say, at this stage no "real work" had yet been performed.

### Handling large object graph easily ###

During my translation from Ruby to Java I migrated back to a rich object model.  Whereas I had to break down true bi-directional associations into hash lookups just to make my Ruby implementation run, Java could handle my full object graph quite easily.  As a result, the code was much more straightforward.  Additionally, I was able to cache values in memory judiciously without eating up the entire heap or breaking the garbage collector.  This helped in making the overall solution faster.

### Built-in thread pooling with producer / consumer model ###

In my Ruby implementation I wrote my own poor man's thread pool, used in conjunction with [JRuby's native thread pool](http://kenai.com/projects/jruby/pages/PerformanceTuning#Enabling_Thread_Pooling).  The former allowed me to do concurrent execution of point comparisons while the latter allowed me to reuse discarded native threads.  While my pool did work, I didn't abstract it very well and had to drain it frequently lest I ran out of stack space.  I'm still amazed I couldn't find anything in the Ruby standard library or even a widely used third party gem for thread pooling.

While constructing threads in Java is not as nice as in Ruby, Java 5 introduced a set of concurrency APIs that remove a lot of the pain with thread management.  Creating a thread pool and using a producer / consumer model backed by the pool was trivial to do.  It was a small thing, but it meant I didn't have to worry about managing the pool in my code and I had a high degree of confidence that the JDK's implementation was correct.  The JVM's native threads scheduling on multiple cores and the easy-to-use thread pool helped increase the throughput of my contest entry.

### Profiling tools ###

The JVM makes it very easy to profile a running Java application.  I used YourKit for Java on my entry early on when it felt slow.  I found a few problems areas and addressed them, allowing the program to run an order of magnitude faster.  This whole process didn't take much more than a couple hours in contrast to the many hours I spent, with minimal results, trying to do the same thing by inspection with my Ruby implementation.

### Debugger ###

Debugging applications in Java is quite simple.  The tools for doing so have been around a long time and are generally quite solid.  I used [IntelliJ IDEA](http://www.jetbrains.com/idea/index.html) as my primary development tool for the Java-based solution and relied on its graphical debugger for solving various problems.  Not having a debugger that disconnects from its process should be a given, but happened all too frequently with RubyMine & JRuby.

Conclusion
----------

After three days of porting, my first run of the Java implementation yielded a 36% prediction accuracy and ran in less than 10 minutes.  My best Ruby implementation, after weeks of development, only had a 31% prediction accuracy and took almost 7 hours to compute it.  With a few more hours of work the Java implementation broke 40% accuracy and ran in less than 3 minutes.  Unfortunately, at this point I became ill and by the time I recovered the contest had ended.

My takeaway from the project is that Ruby is a great language hampered by a terrible execution environment.  When writing Rails apps, this usually isn't a problem.  I'd even go so far as to say for most types of applications it shouldn't be a problem.  For anything that's heavily CPU bound or dealing with large object graphs, however, I don't think Ruby is a suitable option.  In these cases, Ruby may best serve as a rapid prototyping tool.  By writing the code in Ruby first I had a clear approach to use when translating to Java.

This was the first Java project I had written start to finish in a while.  Most of my Java work is on existing codebases.  I was pleasantly surprised to see how few issues I had with the [maven](http://maven.apache.org/) build system and it was nice to have a real profiler at my disposal; for much of my Ruby work I must rely on a hosted service for my profiling needs, which is less than ideal.  The Java language wasn't even that bad to write in.  I'm convinced that if there were easier syntax for collection access and proper language property support that writing in Java may even be a pleasant experience.

It seems Ruby and Java have become the two representatives for the dynamic versus static language debate.  It would be nice if some of the vitriol could be removed from the discussion and the two languages evaluated on their merits for particular tasks.  It should go without saying that every language has its place.  This exercise has given me a clearer understanding of where Ruby should and should not be used.  It also helped reassert to me what Java's strengths and weaknesses are.