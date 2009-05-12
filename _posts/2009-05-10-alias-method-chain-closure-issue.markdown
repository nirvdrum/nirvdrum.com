---
layout: post
title: Nesting alias_method_chain Calls
---

Introduction
============

Rails provides a nifty utility in ActiveSupport called `alias_method_chain`. For those not familiar with it, it simplifies the task of augmenting an already defined method. The newly enhanced method is aliased to the name of the original method and the original method is aliased to some other name in order that it may still be referenced.

More succinctly, the following call:

<code class="brush:ruby">
alias_method_chain :number_printer, :filter
</code>

is effectively the same as:

<code class="brush:ruby">
alias_method :number_printer_without_filter, :number_printer
alias_method :number_printer, :number_printer_with_filter
</code>

Note that these symbols are method names. It is up to you to define both methods `:number_printer` and `:number_printer_with_filter`.

Motivating Example
==================

The <a href="http://github.com/haruska/ninja-decorators/">ninja-decorators project</a> relies heavily on `alias_method_chain` and its usage will be used as the example throughout the remainder of the article.  ninja-decorators gives you `before_filter`, `after_filter`, and `around_filter` functionality outside of Rails controllers.  With these methods you can handle cross-cutting concerns in a class located elsewhere in your Rails app or without having to use Rails at all.  Using the standard examples of security and logging as cross-cutting concerns, we have something like the following:

<code class="brush:ruby">
around_filter :secure_around, [:number_printer]
around_filter :log_around, [:number_printer]
</code>

Here, we want `:log_around` to decorate `:number_printer` with `:secure_around` applied.  Internally, `around_filter` delegates to `alias_method_chain` to handle method decoration.

Problem
=======

The problem with the implementation of `alias_method_chain` is one of definition order. `:number_printer_without_filter` will not exist until after the `alias_method_chain` call is complete. `:number_printer_with_filter` must exist before the `alias_method_chain` call can begin, otherwise the `alias_method` call made internally will fail.  In most cases, this isn't a problem in and of itself.  The problem is revealed once you need to dynamically invoke a method.  E.g.:

<code class="brush:ruby">
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
</code>

<code class="brush:ruby">
>> number_printer 3
4
=> nil
>> square_printer 4
25
=> nil
</code>

If your mindgrapes are hurting, you're in good company.  It's confusing code, but it's also rather powerful.  Likewise, it's wrapped in a library so we never need to look at it again.  It's important to understand it for the purpose of this article, however.

`:number_printer` and `:square_printer` are two simple methods.  They take a number in and print out its value or its square, respectively.  `:increment_filter` is a simple "around filter"; it augments a method by incrementing the input argument by 1.  This is why the IRB output looks like its off by 1.

`around_filter` is where all the hard work is being done.  It takes as its arguments a filter method name and a list of method names to decorate with that filter.  For each method to decorate it defines a new "with" method for `alias_method_chain`.  This newly defined method will call the filter, which will in turn call the original, undecorated method (the "without" method).  Once this is all done, the `alias_method_chain` is made so that the original method name can be used to transparently call the decorated method.

This approach will work dandily, until you need to start decorating a method more than once.  For the sake of the example, I'll pretend that I actually want to increment each input argument as two.  In reality, I'd likely want to apply a completely different filter.  The outcome is precisely the same, but to keep things simple, I'll just apply the `:increment_filter` twice:

<code class="brush:ruby">
around_filter :increment_filter, [:number_printer, :square_printer]
around_filter :increment_filter, [:number_printer, :square_printer]
</code>

<code class="brush:ruby">
>> number_printer 3
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
</code>

That's the polite way of telling you that you have infinite recursion, not the expected value of 5.  The issue is that each `around_filter` call defines a new "with" method that calls the "without" method dynamically.  Calling a method by name with `send`, however, only calls the one at the current lexical scope.  Meanwhile, each call to `alias_method_chain` changes the alias target of the "without" method.  As such, the following expected execution flow does not occur (levels at which a method is defined/aliased are indicated after method name, arrows indicate method execution flow):

<code class="brush:ruby">
:number_printer(L2) -->
  :number_printer_with_filter(L2) --> 
    :number_printer_without_filter(L2) # :number_printer_with_filter(L1) -->
      :number_printer_without_filter(L1) -->
        :number_printer (L1)
</code>

Instead, we have:

<code class="brush:ruby">
:number_printer(L2) -->
  :number_printer_with_filter(L2) -->
    :number_printer_without_filter(L2) # :number_printer_with_filter (L1) -->
      :number_printer_without_filter(L2) # :number_printer_with_filter(L1) -->
        :number_printer_without_filter(L2) # :number_printer_with_filter(L1) -->
          # And so on, until stack overflows.
</code>

The key thing to note is that the second `alias_method_chain` call will alias `:number_printer` to `:number_printer_without_filter`, just like the first call will.  However, at this point, `:number_printer` is aliased to the definition `:number_printer_with_filter`.  Calling `:number_printer` will call `:number_printer_with_filter` because of the decoration and `:number_printer_with_filter` will call `:number_printer_without_filter`, the latter of which is now pointing at the definition of `:number_printer_with_filter` as well.  That's a lot of words to say that we end up in a situation where `:number_printer_with_filter` calls itself inadvertently and there's no base case to break out.

There is no* clean way around this with `alias_method_chain`.  It's a classic chicken-and-egg situation.  The best that can be done is for `around_filter` to maintain a stack of `UnboundMethod` objects in a class instance variable.  While doable, this resource management is error-prone and would have to be replicated by any method affected by the problem.  Effectively, it is the same process the VM would normally perform in managing stack frames at each recursion level, so let's let the VM do it.

The change necessary in `alias_method_chain` is quite minor, but allows the proper formation of closures to the "without" method.  The idea is to yield to a block between the two `alias_method` calls.  Unfortunately, this is not backwards-compatible with Rails because `alias_method_chain` yields to a block elsewhere for reasons that ar not quite clear to me (I believe it's to handle method names with punctuation).

A simplified definition is thus:

<code class="brush:ruby">
def alias_method_chain(target, feature)
  with_method = "#{aliased_target}_with_#{feature}#{punctuation}"
  without_method = "#{aliased_target}_without_#{feature}#{punctuation}"

  alias_method without_method, target
  yield if block_given?
  alias_method target, with_method
end
</code>

The block passed to alias_method_chain can then take care of the creation of the "with" method, which will have access to the "without" method at the current level.  Breaking away from `around_filter` momentarily, we can then do the following:

<code class="brush:ruby">
# Build up a proc that will construct the filtered method.  Execution of the proc is delayed
# until we encounter the alias_method_chain call.
filtered_method_builder = Proc.new do
  # Get a reference to the unfiltered method or, more accurately, the original method with
  # all previous filters already applied.  This new filtered method builds up on the filters
  # already applied.
  unfiltered_method = instance_method :number_printer_without_filter

  # Define the newly filtered method.
  define_method("number_printer_with_around_filter_wrapper") do |*args|
    unfiltered_method.bind(self).call(*args)
  end
end

alias_method_chain :number_printer, :filter, &filtered_method_builder
alias_method_chain :number_printer, :filter, &filtered_method_builder
</code>

In this admittedly convoluted example, the block is built up as a proc first.  This allows me to make the same `alias_method_chain` call without needing to duplicate code.  The proc gets a reference to `:number_printer_without_filter` and calls it within the newly defined `:number_printer_with_filter`.  This forms a closure and lets each level of "with" and "without" methods to pair up, subsequently avoiding the infinite recursion problem when using just `send` alone.


Conclusion
==========

I began writing this post just to document the changes necessary to alias_method_chain in order to make ninja-decorators work.  If this work could make its way back into Rails core, great.  Otherwise, it serves as a decent rationale document.  If you've run into similar issues yourself, you should now know why and how to work around them.

* I suspect someone much smarter than me knows a way.  After a couple days on the issue, I couldn't come up with anything.
