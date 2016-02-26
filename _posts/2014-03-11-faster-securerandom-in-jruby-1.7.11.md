---
title: Faster SecureRandom in JRuby 1.7.11
author: Kevin Menard
tags: [opensource, jruby, performance]
layout: post
excerpt: |-
  While profiling our Rails app recently, SecureRandom surfaced as a hot spot. We use UUIDs to generate request IDs
  so we can correlate different log statements with a logical user request. To isolate the problem
---

While profiling our Rails app recently, SecureRandom surfaced as a hot spot. We use UUIDs to generate request IDs
so we can correlate different log statements with a logical user request. To isolate the problem, I removed the request IDs
and profiled again, but found that SecureRandom still showed up. As it turns out, in Rails 3.2+ the
[ActionDispatch::RequestId](https://github.com/rails/rails/blob/v3.2.17/actionpack/lib/action_dispatch/middleware/request_id.rb)
middleware is enabled by default and does essentially what we were doing &mdash; it creates unique request IDs so
disjoint log statements can be grouped together.  While we were able to cut out 50% of the SecureRandom calls just by
using the Rails middleware instead of our custom solution, we were still making a slow call on every request.  And since
this is a Rails default behavior, it became evident this could be adversely impacting a lot of sites.

One of the things I love about JRuby is that when a compelling benchmark is pulled together, it's easy to convince the
team to focus on an area to improve performance.  And if it's demonstrated that the issue is affecting a lot of people,
the fix is given even higher priority.  With new, stable releases coming out almost every three weeks, performance issues
don't get buried for months or years at a time.

Initially we were concerned with raw execution performance, since that would manifest itself as slow response times.
Figure 1 shows the simple benchmarking code we used, while Figure 2 shows the results from JRuby 1.7.10 and MRI 2.1.0
merged into a single report.

<div class='figure'>
  {% highlight ruby %}
    require 'benchmark/ips'
    require 'securerandom'

    Benchmark.ips do |x|
      x.report('SecureRandom') { SecureRandom.uuid }

      if defined?(JRUBY_VERSION)
        x.report('Java') { java.util.UUID.random_uuid.to_s }
      end
    end
  {% endhighlight %}

  Figure 1: SecureRandom serial benchmark code.
</div>

<div class='figure'>
  <pre>
Calculating -------------------------------------
        SecureRandom      3970 i/100ms
                Java      7661 i/100ms
  SecureRandom (MRI)     14068 i/100ms
-------------------------------------------------
        SecureRandom    47757.0 (±6.6%) i/s -     238200 in   5.013000s
                Java   104769.4 (±12.8%) i/s -    513287 in   5.032000s
   SecureRandom (MRI)  177503.1 (±4.1%) i/s -     900352 in   5.082986s
  </pre>

  Figure 2: JRuby 1.7.10 SecureRandom serial benchmark results.
</div>

If you're not familiar with benchmark/ips, it basically measures throughput in a calculated timeframe and reports results
as the number of iterations per 100ms interval. The two instrumented calls show the difference between calling JRuby 1.7.10's
implementation of SecureRandom for generating a UUID and calling Java's built-in utility method for creating a UUID from JRuby.
Naturally, MRI can't run the Java method, so its benchmarked value is the pure Ruby SecureRandom call.  As can be seen,
both implementations in JRuby are slower than that of MRI. But what's immediately interesting is how the JRuby
implementation is roughly twice as slow as the pure Java implementation. In theory, JRuby can achieve the same speed that
Java can. Of course that's not always the case, but when there's such a wide performance gap as seen here, there's almost
certainly something that can be done more efficiently.

The [SecureRandom implementation in JRuby 1.7.10](https://github.com/jruby/jruby/blob/1.7.10/lib/ruby/shared/securerandom.rb)
is a thin Ruby wrapper around the native Java SecureRandom implementation. In JRuby 1.7.11 it's been [rewritten](https://github.com/jruby/jruby/blob/1.7.11/core/src/main/java/org/jruby/ext/securerandom/SecureRandomLibrary.java)
to move critical sections into pure Java. This allows saving costly coercion between Java and Ruby objects. Moreover,
rather than create a new Java SecureRandom object every time a random value is needed, the new implementation only
creates one for the life of a thread. This is a big performance boost because by default the JVM on Linux will
seed a SecureRandom instance from /dev/random, which locks on a synchronized method and may block indefinitely if /dev/random
determines it needs more environment data for its source of entropy. By reusing an already seeded instance a significant
amount of time can be saved as can be seen with the benchmark code run on JRuby 1.7.11 (Figure 3).

<div class='figure'>
  <pre>
Calculating -------------------------------------
        SecureRandom     33114 i/100ms
                Java      7917 i/100ms
  SecureRandom (MRI)     14068 i/100ms
-------------------------------------------------
        SecureRandom  1019165.3 (±6.9%) i/s -    5066442 in   5.011000s
                Java   108395.6 (±13.8%) i/s -    530439 in   5.052000s
   SecureRandom (MRI)  177503.1 (±4.1%) i/s -     900352 in   5.082986s
  </pre>

  Figure 3: JRuby 1.7.11 SecureRandom serial benchmark results.
</div>

While the measurement for the naive pure Java implementation used in the benchmark has a fairly high margin of error,
we can see the new JRuby implementation is roughly 4x faster than calling the Java method. In contrast, with JRuby 1.7.10
it was 2x as slow. And the new JRuby implementation is now approximately 2.3x faster than MRI, whereas with JRuby 1.7.10
it was 3.5x as slow. The same caching trick could be employed with the Java method call, so JRuby isn't faster than Java,
but all that heavy lifting has been done for us. And while that may seem contrived, the Java method for fetching a UUID
is a static one, so it's often called directly with no opportunity for the developer to cache the underlying
SecureRandom instance.

The second half of our performance story presented itself while doing load testing against the running application.
Unfortunately, it was much harder to create a reproducible benchmark for that. But with JRuby 1.7.10 we were seeing a
lot of blocked threads as we increased the number of concurrent requests. The changes in JRuby 1.7.11 avoid hitting an
internal lock in the JVM and reduces the chance of blocking on /dev/random, which allowed us to increase the number of
concurrent connections by approximately 50% in a test environment. This number will vary wildly for other apps, however,
because the likelihood of multiple threads hitting the same code path is highly variable on the rest of the response cycle.

If you haven't upgraded to JRuby 1.7.11 yet, I'd highly recommend it. This is just one of several big performance
improvements made that have real world benefits. Faster response times mean happier customers and increased request
throughput helps us contain infrastructure costs.