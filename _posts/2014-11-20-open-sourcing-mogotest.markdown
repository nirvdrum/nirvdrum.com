---
layout: post
title: Open Sourcing a Failed Startup
---

Background
----------

In late October, 2014 I announced that I would be shutting down Mogotest.  After close to 5 years of operations it was clear I wouldn't be able to grow the business.  I don't think it was due to lack of business opportunity, but due to some business decisions made early on that became very difficult to course correct.  The exact line of reasoning that justified the shutdown is a topic for another day.  The purpose of this post is to discuss what to do with the code after the fact.

Are you Going to Open Source It?
--------------------------------

Rather predictably, one of the first things I was asked after I announced the shutdown was whether I would be open sourcing it.  I was asked from current customers, by friends, by companies that were interested in the tech but never felt the need to support it by giving us business, by random people on Twitter, and so on.  I had already gone through some of the thought process a priori, but I was in a different state of mind then.  Getting the bombardment of questions after the announcment impacted me in ways I couldn't predict.

For some additional context, I contribute to a lot of open source projects.  I don't have a "brand name" and I've never professionaly sold open source software or sold consulting services around it, but I've worked with a lot of projects.  I use the Apache Software license version 2.0 for just about everything.  And I guess I would consider myself more of a pragmatist than an ideologue when it comes to open source software.

With that said, my gut reaction was to not open source it.  My analytical reaction was also not to open source it.

Why Not?
--------

I'd just like to insert a standard disclaimer at this point that what follows is my own experience and my own potentially irrational thought process.  If anything I say comes off as a generality, please note that my pomposity stops short of speaking for others.

First up is the emotional aspect.  I had just made the extremely difficult decision to walk away from something I spent the past 5 years of my life dedicated to.  During that time, I lost at least two full years of wages, pissed through my savings, and lost ~$40K USD in cash invested into the business.  I battled with some form of founder depression.  Stastically speaking, this was the most likely outcome, so I'm hardly looking for sympathy.  But, having made that gut-wrenching decision to walk away from it all, the prospect of going back to it and investing a non-trivial amount of effort just to give it away is a really tough pill to swallow.

Also on the emotional aspect is just my own human pettiness.  I've been asked to open source the codebase from people that evidently didn't think the software was good enough to be worth paying for as a service.  I've been asked to open source the codebase by other companies in the space that didn't want to buy the rights when I was shopping the company around.  So, while I really want to provide a soft landing for my customers, I really didn't want to just be giving everything away to those that just wanted to mooch.

Setting all that aside, open sourcing the codebase is not some trivial process.  And I'm not talking wanting to clean up stuff I might be embarrassed by.  Here's a non-comprehensive list of issues that need to be addressed:

  * The web site design was a theme bought on WrapBootstrap that I don't have the rights to sublicense.
  * The rich UI widgets come from the commercial version of ExtJS.  That needs to be excised or the whole project needs to be GPLv3.
  * Sidekiq Pro needs to be removed.
  * Every JS lib and every image resource we used must have its license examined and potentially be replaced.
  * Any customer info that made its way into the code needs to be removed.  As an example, we built up an extensive regression suite around customer data that can't be distributed.  This whole process means auditing every file in the codebase.
  * Ensuring any API keys or passwords aren't floating around in the source or configuration files (obviously bad, but things happen).
  * Potentially unobscuring security holes while the service is still running.
  * Removing all the billing code.
  * Removing all the drip email campaign code.
  * Removing any other non-Web Consistency Testing parts from the code.

A lot of this is a liability.  Going through it all is a ton of work.  After all that, I open myself up to all sorts of scrutiny I don't really care for.  Sometimes I swear in code.  I hold a somewhat traditionalist view of English and prefer my plurality to match up, so I use gendered pronouns in my personal writings, which will have now just become public.  I'm certain there is some colorful commentary about each of the browser vendors buried somewhere in the source.  Without a doubt, something in this codebase will offend someone and my personal reputation is at risk when it simply wouldn't have been by keeping it private.

It's basically all the work required to clean up during an acquisition, but with the inverse financial outcome.

If I managed to clear that hurdle, the next problem is that I simply don't find there to be much value in open source code.  Open source projects, yes.  Open source code, rarely.  I won't have either the time or the wherewithal to spend any additional effort on this project.  If I make the code publicly available, people will have questions that I won't have time to answer.  Consequently, I'm just going to constantly feel like an inadequate piece of garbage.  On the other hand, if I manage to find time to engage, I don't have the energy to justify every design decision.  Some things do just look silly, but they were the product of the constraints imposed at the time.  Contextually, they were sound.  In today's world &hellip; probably not so much.  Fixing them would certainly be progress, but in my experience these sorts of things aren't approached tactfully and I'd rather not be called an idiot without having the resources to defend the context.


Second Thoughts
---------------

At the end of the day, I want Web Consistency Testing to evolve.  If making Mogotest open source will help achieve that, I'm willing to overlook some of the other problems.  I've already released the [ancillary libraries](https://bitbucket.org/mogotest/) as ASLv2, and I was going to release the main application under the Affero General Public License (AGPL).  After spending 14 hours cleaning things up this past weekend, I'm still not 100% certain I'm not violating IP somewhere or leaking customer info and I've had to gut the product so thoroughly that it's virltually useless.  Rewriting all the view code just isn't something I have the desire to do.

In conjunction with the decision to use the AGPL, I decided to try a [crowd-sourced campaign](https://www.indiegogo.com/projects/open-source-mogotest/x/2556255) to help with the open sourcing effort.  Precisely zero of the companies that have been begging me to open-source the code have contributed in any capacity.  The incredible amount spam I've received via comments on IndieGogo and Twitter have been equally disheartening.

Conclusion
----------

I had my initial emotional reaction, I analyzed the hell out of it, I decided against my better judgment to try opening the code anyway, and I simply can't do it.  I think the tools I have open-sourced will be beneficial to others and I've explained how things work fairly extensively in a talk I gave at [Google's Test Automation Conference](http://webconsistencytesting.com/).  A clean-room implementation shouldn't be too onerous, given I've solved a lot of the environmental problems you're apt to encounter.  Unfortunately, this is where I have to get off the train.