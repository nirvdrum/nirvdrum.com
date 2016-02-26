---
title: "Speed Up Web Testing with a Caching Proxy"
author: Kevin Menard
tags: [service, opensource, selenium, performance]
layout: post
excerpt: |-
  If you've ever done Web integration testing you know it can be really slow.  Starting up browsers is slow.  Loading Web pages is slow.  Interacting with those pages is slow.  Since Mogotest is a service built mostly around these concepts, we're constantly looking for ways to speed things up.  This post is about how to speed up page load times.
---

If you've ever done Web integration testing you know it can be really slow.  Starting up browsers is slow.  Loading Web pages is slow.  Interacting with those pages is slow.  Since Mogotest is a service built mostly around these concepts, we're constantly looking for ways to speed things up.  This post is about how to speed up page load times.

More often than not, your test environment is going to be anemic in comparison to your production environment.  If you're running integration tests locally, you're probably hitting an untuned, simple-to-use server, like WEBrick on Ruby or an embedded Jetty server on Java.  Request processing is likely sluggish and the server isn't configured for concurrent requests, meaning if you run tests in parallel to speed things up, you're going to hit a wall.

To that end, we've long routed all requests through a [Squid](http://www.squid-cache.org/) proxy.  Using a proxy allows us to:

  * Gate the number of connections to a remote server so we don't DoS it
  * Route requests from a known set of IPs (great for filtering or whitelisting)

Layering in a cache on top of that proxy would allow us to:

  * Deliver test run results to our customers faster
  * More efficiently use our browser cluster

Unfortunately, we never could get our Squid cache configuration right.  Granted, we do have some esoteric requirements that most won't have.  In particular, we need to work around third party sites with broken cache configurations.  The easiest way around this is to have a low TTL on the cache; that way we'd have the content cached for that short burst period of testing and then evict it before the next test starts.

We recently swapped out our Squid server for [Apache Traffic Server](http://trafficserver.apache.org/) (ATS).  While Squid has served us well, ATS grants us a very fine-grained control over the cache.  Additionally, it has a much simpler config and ships with a utility that makes cache usage monitoring easy.

Since soft-launching about two weeks ago, we've had **169,437** cache hits and **9,864** cache misses.  We hit the cache on **72.5%** of all our requests and save **69%** of our incoming bandwidth as a result.  The numbers are a bit skewed towards smaller documents, which have a higher hit rate.  This is because we start up all browsers at roughly the same time and they can only read from the cache if a resource is fully written out to the cache.  As such, larger objects may be fetched several times on the first page load, but things smooth out on subsequent pages.

It's hard to measure how much this speeds tests up given the variability in our customers' test targets.  For fast servers, the savings is minimal as the tests are dominated by browser rendering time.  But on particularly slow servers, like you'd see in a typical testing environment, we've seen tests complete 2 - 4x faster.

If you're integration testing your site using a tool like [Selenium](http://seleniumhq.org/), you may want to try placing ATS in front of your test environment server.  The set up is very simple, the overhead is minimal, and you'll almost certainly see faster test results due to faster page load times.