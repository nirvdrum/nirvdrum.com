---
layout: post
title: Nesting alias_method_chain Calls
---

Introduction
------------

Rails provides a nifty utility in ActiveSupport called `alias_method_chain`. For those not familiar with it, it simplifies the task of "replacing" an already defined method with an augmented one. The new method is aliased to the name of the original method and the original method is aliased to some other name in order that it may still be referenced.

More succinctly, the following call:

<pre class="brush:ruby">
alias_method_chain :number_printer, :filter
</pre>

is effectively the same as:

<pre class="brush:ruby">
alias_method :number_printer_without_filter, :number_printer
alias_method :number_printer, :number_printer_with_filter
</pre>

Fig. 1 illustrates how the `alias_method_chain` call changes references to the method definitions like so:

<div class="figure">
  <img src="/images/alias-method-chain-closure-issue/alias_method_chain.png" />
  
  Fig. 1: Results of alias_method_chain call.
</div>

Now the original method defined as `:number_printer` is referenced as `:number_printer_without_filter`.  `:number_printer` now points to the method definition for `:number_printer_with_filter`, which can be referenced as either `:number_printer` or `:number_printer_with_filter`.

This implies that prior to the execution of the `alias_method_chain` call, you must define both methods `:number_printer` and `:number_printer_with_filter`.

Motivating Example
------------------

The <a href="http://github.com/haruska/ninja-decorators/">ninja-decorators project</a> relies heavily on `alias_method_chain` and its usage will be used as the example throughout the remainder of the article.  ninja-decorators gives you `before_filter`, `after_filter`, and `around_filter` functionality outside of Rails controllers.  With these methods you can handle cross-cutting concerns in a class located elsewhere in your Rails app or without having to use Rails at all.  Using the standard examples of security and logging as cross-cutting concerns, we have something like the following:

<pre class="brush:ruby">
around_filter :secure_around, [:number_printer]
around_filter :log_around, [:number_printer]
</pre>

Here, we want `:log_around` to decorate `:number_printer` with `:secure_around` applied.  Internally, `around_filter` delegates to `alias_method_chain` to handle method decoration.

Problem
-------

The problem with the implementation of `alias_method_chain` is one of definition order with regards to its two internal `alias_method` calls.  If the new head of the chain is an enhancement of an existing method in the chain, there likely exists a coupling between the two.  Since the `alias_method_chain` call is effectively atomic, however, this complicates how the two methods reference each other. Fig. 2 shows the intermittent states between each of the two `alias_method` calls made internally by `alias_method_chain`.

<div class="figure">
  <img src="/images/alias-method-chain-closure-issue/alias_method_chain_detailed.png" />
  
  Figure 2: Detailed breakdown of alias_method_chain mechanics.
</div>

`:number_printer_without_filter` will not exist until after the `alias_method_chain` call is complete. `:number_printer_with_filter` must exist before the `alias_method_chain` call can begin, otherwise the second `alias_method` call made internally will fail.  As a consequence, `:number_printer_with_filter` must call `:number_printer_without_filter` dynamically.  As long as you only use one level of `alias_method_chain` calls, this isn't a problem.  With multiple levels of chaining, however, dynamic calls like this fall apart.

To make the discussion a little more concrete, we'll use the following example adapted from the ninja-decorators project.  It is a bit contrived, but should serve well enough as a basis for discussion.

<pre class="brush:ruby">
require 'activesupport'
  
class NumberFun
  def self.around_filter(around_method, method_names)
    method_names.each do |meth|
      define_method("#{meth}_with_around_filter") do |*args|        
        send(around_method, *args) do |*ar_args|
          send("#{meth}_without_around_filter", *ar_args)
        end
      end

      alias_method_chain meth, :around_filter
    end
  end

  def increment_filter(num)
    yield(num + 1)
  end

  def number_printer(num)
    puts num
  end

  def square_printer(num)
    puts num * num
  end

  around_filter :increment_filter, [:number_printer, :square_printer]
end
</pre>

`:number_printer` and `:square_printer` are two simple methods.  They take a number in and print out its value or its square, respectively.  `:increment_filter` is a simple "around filter"; it augments a method by incrementing the input argument by 1 before executing the original method.  Running both methods will produce the following in IRB:

<pre class="brush:ruby">
>> NumberFun.new.number_printer 3
4
=> nil
>> NumberFun.new.square_printer 5
36
=> nil
</pre>

`around_filter` is where all the hard work is being done and is where `alias_method_chain` is employed.  It takes as its arguments a filter method name and a list of method names to decorate with that filter.  For each method to decorate it defines the required "with" method for `alias_method_chain`.  This newly defined method will call the filter method (`:increment_filter` in this case), which will in turn call the original, undecorated method (`:number_printer_without_increment_filter` or `:square_printer_without_increment_filter`) as a block.  Once the "with" method is defined,  `alias_method_chain` is called so that the original method name can be used to transparently call the newly decorated method.

While convoluted (don't worry, it's wrapped up a library), this approach will work dandily until you need to start decorating a method more than once.  For the sake of the example, we'll pretend that we actually want to increment each input argument as two.  In reality, well'd likely want to apply a completely different filter.  The outcome is precisely the same, but to keep things simple, we'll just apply the `:increment_filter` twice:

<pre class="brush:ruby">
around_filter :increment_filter, [:number_printer, :square_printer]
around_filter :increment_filter, [:number_printer, :square_printer]
</pre>

Running this through IRB again, we'd likely expect to see `:number_printer` print out `2 + num` for argument `num`.  Instead, the session looks like this:

<pre class="brush:ruby">
>> NumberFun.new.number_printer 3
SystemStackError: stack level too deep
	from /Users/nirvdrum/dev/workspaces/alias_method_chain/blah.rb:6:in `send'
	from /Users/nirvdrum/dev/workspaces/alias_method_chain/blah.rb:6:in `number_printer_with_around_filter'
	from /Users/nirvdrum/dev/workspaces/alias_method_chain/blah.rb:7:in `number_printer_without_around_filter'
	from /Users/nirvdrum/dev/workspaces/alias_method_chain/blah.rb:7:in `send'
	from /Users/nirvdrum/dev/workspaces/alias_method_chain/blah.rb:7:in `number_printer_with_around_filter'
	from /Users/nirvdrum/dev/workspaces/alias_method_chain/blah.rb:16:in `increment_filter'
	from /Users/nirvdrum/dev/workspaces/alias_method_chain/blah.rb:6:in `send'
	from /Users/nirvdrum/dev/workspaces/alias_method_chain/blah.rb:6:in `number_printer_with_around_filter'
	from /Users/nirvdrum/dev/workspaces/alias_method_chain/blah.rb:7:in `number_printer_without_around_filter'
	from /Users/nirvdrum/dev/workspaces/alias_method_chain/blah.rb:7:in `send'
	from /Users/nirvdrum/dev/workspaces/alias_method_chain/blah.rb:7:in `number_printer_with_around_filter'
	from /Users/nirvdrum/dev/workspaces/alias_method_chain/blah.rb:16:in `increment_filter'
	from /Users/nirvdrum/dev/workspaces/alias_method_chain/blah.rb:6:in `send'
	from /Users/nirvdrum/dev/workspaces/alias_method_chain/blah.rb:6:in `number_printer_with_around_filter'
	from /Users/nirvdrum/dev/workspaces/alias_method_chain/blah.rb:7:in `number_printer_without_around_filter'
	from /Users/nirvdrum/dev/workspaces/alias_method_chain/blah.rb:7:in `send'
... 7586 levels...
</pre>

That's the polite way of telling you that you have infinite recursion, not the expected value of 5.  The issue is that each `around_filter` call defines a new "with" method that calls the "without" method dynamically.  Calling a method by name with `send`, however, only calls the one at the current lexical scope.  Meanwhile, each call to `alias_method_chain` changes the alias target of the "without" method.  As such, we have the execution flow illustrated in Fig. 3 rather than the expected one in Fig. 4.

<div style="float: left; width: 100%; margin: 10px;">
  <div class="figure" style="float: left; width: 50%;">
    <img src="/images/alias-method-chain-closure-issue/actual_nesting_behavior.png" />
  
    Figure 3: Actual behavior when chaining <code>alias_method_chain</code> calls.
  </div>

  <div class="figure" style="float: left;">
    <img src="/images/alias-method-chain-closure-issue/expected_nesting_behavior.png" />
  
    Figure 4: Expected behavior when chaining <code>alias_method_chain</code> calls.
  </div>
  
  <p style="clear: both;" />
</div>

The key thing to note about the code behavior observed in Fig. 3 is that the second `alias_method_chain` call will alias `:number_printer` to `:number_printer_without_filter`, just like the first call will.  However, after the first `alias_method_chain` call is made, `:number_printer` is aliased to the definition `:number_printer_with_filter` (as seen in Fig. 2).  Calling `:number_printer` at this point will call `:number_printer_with_filter` because of the decoration and, subsequently, `:number_printer_with_filter` will call `:number_printer_without_filter`, the latter of which is now pointing at the definition of `:number_printer_with_filter` as well.  That's a lot of words to say that we end up in a situation where `:number_printer_with_filter` calls itself inadvertently and there's no base case to break out.

There is no* clean way around this with `alias_method_chain`.  It's a classic chicken-and-egg situation.  The best that can be done is for `around_filter` to maintain a stack of `UnboundMethod` objects in a class instance variable.  While doable, this resource management is error-prone and would have to be replicated by any method affected by the problem.  Effectively, it is the same process the VM would normally perform in managing stack frames at each recursion level, so it's best to let the VM do it.

The problem can be averted with a minor change in `alias_method_chain`.  The idea is to yield to a block between the two `alias_method` calls, allowing the proper formation of closures to the "without" method.  Unfortunately, this is not backwards-compatible with Rails because `alias_method_chain` yields to a block elsewhere for reasons that are not quite clear to me (I believe it's to handle method names with punctuation).

A simplified definition is thus:

<pre class="brush:ruby">
def alias_method_chain(target, feature)
  with_method = "#{target}_with_#{feature}"
  without_method = "#{target}_without_#{feature}"

  alias_method without_method, target
  yield if block_given?
  alias_method target, with_method
end
</pre>

The block passed to alias_method_chain can then take care of the creation of the "with" method, which will have access to the "without" method at the current level.  Breaking away from `around_filter`, we can more easily see how nested `alias_method_chain` calls work with the new definition:

<pre class="brush:ruby">
class MoreNumberFun
  # Build up a proc that will construct the filtered method
  # Execution of the proc is delayed until we encounter the alias_method_chain call.
  filtered_method_builder = Proc.new do

    # Get a reference to the unfiltered method or, more accurately, the original method with
    # all previous filters already applied.  This new filtered method builds up on the filters
    # already applied.
    unfiltered_method = instance_method :number_printer_without_filter

    # Define the newly filtered method.
    define_method("number_printer_with_filter") do |*args|
      unfiltered_method.bind(self).call(args.first + 1)
    end
  end

  def number_printer(num)
    puts num
  end

  alias_method_chain :number_printer, :filter, &filtered_method_builder
  alias_method_chain :number_printer, :filter, &filtered_method_builder
end
</pre>

In this admittedly convoluted example, the block passed to the `alias_method_chain` calls is built up as a proc first.  This allows us to make the same `alias_method_chain` calls without needing to duplicate code.  The proc gets a reference to `:number_printer_without_filter` and calls it within the newly defined `:number_printer_with_filter`, which for simplicity in the example, provides the same behavior that `:increment_filter` previously did.  This forms a closure and lets each level of "with" and "without" methods to pair up, subsequently avoiding the infinite recursion problem when using just `send` alone.

Running in IRB now, we get the expected behavior of print out of `2 + num` for argument `num`, rather than the stack overflow exception we previously experienced:

<pre class="brush:ruby">
>> MoreNumberFun.new.number_printer 3
5
=> nil
</pre>

Conclusion
----------

I began writing this post just to document the changes necessary to alias_method_chain in order to make ninja-decorators work.  If this work could make its way back into Rails core, great.  Otherwise, it serves as a decent rationale document.  If you've run into similar issues yourself, you should now know why and how to work around them.  One issue not addressed here is reordering the chain or removing links from the chain.  Since each link has a tight coupling at the time of definition, altering the chain via anything other than an append/prepend may be confusing.



\* I suspect someone much smarter than me knows a way.  After a couple days on the issue, I couldn't come up with anything.
