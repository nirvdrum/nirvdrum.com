---
title: Improving Sidekiq Performance with JRuby
author: Kevin Menard
tags: [opensource, jruby, performance]
layout: post
excerpt: |-
  Sidekiq performance in JRuby can be substantially improved by reducing Thread and Fiber allocations. We take a look
  at how JRuby 1.7.11 improves this by backing Fibers with an internal thread pool and how layering in a Fiber pool
  with Celluloid can improve things even more.

metadata: |-
  JRuby 1.7.10

  1,000   : 1.343, 1.404, 1.279      ->   1.342
  10,000  : 12.813, 11.36, 11.406    ->  11.860
  100,000 : 116.453, 117.69, 115.766 -> 116.636

  JRuby 1.7.10 w/ Fiber Pool

  1,000   : 0.944, 0.94, 0.91      -> 0.931
  10,000  : 7.094, 7.108, 7.188    -> 7.130
  100,000 : 68.207, 69.077, 69.468 -> 68.917

  JRuby 1.7.11

  1,000   : 0.921, 1.102, 0.897    ->  0.973
  10,000  : 7.755, 7.374, 7.214    ->  7.448
  100,000 : 71.504, 72.295, 71.993 -> 71.930

  JRuby 1.7.11 w/ Fiber Pool

  1,000   : 0.897, .91, 0.824      ->  0.877
  10,000  : 7.105, 6.996, 7.009    ->  7.037
  100,000 : 68.539, 68.879, 69.171 -> 68.863
---

To handle all the work in our [Web Consistency Testing](http://webconsistencytesting.com/) process we use a
background job queueing system called [Sidekiq](https://github.com/mperham/sidekiq). Sidekiq, in turn, makes use of a
Ruby actor framework called [Celluloid](https://github.com/celluloid/celluloid). And Celluloid ultimately makes use of
a Ruby 1.9 feature called [Fibers](http://www.ruby-doc.org/core-1.9.3/Fiber.html). Fibers are Ruby's implementation of
coroutines. Unfortunately, the JVM doesn't natively support coroutines, so in order to be API compatible with MRI,
JRuby needs to fake it with JVM threads. This has historically proven tricky to implement properly and has resulted in
a variety of issues related to both correctness and performance throughout the JRuby 1.7 release cycle.

Fortunately, we haven't encountered any Fiber bugs since JRuby 1.7.4. However, while recently profiling our application
in production we found that anywhere from 25% - 35% of total CPU time was spent in Fiber creation. Incidentally,
at least two others running JRuby 1.7.10 discovered the same thing around the same time, making for some fun IRC conversation.

Fibers on MRI are cheap to create so their use case usually entails using a lot of them in a compressed timeframe. JRuby's
Fiber implementation requires using one thread per Fiber.  This can result in extremely high thread churn if they're
not reused.  In the case of Sidekiq &amp; Celluloid, every background job ends up creating a Fiber.  With JRuby 1.7.10,
if we have 100,000 jobs to run, we're creating at least 100,000 threads.  While the JVM may be very good at
multi-threading, thread creation itself is still something of a heavy process and unnecessary object allocations are
generally bad for performance.

The fix was straightforward: add a pool of some sort. [Chris Heald](https://github.com/cheald) had
come across the same Fiber performance issue and solved it by implementing a
[Fiber pool for Celluloid](https://github.com/celluloid/celluloid/pull/371) that fixes the problem for all versions of
JRuby. Unfortunately, in order to get Sidekiq to use it one must
[monkeypatch Sidekiq](https://github.com/nirvdrum/jruby_fiber_benchmark/blob/master/noop_worker.rb#L10-L12) after it's
already started. JRuby 1.7.11 fixes the Fiber problem by using an internal thread pool dedicated to Fibers, substantially
reducing the number of allocated threads.

In order to measure the performance profile of each of these options, I pulled together a
[very simple Sidekiq job](https://github.com/nirvdrum/jruby_fiber_benchmark).
This job effectively does nothing more than run through the mechanics of Sidekiq. The test methodology here isn't
exactly precise, but it's good enough to get a sense of magnitude.  Basically, in one process we run the Sidekiq daemon,
which will spin up 100 workers to process a work queue.  In another process we enqueue a configurable number of jobs
and then busy loop until the workers have depleted the queue; when the queue is empty, we report the execution time.
Since that busy loop will sleep for 100ms, results may be be off by at least that much.  Likewise, an empty queue doesn't
actually mean all the work has been completed, it just means a worker has taken a job from the queue, so results will be
skewed by however long it takes a Sidekiq worker to no-op. At worst, that is expected to be another couple 100ms.

All tests were run against a Linux Mint 16 machine (Ubuntu 13.10 derivative) with 16 GB RAM, SATA III SSD, running the
3.11.0-12-generic kernel on an Intel Core i7-2760QM (2.40 GHz quad core with hyperthreading). I used the 64-bit 1.7.0u51
JVM (Java HotSpot(TM) 64-Bit Server VM (build 24.51-b03, mixed mode) and my JRUBY_OPTS are set to
`-J-XX:+TieredCompilation -J-XX:TieredStopAtLevel=1 -J-noverify -J-Xmx2G -Xcompile.invokedynamic=false -Xreify.classes=true`
Redis 2.6.3 was used and local RDB saving and AOF were disabled to remove disk I/O.

I also wanted to get a handle on whether performance scaled with the number of jobs, so I measured each dimension
with 1,000, 10,000, and 100,000 jobs.  Since I am talking to Redis over a network socket (but on localhost), I ran each
experiment 3 times and took the average as a way to mitigate against any sort of network jitter.  Table 1 summarizes
the results of these experiments.

<table class="table table-bordered">
  <caption>Table 1: Job execution time (s) per JRuby configuration.</caption>

  <thead>
    <tr>
      <th>Jobs</th>
      <th>JRuby 1.7.10</th>
      <th>JRuby 1.7.10 <br /> w/ Fiber Pool</th>
      <th>JRuby 1.7.11</th>
      <th>JRuby 1.7.11 <br /> w/ Fiber Pool</th>
    </tr>
  </thead>

  <tbody>
    <tr>
      <th>1,000</th>
      <td>1.342</td>
      <td>0.931</td>
      <td>0.973</td>
      <td>0.877</td>
    </tr>

    <tr>
      <th>10,000</th>
      <td>11.860</td>
      <td>7.130</td>
      <td>7.448</td>
      <td>7.037</td>
    </tr>

    <tr>
      <th>100,000</th>
      <td>116.636</td>
      <td>68.917</td>
      <td>71.930</td>
      <td>68.863</td>
    </tr>
  </tbody>
</table>

As it turns out my numbers did not vary significantly from what Chris saw in his own
[benchmark of Celluloid directly](https://github.com/celluloid/celluloid/pull/371#issuecomment-33721140).  In both
cases, use of a Fiber pool substantially sped things up.  Since almost all activity is CPU-bound here, that's a pretty
massive savings.  There's no need to introduce a caching layer or optimize queries &mdash; just reduce costly object
allocation.  And simply upgrading to JRuby 1.7.11 will yield a similar savings without any code modifications to your
application.

It's unfortunate that JRuby ever spent this much time just burning CPU, but it's also impressive that even with this
deficiency it has routinely beaten out MRI on Sidekiq based benchmarks. It continues to be great that the JRuby
team is very responsive to reported performance issues. This matter was fixed within days of the issue being filed and
a new release was published a couple weeks later.

We've opted to run both JRuby 1.7.11 and the Fiber pool.  Our backend jobs run faster and now that we've removed a major
bottleneck, we can focus on other areas in profiling.