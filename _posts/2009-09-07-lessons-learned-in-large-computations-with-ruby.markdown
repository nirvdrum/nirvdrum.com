---
layout: post
title: Lessons Learned in Large Computations with Ruby
---

Introduction
------------

This is the follow-up post to my [GitHub Contest Recap](http://nirvdrum.com/2009/09/03/github-contest-recap.html) post that I promised.  It details some of the technical problems I encountered while working on my contest entry and some of my thoughts on them.  I actually wrote two entries for the GitHub contest: one in Ruby and one in Java.  This post summarizes why I ultimately dropped Ruby in favor of Java for this particular task.


Things that Worked Well with Ruby
---------------------------------

### Very quick and easy text processing ###

Overall, Ruby excelled at the tasks I knew it would excel at.  Principally, I was able to write fairly clean code that produced results in short order.  Its text processing capabilities are solid and were a big boon in processing the data files supplied as part of the contest.  Likewise, generating a results file was a trivial matter.

### Class re-opening ###

I generally shy away from monkey-patching unless absolutely necessary, but having the ability to do it easily is always nice.  I found myself working around limitations in [AI4R](http://ai4r.rubyforge.org/), a Ruby-based artificial intelligence library, and adding basic statistics functions to Array.  Being able to call `[x, y, z].mean` or `[t, u, v].sum` is an extremely concise way to represent terms in some of my equations.


Things that Did Not Work Well with Ruby
---------------------------------------

I had thought that I understood Ruby fairly well, but this project taught me otherwise.  I suppose one of the biggest jabs at Java is that there seems to be a lot of cruft in the JDK that needs to be sifted through.  Additionally, a couple dozen methods and/or interfaces have seemingly arcane rules in their use that are not enforced by the compiler.  I still don't think I've ever seen anyone use a serialization ID appropriately.

Ruby is not devoid of these artifacts.

### Marshalling circular relationships ###

One of my earliest setbacks was related to marshalling.  Given that I was dealing with processed datasets and a large number of objects, I thought marshalling the data to and from disk would be an appropriate thing to do in order to save start-up time.  I had written my code such that a `Watcher` class and a `Repository` class maintained bi-directional references to one another.  By using a set for the associations, I avoided issues related to infinite loops resulting from a watcher adding a repository, which in turn added a watcher, vice versa.  However, the marshalling library does not know how to deal with circular references.  Thus, an attempt to marshal resulted in an infinite loop.  This seems quite odd to me as the problem of persisting a graph of objects is not intractable and thus I consider the limitation to be a bug in the marshalling library.

To be fair, Ruby does provide means of manually controlling marshalling.  I did not pursue this path, however.  Ultimately, I was using Ruby because it was supposed to make my development faster.  Getting drawn into the nuances of marshalling was something I didn't have time for and had no interest in doing.


### Creating large number of objects ###

An early design decision was to keep as much data in memory as possible.  This was because I was planning on running a large number of computations and wanted data access to be as quick as possible; faulting in from disk or DB would have been too slow.  A consequence of that decision is that in my very first pass at the program I had 750,000 objects in memory.  I never grew significantly beyond that because I managed to make the garbage collector in MRI segfault several times.  For about a week I tried to clean up my memory space: I dropped the bi-directional relationships already mentioned (incidentally fixing my marshalling issue); I removed memoized calculations; I did away with local copies of data; and I tossed any legibility my code once had.  In the end, the program was able to run without crashing the garbage collector, but was several multitudes slower, leading me to attempt profiling.

### Profiling with profile.rb ###

My first attempt at profiling was with profile.rb.  This uses Google's profiling tools and I had heard good things about it from people that I generally hold in high regard.  Alas, I was unable to run it.  On both my MacOS X 10.5.7 laptop using prefixed Gentoo's Ruby (1.8.7p174) and Ubuntu 8.10's Ruby (1.8.6pXXXXX) the profiler segfaulted almost immediately.  I believe this had to do with the large object graph, but the error message wasn't terribly helpful and I had little interest in analyzing the coredump.


### Profiling with ruby-prof ###

Having failed to accomplish anything with profile.rb, I fell back to ruby-prof.  This profiler worked, but was extremely slow.  After several hours of running, I finally relented and sent it a SIGTERM.  I was pleasantly surprised to see that it maintained intermediary stats and output them upon application exit.  Unfortunately, it hadn't moved beyond my initial data loading code, so the reported information was of limited value.

### Memoizing at class method level ###

At one point my back-of-the-napkin calculation was that I was performing 13MM point comparisons in [my instance space](http://nirvdrum.com/2009/09/03/github-contest-recap.html), each of which required a sized calculation for both terms.  This was an easy target for optimization, so I set my eyes on the memoize gem.  The general lack of examples for it leads me to believe that it is not in widespread use.  All examples I found, including the one in "The Ruby Way" use the library outside of any class structure.  After searching news groups for a while and studying the source for a bit, I managed to get memoization going for instance methods on an object.  I was unable to figure out how to get the library to work on class methods without code modification, another irritating setback.

### Memcache only storing 1MB files ###

Having had issues with the MRI garbage collector and hoping to save costly calculations between data runs, I turned to the quintessential memory caching tool, memcached.  I was running version 1.4.0, which sports a new binary protocol for compressed communication.  Although I was connecting to localhost, I wanted to reduce latency as much as possible.  Unfortunately, the memcache-client library had not yet been updated to use the new protocol.  That point became irrelevant once I learned that memcache has a 1MB limit on the size of objects being stored.  Although the limit can be configured at compilation time, preliminary research indicated this could be perilous.  Even then, the memcache-client lib has the 1MB limit hardcoded so that it does not attempt network traffic for large data, so the gem would also have to be patched and recompiled.


MRI (1.8.7) vs JRuby (1.3.1) vs YARV (1.9.1)
--------------------------------------------

### Lack of debugging with YARV ###

I was excited at the prospect of being able to use Ruby 1.9 for a project.  My contest entry had a small number of dependencies to cope with so I thought my potentials for failure were limited.  Generally, things seemed to work well with YARV.  I definitely saw a speed improvement over MRI.  However, the ruby debug gem has not been updated for 1.9.  Not having a debugger is not acceptable for development, so I had to abandon this path.

### Lack of profiling with JRuby ###
### JRuby "--fast" mode breaking debugger because debugger tied to stack trace output ###

After having experienced issues with MRI & Ruby 1.9, JRuby ended up being my runtime of choice.  It was remarkably faster than MRI and did not suffer from the garbage collection issues.  Likewise, it had the full debugger support that Ruby 1.9 lacked.  However, there were several issues I ran into with JRuby.

In order to maximize execution speed, I was using the `--fast` flag which performs some bytecode optimizations especially suited for FixNum operations at the cost of compatibility with Ruby's stacktrace format.  Generally, this was acceptable because I could largely deduce what a problem was by unraveling the stacktrace by inspection.  The problem is that the Ruby debugger apparently relies quite heavily on its stacktrace format.  I was astonished by this finding, but I was unable to use the debugger until the `--fast` flag was removed, forcing me to have to bounce from one to the other.

Another issue I ran into was directly attributable to me, but presented itself in odd ways.  I borrowed some ideas from ActiveRecord and kept a field named "id" in each of my models.  This worked fine when I had a rich object graph, but as I had to replace objects with strings to reduce my memory overhead, it became confusing to keep track of when I was using my "id" field versus the built-in Ruby "id" field.  Thus, I was effectively calling `object_id` when I thought I was calling a domain-specific `id` override.  This manifested itself by blowing out the JRuby heap.  My best guess is that I was creating a huge number of hash entries when using "id" as a key, as each `object_id` call would be unique, never allowing me to overwrite an existing entry.  For a while, though, I thought I had done something more severe to explode memory usage and began down the wrong debugging path.  Once I recognized the symptom it became a much easier problem to deal with.

Discovering the source of massive heap was difficult due to the lack of profiling tools for JRuby.  None of the standard Ruby ones seemed to work at all.  I tried to use [YourKit for Java](http://yourkit.com/), which technically worked, but the class names did not match the Ruby ones and I think I was profiling more JRuby proper rather than my application.  Unable to make much use of the results, I abandoned the profiling effort and resorted to manual code analysis, something humans are notoriously bad at.


The Switch to Java
------------------

### Fast execution ###

### Built-in thread pooling with producer / consumer model ###

### Profiling tools ###

### Debugger ###

### Could handle large object graph easily ###