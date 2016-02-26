---
layout: post
title: On Amazon EC2 Spot Instances
author: Kevin Menard
---

Introduction
------------

A couple months ago Amazon [announced](http://aws.amazon.com/about-aws/whats-new/2009/12/14/announcing-amazon-ec2-spot-instances/) support for [EC2 spot instances](http://aws.amazon.com/ec2/spot-instances/).  In a nutshell, a spot instance is an EC2 instance that you bid on and that Amazon creates and destroys based upon whatever spare capacity is available in a given EC2 availability zone (i.e., supply) and your maximum bidding price versus the current spot instance price (i.e., demand).  A spot instance is less flexible than an on-demand or reserved instance is in terms of lifecycle, but could be significantly cheaper if your application can handle that volatility.

This post summarizes my experience with spot instances and how I make use of them.

Background
----------

My [latest project](http://mogotest.com/) is a front-end web testing tool, running a variety of web browsers across both Linux and Windows.  We make heavy use of EC2, which allows us to pay for servers as we use them.  While EC2 drastically reduces the start-up costs because we don't need to bulk purchase equipment, it can still be costly.  The rate as of this posting for a small Windows instance is $0.12 USD / hour.  At approximately 720 hours in a month, that's roughly $86 USD / month.  In order to process our work queue quickly we need to run a decent sized cluster.

Like any reasonable organization, we'd like to reduce cost without adversely impacting quality of service.  Prior to spot instances there were several ways to reduce cost, but none were ideal:

* The simplest, but most costly, is to purchase a [reserved instance](http://aws.amazon.com/ec2/#pricing).  With a reserved instance you pay an up-front fee but then pay reduced hourly rates as you run your instance.  Over the long term there are significant savings, but you have to be able to afford the initial cost and Amazon only supports reserved instances for Linux.

* Another cost-saving technique is to adjust your number of running instances based on load, so you don't pay for resources you aren't really using.  This can be tricky to do correctly though and you could be caught with an anemic cluster if you have a large burst of traffic.

* The hardest approach is to try to increase throughput on a given server.  This could require significant man power to achieve and for some applications may not even be possible.

Spot instances change the problem domain by making the instance price variable without having to be burdened with the initial expense of a reserved instance.  We've been able to get small Windows instances for as low as $0.05 / hour, which equates to a nearly 60% savings.  Similar savings can be had for linux servers as well at all of the various EC2 sizes (e.g., we routinely pay less for a medium linux spot instance than for a small linux on-demand instance).  Spot instance pricing can change at various times throughout the day, but the price is almost always below the current on-demand instance price.  Theoretically it could go higher than the on-demand price, but it would be silly to do so because you could just get an on-demand instance then.  With that savings, we can run more instances for each browser type on the same budget, increasing quality of service.

Of course, this is all predicated on the cluster being able to handle the dynamic addition and removal of instances.  You will have to account for the case where a spot instance dies in the middle of processing a request and be able to recover from that.  So, spot instances are not ideal for all applications.  But, for a background worker system it can be a cost-effective way to work through your queue more quickly.

Implementation
--------------

We use [rubber](http://github.com/wr0ngway/rubber) for our cluster configuration and app deployment.  Rubber is a [capistrano](http://www.capify.org/) plugin that simplifies working with Amazon Web Services.  Using role-based deployment, we can configure the packages and gems to be installed on an EC2 instance, attach an EBS volume if necessary, and backup files to S3 with succinct YAML configuration.  As of the [1.2.0 version](http://github.com/wr0ngway/rubber/commits/v1.2.0), rubber can now handle spot instance requests. 

A sample configuration for a single host in rubber would look like the following rubber.yml extract.

{% highlight ruby %}
# Sample spot instance request configuration in rubber.yml.
hosts:
  ie8:                         # The instance's hostname
    instance_roles: "vnc,rdp"  # Only expose VNC and RDP for this server
    cloud_providers:
      aws:
        image_id: ami-df20c3b6 # Standard 32-bit Windows 2003 Server image
        image_type: m1.small   # Create a small EC2 instance
        spot_price: "0.12"     # Max. spot price you are willing to pay
        spot_instance: true    # Default is false.
        spot_instance_request_timeout: 600 # Fall back to on-demand after 5 min.
{% endhighlight %}

While this example shows configuration for a single host, any option could also be applied globally for all nodes in your cluster and can be overridden on a host-by-host basis.  So, you can vary the maximum price you're willing to pay for a server on a per-instance basis and you can have a combination of on-demand and spot instances in your cluster.

One thing to note is that rubber was originally designed for on-demand and reserved instances, which have synchronous creation characteristics.  Spot instance requests, on the other hand, are satisfied asynchronously.  Rubber's solution is to block after the spot instance request is made and to poll Amazon until the instance is created.  Since waiting ad infinitum isn't ideal for everyone, rubber lets you set your own service level target through a request timeout value (`spot_instance_request_timeout` in the example).  If the request fails to be fulfilled before that timeout is exceeded, the spot instance request will be canceled and rubber will fallback to creating an on-demand instance.

We use [resque](http://github.com/defunkt/resque) for our work queue.  Resque does an excellent job of adapting to changes in the worker topology.  So, adding new workers through spot instances and even removing instances cleanly is managed nicely for us.  Additionally, resque was designed to handle job failures from the outset.  While this won't help you if your job is shutdown midway-through a non-atomic operation, it does ease the task of job management -- you just have to make sure your jobs are resumable.

Conclusion
----------

As would be expected, spot instance requests are easier to fulfill during non-peak hours.  Likewise, the most expensive operating times are during peak hours.  We've found that trying to create a spot instance during peak business hours may take a while to fulfill, whereas requests during non-peak hours are fulfilled quickly (oftentimes under 3 minutes).  If you set your maximum price high enough, you shouldn't lose your instance after it's created either, unless Amazon needs to reclaim resources for on-demand customers.  In practice we've run spot instances for weeks at a time.  We've also had some die shortly after creation because we didn't set a high enough maximum price.  You'll have to do some analysis to find out what's best for your application.

If you can be flexible with your EC2 availability zones, you'll see the best results.  There are marginal bandwidth fees between availability zones in the same region, but in our case the savings from a spot instance trump the bandwidth charges.  However, if you do large amounts of data transfer between instances, you should take that into consideration.

Overall, we've found spot instances to be a great way to grow our cluster with a fixed budget.  We've had to architect our application to be resilient to nondeterministic node additions and removals, but that was a lot easier for us than trying to increase the work throughput on any single server.
