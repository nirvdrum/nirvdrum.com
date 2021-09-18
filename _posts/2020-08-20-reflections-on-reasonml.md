---
layout: post
title: Reflections on ReasonML
author: Kevin Menard
date: 2020-08-20 08:59 -0400
---

Introduction
------------

For the past year or so I've been working on a side project with ReasonML.
When people hear about it, they often ask me what my thoughts are and how it's working out, so I've collected that feedback here.


Why Did I Choose ReasonML?
--------------------------

I'll start off by saying that I deliberately picked ReasonML for a personal project so I didn't need to factor in all the reasons that a business may or may not want to adopt a new niche technology.
At the core of it, I loved Standard ML while in university and OCaml, which ReasonML is based on, scratches that it itch for me.
JavaScript, on the other hand, really doesn't appeal to me even with the ES6 additions.
It's a perfectly fine language that gets the job done, but I just wasn't excited to work with it on a personal project.

Of course, there are plenty of non-JavaScript languages with all sorts of language semantics.
I had considered Elm, amongst others, but landed on ReasonML because it looked to have excellent support for React and JavaScript interop in general.
Additionally, being backed by Facebook suggested to me that the language may have some longevity by way of a corporate booster.

As a secondary concern, I wanted to get a feel for the productivity trade-offs between ReasonML and TypeScript as a discussion topic for my new company.

ReasonML is an interesting beast in that it layers in a new JS-like syntax for OCaml.
I wasn't a fan of it at first, but it eventually started to feel natural for writing a web application.
I think that was perhaps due to ReasonReact's support for JSX.


How Do I Feel About that Decision?
----------------------------------

If I had to do it over again, I would be hard-pressed to go with ReasonML.
This probably isn't a shocking conclusion for many: JavaScript has first-class support in web browsers and languages that target JavaScript spring up and wither away with regularity.
When things are going well, ReasonML really shines and it's a joy to work with.
Unfortunately, I hit several snags during my evaluation of the language and as a consequence my enthusiasm with the project waned.
These things happen and I expected to run into them, but I hadn't adequately considered how demotivating they could be.

When I originally got to this point in the writing, I had decided I wouldn't write this post.
I have no real interest in criticizing a project or its community, but I was encouraged to complete my writing anyway in the spirit of all feedback being good feedback.
Please try to read the rest of this in the most charitable way possible.
At best, my feelings on ReasonML are conflicted.
The ReasonML community has been nothing but warm and helpful, even when I was clearly frustrated.


Community
---------

The ReasonML community is perhaps the most welcoming one I've participated in.
There is an active Discord server with several focused channels.
I found the discussions there informative and questions are answered fairly quickly.
It's nice to see a group of enthusiasts willing to donate their time to help newcomers to the language.

Unfortunately, the discussion is now split between a Discord server and a Discourse instance.
I appreciate that Discourse makes it easier for asynchronous communication, but it's also a siloed community.
With Discord, I can be connected to multiple servers at one time and engage in chat at my leisure, getting notifications in something that isn't my web browser.
With Discourse, I just can't keep up with all the various communities expecting me to sign up for yet another account.
This isn't particular to ReasonML, but I do find it lamentable.
We've regressed a long way from multi-community IRC servers and mailing lists.


Documentation
-------------

ReasonML and BuckleScript both have fairly comprehensive documentation.
ReasonReact, however, has very little documentation.
Consequently, you're left having to look at the JS docs for React, looking at the type definitions for ReasonReact, and maybe a tutorial or two online.
Things don't always match up 1:1 and it's just a very difficult way to get started.

To their credit, the ReasonReact team acknowledges this is a shortcoming, but given constrained resources are [seeking community help](https://github.com/reasonml/reason-react/issues/473).
I'd love to be able to help out, but writing docs for something I barely understand is unlikely to be all that helpful.
Moreover, I was looking to use ReasonML in large part for its purported productivity gains; having to pause to write the documentation for a big project in that ecosystem is a (helpful) distraction.



Ecosystem
---------

The ReasonML ecosystem is frankly rather confusing.
ReasonML is the language, but I never installed it.
Instead, I installed BuckleScript, which packages its own version of ReasonML and it's generally not clear what that version is.
The only way I found to tell which version of Reason I was using was to run its code formatter with a version flag.

I still don't know how one goes about installing ReasonML standalone.
There's a package called `reason-cli` that looks like it will do it, but it's wildly out of date.
Alas, there is documentation floating around telling you to do just that, which means you'll have a tool that won't run many code examples and it won't be obvious why.

Then there's ReasonReact, which is a dependency you need to add to use, but part of ReasonReact also ships inside BuckleScript.
Between BuckleScript 7.0.1 and 7.1.0, a correctness change was made to ReasonReact code shipping within BuckleScript that broke several major projects in the ReasonML ecosystem.
Just to reiterate, even if you didn't update the ReasonReact version in your package.json/yarn.lock, suddenly code that worked before stopped working.
It took over a month for this to finally settle down.
In that time, I had to run forks of both direct and transitive dependencies just to get my project working with the newer BuckleScript.
I suppose I could have waited to upgrade, but there was a bug in BuckleScript 7.0.1 that was fixed in the 7.0.2-dev releases that only appeared in 7.1.0, as 7.0.2 was never released.
For people completely new to ReasonML, things were broken out of the box.

It was an unfortunate sequence of events, but variations of it have played out multiple times in the past year.
When BuckleScript 6.0.0 was released, _graphql_ppx_ was broken and the maintainer of that project had stopped maintaining it.
That necessitated a fork, which in turn required dependent projects to update their dependencies to work with the new fork.
It all worked out, but hitting these issues that are largely out of your control, and with such frequency, is really demoralizing.
As of this writing, [_reason-apollo_ wasn't compatible with the ReasonReact 0.8.0](https://github.com/apollographql/reason-apollo/issues/246).

It might be that ReasonML isn't a great fit for React and GraphQL applications, in which case I just picked the wrong tool for the job.
There is, however, a lot of promising work going on with the _reason-relay_ bindings and a lot of activity on improving _graphql_ppx_ and _reason-apollo-hooks_.
I'm not all that interested in switching to Relay, however, so I'm sticking with Apollo for the time being.
I've been contemplating just using something like [RxDB](https://rxdb.info/) and offload the GraphQL server interaction to another library.

Setting compatibility issues aside, there just aren't that many published ReasonML bindings or libraries.
The ReasonML community promotes writing bindings for just the parts of a library that you need, since it has pretty good JS interop.
Sadly, that means there's a dearth of good bindings to look at as an example and I found the documentation a bit too high-level to be entirely practical.
BuckleScript's interop facilities are certainly rich, but if you mess something up, it can be incredibly obtuse to work out.
It's also evolved a lot over several major releases, so any examples you do find may well be out of date.
I think I have a pretty good handle on it, but it was a lot of effort to get to that point, and I don't think it would have been possible at all without help from others on Discord.

Moreover, what bindings or libraries do exist often lack a changelog or tagged releases.
That makes it hard to tell what's changed between releases.
It's a problem from the top down, as ReasonML [hasn't tagged a release since 2017](https://github.com/facebook/reason/issues/2461).
It leads to this situation where you need to be "in the know" to figure out what's changing where and when.
Or, just blindly upgrade, which can lead to the aforementioned compatibility problems.
And if you pick up a new library that doesn't work with the current ecosystem, good luck trying to find an older version that might work because you're unlikely to get any more help than the simple version listing on NPM.


### Type Definitions

As I previously mentioned, people are discouraged from releasing packages that are little more than bindings for existing JavaScript/Flow/TypeScript projects.
Consequently, a lot of my time is spent manually converting TypeScript definitions to ReasonML.
While straightforward once you learn how to do it, it's slow and frustrating.
The reality is, if I just used TypeScript I could get on with writing the application logic.

Since the bindings take a long time to write and easily fall out of date, the community recommendation is to only map what you need.
But, then you don't get any of the wonderful IDE support that you'd have with TypeScript, such as API discovery and full auto-complete.
You'd also have to keep the TypeScript definitions around so you can consult them every time you want to see the full API.
It's awkward and hard to view as anything other than a waste of time.

There have been a few aborted attempts at automating the conversion of TypeScript to ReasonML definitions.
For simple type definitions they should map straightforwardly.
Having looked into it a bit myself, I believe one of the biggest problems is you can't inherit or mix in record definitions in ReasonML, so things like inherited interfaces can't be mapped easily (or well).
It might be interesting if BuckleScript had a `@mix-in` or `@include` annotation that could be applied to record fields that are of type record.
Then from ReasonML you could use nested field access like normal, but BuckleScript could then map that back to a flattened property list in JavaScript.

Without a tool to convert TypeScript definitions to BuckleScript, I think ReasonML will always remain a niche technology.
Building up your own types works wonderfully when building up an internal API.
But, modern web apps pull in many modules and having to write bindings for each is overwhelming.

Another community recommendation is to use a hybrid application, where part is written in ReasonML and part written in JavaScript/Flow/TypeScript.
While that would solve the complex type mapping problem, it comes at the cost of a more complicated project structure.
Personally, at that point I'd find it hard to justify using ReasonML if I already need to maintain a parallel TypeScript project.


### Standards

The ReasonML community is interesting in that it's incredibly small, so a lot is up for grabs.
It actively encourages newcomers to participate in various ways.
However, it also has very strongly held opinions on code structure and formatting which comes off as gatekeeping to me.
People tend to fall into two camps on this debate, but if it truly doesn't matter, then my arbitrary choice is just as good as yours.
I'll provide two such examples.

The first one is the compiled ReasonML file output is placed in the same directory as the source _.re_ files.
I've worked with a lot of languages and systems that use code generation and in every other case the generated files are placed somewhere else, oftentimes not committed.
I believe the idea here is for incremental adoption of ReasonML in existing JavaScript projects, so you can directly modify the generated JS files if needed.
I found it just made working with the code harder.
Having both _src/App.re_ and _src/App.bs.js_ makes navigating code harder.
Tab-completion gets messed up, an IDE's UI gets cluttered, and jumping to code doubles the number of candidates.
Changing the location is configurable, but I was discouraged from doing so.
Tools like [Parcel](https://parceljs.org/) just silently fail if you use anything other than the defaults.

The second one has to do with `refmt` preferring 80 character wide lines.
I can run four terminals side-by-side with 120 characters and still have room to spare, so I generally find 80 characters to be unereasonably narrow.
This problem, however, is compounded by BuckleScript's interop annotations.
I've had cases where they'll take up ~60 characters themselves, so even relatively short, nicely formatted code is getting split over two lines.
Moreover, if a function call gets split, each argument will be placed on its own line, so a line of 85 characters suddenly turns into four lines.

Fortunately, the character width is controllable, but I was requested not to do that for any open source code in order not to create problems for any hypothetical contributors.
I didn't quite understand the problem if I just added my own "script" to _package.json_, but I guess it creates problems with editors.
As a result, I've just opted not to open source any of my bindings.
I find the wider lines considerably easier to read and this whole project was supposed to be for fun.
If I need to give that up to participate in the open source community, it's not really worth it to me.


Facebook
--------

Inititally, I thought the backing of a major corporation would be a strength of ReasonML.
Essentially, if Facebook is relying on the technology I expected it would survive where other smaller community projects have died out.
However, over the course of the past year I've come to realize Facebook does open source a lot differently than others.
First, I find Facebook doesn't quite interact with the community like many others.
People give my previous employer (Oracle) a lot of flack, but if you have a question about GraalVM, you can expect timely response on Slack, GitHub, or Twitter.
Facebook seems to do a lot of work internally, quietly, and maybe eventually releases it.
The other problem I have is Facebook takes "opinionated" to a level I haven't really seen elsewhere.
Each of their projects I've tried makes design decisions for Facebook's unique use cases and doesn't make that configurable, instead trying to pass them off as best practices.
If you work on large polyglot teams focusing on real-time newsfeed-like products, then their decisions make a lot of sense.
If like most of us, you don't, you just have to learn to adapt to those design decisions.
I believe tools should adapt to the needs of the user, not the other way around.

That's to say nothing of their contributor license agreement (CLA) requirement.
I don't have an inherent problem with CLAs.
I've signed a few over the years, mostly for open source organizations (Apache Software Foundation and Software Freedom Conservancy, for Selenium).
I've signed one with Oracle to contribute to GraalVM.
I can't say if I've just had a change of heart on them or if the [phrasing of the Facebook one](https://code.facebook.com/cla/individual) is problematic, but this was the first time I felt the language was dense enough to warrant hiring a lawyer.
I have no interest in paying the fees for a lawyer in order to contribute documentation fixes for a project I'm working on on the side with no commercial value.
So, this is a situation where being open source doesn't really gain me much.

<a name="summary">Summary</a>
-------

I think ReasonML is a really interesting project with a lot of teething problems.
Just recently, ~~BuckleScript~~ ReScript<a href="#footnote_1"><sup>1</sup></a> unveiled [a brand new syntax](https://reasonml.org/blog/bucklescript-8-1-new-syntax) that further complicates the basic question of "what is ReasonML?".
When ReasonML works, it's fantastic.
ReasonML's compilation speed is ridicuously faster than TypeScript's.
Setting up a React project is considerably less involved than using Create React App.
At the language level, you get a much richer type system than TypeScript's.
Type-safe GraphQL queries and pattern matching over values makes for a very pleasent programming environment.

However, I've found myself simply unmotivated to work on my side project.
I poke at it every couple weeks for a few hours and I invariably end up side-tracked dealing with a library compatibility issue.
While I could just stick with the set of libraries I was using six months ago and make progress with that, it's also a bit disheartening because recent BuckleScript versions have really improved the JavaScript interop and I'd hate to give those improvements up.
Then, even when things work, I find I spend a lot of time manually translating TypeScript types to ReasonML types.

I think the core problem is ReasonML is in a state right now where if you can't afford to keep up with everything going on in the ecosystem, you're going to run into confusing problems.
The community is great and will take the time to explain what the situation is, but I shouldn't have to be active on a Discord server in order to get anything done.
On the other hand, having a taste of what ReasonML provides when it works, I'm also very reticent to jettison the whole project and switch over to TypeScript.
I'm currently using TypeScript, Create React App, and Relay for a project at work and while it mostly works, it's brought a whole different set of problems I don't really want to deal with in my free time.

I wish I had more time to contribute to ReasonML.
I've been very active with many open source projects over the past two decades, so I don't mind rolling my sleeves up and helping out.
I just simply don't have the time take on this large an effort.
I'm extremely grateful to the community members that have been able to dedicate time to making ReasonML better and I hope my reflections here aren't taken as a critique of their efforts.
Building up a new language ecosystem and community is a massive undertaking largely handled by a small group of people.

Given Facebook's internal usage of ReasonML, I naïvely thought it would be at the back-half of the [early adopter stage](https://en.wikipedia.org/wiki/Crossing_the_Chasm), maybe even early majority.
But, it feels a lot more like it's still in the innovator stage.
That's okay.
Every project needs to start somewhere.
If you're comfortable with that, you can have a lot of fun working with ReasonML and helping advance the language.
If you're just looking to tinker with something, even knowing there'll be some bumps, you'll probably want to use something a bit more refined.

<hr/>

<a name="footnote_1"></a>
<sup>1</sup>
<small>
  I had intended to publish this post in early August, 2020, which was after BuckleScript announced its new syntax but before it announced its [renaming to ReScript](https://reasonml.org/blog/bucklescript-is-rebranding).
  Unfortunately, I hit some technical snags that meant this wasn't published until several days after the rename was announced.
  My initial reaction is that the rename is going to make it harder for people searching for information, as what little 3rd party content is out there will be using the old name, BuckleScript.
  I don't mean to be alarmist, but that's how other renames I've seen have gone &mdash; there's always a big thrashing period up front.
  I truly hope the intention of reducing complexity comes to fruition because the ReasonML ecosystem sorely needs it.
  For the time being I remain cautiously optimistic.
  <br/><small><a href="#summary" style="font-style: italic;">Go back</a></small>
</small>
