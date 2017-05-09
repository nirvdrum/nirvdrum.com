---
layout: post
title: A Systematic Approach to Improving TruffleRuby Performance
author: Kevin Menard
draft: true
---

Summary
-------

We care a lot about performance in the TruffleRuby project.
We run a set of benchmarks on every push in a variety of VM configurations and use those as proxies for system-wide issues.
The problem with benchmarks, however, is unless you're intimately familiar with the benchmark it's hard to tell what your reported values should be.
Looking at a graph, it's easy to see if you've regressed or improved.
It's easy to see how you compare to other implementations.
But it's not terribly useful once you've leveled off.
We really need another way to analyze performance.

This is a bit of a lengthy post that introduces some of the tools available to Truffle language developers for performance analysis.
To help make things concrete, I walk through the end-to-end analysis of a simple Ruby method.
If you're just curious as to how we make Ruby faster, you can certainly skip over some of the nitty-gritty details.
I've tried to keep the post high-level enough to be interesting to those just curious about technology.
If you're developing a language in Truffle, you should find enough information here on how to get started in analyzing performance in your own language.


Introduction
------------

In talking with Rubyists at various venues I've run into a lot of people that are curious about how a runtime is implemented, which is fantastic.
Even if you're writing in a high-level language day-to-day, it's good to understand how things work under the covers.
It can help you build better software and provide a better mental framework for debugging issues, performance or otherwise.

Unfortunately, there's really no clear starting point for learning about how a runtime is built.
In the absence of some golden path, I'm going to walk through the analysis and improvement of a single core method in Ruby.
While I did pick this method out solely for the point of writing this blog post, the analysis and results are not contrived and the improved version of this method is now live in TruffleRuby.

To get started I looked at the compiled code resulting from running the Ruby language specs from the Ruby Spec project.
Generally, this isn’t a use case we pay much attention to because the code is short-lived and the executed paths won't match more conventional workloads, such as serving web applications.
But, it was a nice simple example of real world code that I thought might yield some interesting insights.
One issue that caught my attention was how we were optimizing `Array#empty?`.

A substantial portion of the Ruby core library in TruffleRuby is authored in Ruby.
`Array#empty?` is one such method and its implementation is straightforward:

```ruby
def empty?
  size == 0
end
```

Basically we have two method calls (`Array#size` and `Fixnum#==`) and a `Fixnum` literal reference.
Each of these methods is fairly simple, so let's look at them first.


Tracing Compilation
-------------------

We can instruct Truffle to print out trace details about its compilation by setting the Java system propert `-Dgraal.TraceTruffleCompilation=true`.
We've simplified that a bit in TruffleRuby by way of our [jt.rb tool](https://github.com/graalvm/truffleruby/blob/graal-vm-0.22/doc/contributor/workflow.md).
To see the trace output for `Fixnum#==` call, I ran:

```bash
GRAALVM_BIN=path/to/graalvm-0.22/bin/java ruby tool/jt.rb --trace -e 'loop { 3 == 0 }'
```

That yielded the following trace information:

```
[truffle] opt done         Fixnum#== (builtin) <opt> <split-3dce6dd8>                  |ASTSize       8/    8 |Time    79(  76+3   )ms |DirectCallNodes I    0/D    0 |GraalNodes    33/   28 |CodeSize          147 |CodeAddress 0x7f5f287b5c10 |Source       (core):1 
```

Restructuring this as a table, we have:

| AST Nodes                     | 8   |
| AST + Inlined Call AST Nodes  | 8   |
| Inlined Calls                 | 0   |
| Dispatched Calls              | 0   |
| Partial Evaluation Nodes      | 33  |
| Graal Lowered Nodes           | 28  |
| Code Size (bytes)             | 147 |
| Partial Evaluation Time (ms)  | 76  |
| Graal Compilation Time (ms)   | 3   |
| Total Compilation Time (ms)   | 79  |

As a trace statement, some of this can be daunting at first, but once you break it down it's really not too bad.
We don't need to make any extra method calls in `Fixnum#==`, so there are no inlined or dispatched calls.
Because we don't inline any calls, the number of AST nodes in total is the same as the number of AST nodes for the `Fixnum#==` method itself.
After partial evaluation (i.e., the Truffle compilation phase), we have 33 compiler nodes.
After Graal is done processing those nodes, we have 28 compiler nodes being handed off to code generation.
We can see compiling the method took 79 ms, with most of that being spent during partial evaluation (76 ms).
The resulting compiled method is 147 bytes.

In general, we want to reduce compilation time and code size.
If we can avoid dispatching method calls, that's best too.
The various node counts are interesting, but mostly as descriptive statistics.
As we work on optimizing a method, we can compare current and previous counts of each of these metrics to see if we're trending in a positive direction.


Visualizing the Compiler
------------------------

To gain more insight into the `Fixnum#==` method, we can also visualize the transformations of the Truffle AST to a lowered graph using a tool known as the [Ideal Graph Visualizer (IGV)](http://ssw.jku.at/General/Staff/TW/igv.html).
To start IGV, it's easiest to clone the [Graal repository](https://github.com/graalvm/graal).
You'll also need to clone the [mx repository](https://github.com/graalvm/mx) and add that to your PATH.
Then you can `cd` into the graal/compiler directory and run `mx igv`.

Once running, we'll feed data into IGV over the network.
You can instruct your Truffle language to dump graph data with the `-Dgraal.Dump=Truffle` system property & value.
The TruffleRuby `jt` tool makes this easier as well, by way of the `--igv` and `--full` flags.

```bash
GRAALVM_BIN=path/to/graalvm-0.22/bin/java ruby tool/jt.rb --igv --full -e 'loop { 3 == 0 }'
```

IGV will show the compiler graph after each phase of compilation.
For our purposes, the phase "After Truffle Tier" is a good starting point.
The compiler graph for `Fixnum#==` after Truffle has completed its partial evaluation looks like:

<a href="/images/a-systematic-approach-to-improving-truffleruby-performance/fixnum-equals-old.png" data-toggle="lightbox" data-title="Compiler Graph of Fixnum#== After Truffle Tier">
  <img src="/images/a-systematic-approach-to-improving-truffleruby-performance/fixnum-equals-old.png" />
</a>

Working from the top down to the bottom, we see a bunch of `LoadIndexed` nodes.
These correspond to loading values out of an array.
Every TruffleRuby method has some boilerplate consisting of 7..N such `LoadIndexed` nodes as [packed arguments](https://github.com/graalvm/truffleruby/blob/graal-vm-0.22/truffleruby/src/main/java/org/truffleruby/language/arguments/RubyArguments.java#L26-L32) are read from the frame.
The first 7 are fixed fields that TruffleRuby uses to track state for the method.
Any subsequent `LoadIndexed` nodes correspond to the Ruby-level arguments for the method.

Having run through our preamble, we then we unbox the `Fixnum` values; one for the receiver and one as the argument.
The unboxed values show up as constants attached to other nodes (`C()` corresponds to a constant and the `C(3)` & `C(0)` are just the int values `3` & `0` from our code snippet).
Then we can encounter a guard that checks assumption about this specialization hold and if not, it triggers deoptimization.
Finally we return the boxed value for `false` as integer value 0 (the `C(0)` attached to the `Return` node).

The graph will go through many more transformations before ultimately emitting machine code.
This level, however, is a good starting place for evaluating method performance.
Enough of the original Ruby AST is retained to trace things back to their source, while Truffle's partial evaluator has run giving a good idea of what the method looks like with inlined callers.
In this case, we have a very straightforward implementation for `==` and the graph has a tight, linear structure to it.


Method Specialization
---------------------

We can run through this entire process for the `Array#size` call as well.
Note that TruffleRuby has two implementations for for this method: one for empty arrays and one for non-empty arrays.

```bash
GRAALVM_BIN=path/to/graalvm-0.22/bin/java ruby tool/jt.rb --trace -e 'x = []; loop { x.size }'
GRAALVM_BIN=path/to/graalvm-0.22/bin/java ruby tool/jt.rb --trace -e 'x = [1, 2, 3]; loop { x.size }'
```

I'll skip the command-line trace info for these executions and collapse their trace data together into the same table:

| Metric                        | Empty Array | Non-empty Array |
|-------------------------------|-------------|-----------------|
| AST Nodes                     | 5           | 5               |
| AST + Inlined Call AST Nodes  | 5           | 5               |
| Inlined Calls                 | 0           | 0               |
| Dispatched Calls              | 0           | 0               |
| Partial Evaluation Nodes      | 34          | 40              |
| Graal Lowered Nodes           | 16          | 82              |
| Code Size (bytes)             | 131         | 289             |
| Partial Evaluation Time (ms)  | 49          | 59              |
| Graal Compilation Time (ms)   | 3           | 5               |
| Total Compilation Time (ms)   | 52          | 64              |

Comparing the two different implementations, we can see that they have the same number of AST nodes and method calls.
In fact, it's the same exact AST for both implementations; what's different is the structure of the receiver (i.e., the array instance).
In the empty array case, we simply always return 0.
For the non-empty array case, we must read the array length from the backing array store.
The difference here is quite large.
The empty array case is 45% of the size of the non-empty array case.

The IGV graphs for these two specializations don't add a whole lot more to the analysis, but for completeness I've linked to IGV graphs for `Array#size` for the <a href="/images/a-systematic-approach-to-improving-truffleruby-performance/array-size-empty-old.png" data-toggle="lightbox" data-title="Compiler Graph of Array#size (Empty Array) After Truffle Tier">empty array</a> and <a href="/images/a-systematic-approach-to-improving-truffleruby-performance/array-size-non-empty-old.png" data-toggle="lightbox" data-title="Compiler Graph of Array#size (Non-empty Array) After Truffle Tier">
non-empty array</a> cases.


Bringing it All Together
------------------------

Now that we've seen what the `Fixnum#==` and `Array#size` methods look like on their own, we're ready to see how they combine into `Array#empty?`.
For this analysis, I ran the Ruby language specs:

```bash
GRAALVM_BIN=path/to/graalvm-0.22/bin/java ruby tool/jt.rb test --graal :language
```

This time, the trace details look a bit more interesting:

| AST Nodes                     | 19   |
| AST + Inlined Call AST Nodes  | 32   |
| Inlined Calls                 | 2    |
| Dispatched Calls              | 0    |
| Partial Evaluation Nodes      | 112  |
| Graal Lowered Nodes           | 48   |
| Code Size (bytes)             | 263  |
| Partial Evaluation Time (ms)  | 58   |
| Graal Compilation Time (ms)   | 8    |
| Total Compilation Time (ms)   | 66   |

We can see that two methods were inlined into `Array#empty?`.
We know the only two methods being called are `Fixnum#==` and `Array#size`, but we can also figure that out by looking at the total AST node count.
We see that `Array#empty?` consists of 19 AST nodes on its own and we know `Fixnum#==` has 8 nodes and `Array#size` has 5, which added together is 32; the same value reported for the size of the AST and its inlined nodes.
The large number of partial evaluation nodes sticks out a bit, as does the code size.
When looking at the IGV graph for the `Array#empty?` method, we see something that looks a fair bit more complicated than expected.

<a href="/images/a-systematic-approach-to-improving-truffleruby-performance/array-empty-old.png" data-toggle="lightbox" data-title="Compiler Graph of Array#empty? After Truffle Tier">
  <img src="/images/a-systematic-approach-to-improving-truffleruby-performance/array-empty-old.png" />
</a>

You can enlarge the above graph by clicking on it.
Understanding the entire graph is useful in its own right, but I presented it mostly for contrast purposes.
When looking in IGV, you can scroll through the graph and selectively hide nodes that aren't of interest.
Unfortunately, doing that with PNGs is much harder so I've extracted the most interesting portion and reproduced below:

<a href="/images/a-systematic-approach-to-improving-truffleruby-performance/array-empty-old-interesting.png" data-toggle="lightbox" data-title="Compiler Graph of Array#empty? After Truffle Tier">
  <img src="/images/a-systematic-approach-to-improving-truffleruby-performance/array-empty-old-interesting.png" />
</a>

When looking at these larger graphs, I usually start by looking for nodes in red since they often indicate where something is going to be relatively slow.
Some of them are unavoidable, such as the `Return` node.
But here we can also see a `Merge` node in red and it's attached to a `Phi` node.
Here, a phi node corresponds to a phi function in [static single assignment form](https://en.wikipedia.org/wiki/Static_single_assignment_form).
In a nutshell, Truffle is unable to determine a definitive a way to get the array's size and as a result, it can't make any assumptions about the array's size, preventing some powerful speculative optimizations from being performed.


Method Splitting
----------------

Looking at that big green node, we can see that the `Array#size` method has become polymorphic.
Being polymorphic isn't a bad thing in and of itself, but it looks like the two different `Array#size` specializations prevent `Array#empty?` from doing an optimal job.
And this highlights how two different optimizations can work against each other.
Our optimized version of `Array#size` for empty arrays can produce tighter code with fewer memory accesses when a call site is monomorphic, but that comes at the cost of complicating callers handling different array types.
In this case if we had treated all arrays uniformly, we'd never have a polymorphic call site, allowing methods that call `Array#size` to perhaps optimize differently.

However, we do want to split methods up into their constituent cases.
Typically a call site for a method isn't really going to exercise every branch in that method.
By breaking a method down into its various cases and specializing for them, we can improve the quality of compiled code markedly for common cases.
To help us keep call sites monomorphic, Truffle can "split<a href="#footnote_1"><sup>1</sup></a>" a method.
With method splitting, a new copy of the subgraph corresponding to a method is made and inserted at a particular call site.
That way if two different pieces of code are calling `Array#size` and one always has an empty array and the other always a non-empty array, they each have their own copy of `Array#size` optimized for their respective case.

So, the real problem here is the splitting strategy isn't optimal.
In TruffleRuby we currently use a very coarse heuristic that methods implemented in Java are the most critical and thus we instruct Truffle to always split them.
Both `Array#size` and `Fixnum#==` are implemented in Java and thus are always split.
However, splitting every method would result in massive explosion in graph size, so we do not explicitly split methods written in Ruby, such as `Array#empty?`.
That doesn't mean that methods written in Ruby are never split, but we transfer control to Truffle to make that determination.
It appears in this situation we haven't given Truffle enough information to make an optimal decision.
As a result, `Array#empty?` isn't split and all of its callers ends up sharing a single copy of the method, causing its call of `Array#size` to become polymorphic.
(Note, while `Array#size` will split, that's done based on the call site.
The lone call site here is the one inside the `Array#empty?` method definition.)


Improving the Method
--------------------

Revisiting the `Array#size` implementation, we realized we could meet our original needs with a single specialization and using [value profiles](http://graalvm.github.io/graal/truffle/javadoc/com/oracle/truffle/api/profiles/ValueProfile.html).
Note that using value profiles is orthogonal to improving the splitting heuristic, but it does allow us to solve the immediate problem with `Array#size`.
With the new implementation the performance profile for the empty & non-empty array cases is now identical *and* smaller than the original.
Note that compilation times are non-deterministic.
Despite compiling the exact same graph, we seem some variance in how long it takes compilation to complete.

| Metric                        | Empty Array | Non-empty Array |
|-------------------------------|-------------|-----------------|
| AST Nodes                     | 5           | 5               |
| AST + Inlined Call AST Nodes  | 5           | 5               |
| Inlined Calls                 | 0           | 0               |
| Dispatched Calls              | 0           | 0               |
| Partial Evaluation Nodes      | 34          | 34              |
| Graal Lowered Nodes           | 17          | 17              |
| Code Size (bytes)             | 130         | 130             |
| Partial Evaluation Time (ms)  | 49          | 63              |
| Graal Compilation Time (ms)   | 3           | 2               |
| Total Compilation Time (ms)   | 52          | 65              |

More importantly, this new `Array#size` implementation allows Truffle to perform more optimizations in `Array#empty?`.
Re-running the Ruby language specs, we now see:

| Metric                        | Old Array#empty? | New Array#empty? |
|-------------------------------|------------------|------------------|
| AST Nodes                     | 19               | 19               |
| AST + Inlined Call AST Nodes  | 32               | 32               |
| Inlined Calls                 | 2                | 2                |
| Dispatched Calls              | 0                | 0                |
| Partial Evaluation Nodes      | 112              | 42               |
| Graal Lowered Nodes           | 48               | 26               |
| Code Size (bytes)             | 263              | 176              |
| Partial Evaluation Time (ms)  | 58               | 57               |
| Graal Compilation Time (ms)   | 8                | 5                |
| Total Compilation Time (ms)   | 66               | 62               |

The new `Array#empty?` (really, the same method but with a better `Array#size`) has almost 1/3 the number of nodes after partial evaluation and is about 2/3 the code size.
If we look at the compiled method in IGV now, we see a much simpler graph:

<a href="/images/a-systematic-approach-to-improving-truffleruby-performance/array-empty-new.png" data-toggle="lightbox" data-title="Compiler Graph of Improved Array#empty? After Truffle Tier">
  <img src="/images/a-systematic-approach-to-improving-truffleruby-performance/array-empty-new.png" style="margin-left: 250px"/>
</a>

This graph looks much more like the composition of the `Array#size` and `Fixnum#==` graphs we would expect to see with both of those methods inlined.
Reading from top to bottom, we see the standard set of `LoadIndexed` nodes for reading packed values out of the frame.
We then see a field load to get the size for the array (the constant attached to the node labeled `509`).
We see the size compared to the value `0` (node `1291`).
And finally we a boolean value, represented as integers, returned depending on the results of the comparison.


Conclusion
----------

When it comes to performance analysis we often turn to benchmarks and profilers.
Both are great tools in their own right, but they often lack granularity.
Truffle's tracing output and IGV provide lower-level metrics that can be used to improve a method's compilation.

As a Truffle language developer, ensuring our code is amenable to Truffle's partial evaluator is one of the most important things we can do.
So, while a lot of the preceding discussion looks like micro-optimization, it's the sort of thing that can have large cascading consequences.
E.g., now that `Array#empty?` no longer has a phi node, it itself can be constant-folded into a piece of larger code and so on.

I think perhaps the most important part of all this is you don't need to be a compiler expert to help the compiler out.
You don't even need to be able to evaluate the resulting machine code.
If you know the basic rules of how to play nicely with Truffle you can go really far.
Granted, reading the IGV graphs does take some getting used to &mdash; there's still a fair bit I don't understand!
But you can get pretty far with pattern matching after learning the basics.
You can easily test out a hypothesis in a high level language (Java or Ruby, in this case) and compare the before & after metrics.
To me, that's incredibly powerful.

<hr/>

<a name="footnote_1"></a>
<sup>1</sup>
<small>
  In the original ["One VM to Rule Them All"](https://pdfs.semanticscholar.org/8251/bd1f9496b61e7418b96a816f31477de4c75d.pdf) paper, method splitting is referred to as "tree cloning."
  You can consult the paper for a more thorough treatment of the topic.
</small>
