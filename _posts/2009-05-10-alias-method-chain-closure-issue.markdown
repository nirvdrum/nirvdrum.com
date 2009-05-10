---
layout: post
title: Nesting alias_method_chain Calls
---

Context
=======

Rails provides a nifty utility in ActiveSupport called <code>alias_method_chain</code>. For those not familiar with it, it simplifies the task of augmenting an already defined method. The newly enhanced method is aliased to the name of the original method while the original method is aliased to some other name in order that it may still be referenced (normally by the enhanced method).

More succinctly, the following call:

<code class="brush:ruby">
alias_method_chain :blah, :filter
</code>

is effectively the same as:

<code class="brush:ruby">
alias_method :blah_without_filter, :blah
alias_method :blah, :blah_with_filter
</code>

Note that these symbols are method names. It is up to you to define both methods <code>:blah</code> and <code>:blah_with_filter</code>
as these are the targets of the two <code>alias_method</code> calls.

Problem
=======

The problem with the approach taken by <code>alias_method_chain</code> is one of definition order. <code>:blah_without_filter</code> will not exist until after the <code>alias_method_chain</code> call is complete. <code>:blah_with_filter</code> must exist before the <code>alias_method_chain</code> call can begin, otherwise the <code>alias_method</code> call made internally will fail. So, if you want to call your original method from your enhanced one, you're left with having to do this dynamically.
The most common way to invoke a method dynamically is to call <code>send</code> with the method name.  E.g.:

<code class="brush:ruby">
def blah(num)
  puts num
end

def blah_with_filter(num)
  send(:blah_without_filter, num + 1)
end

alias_method_chain :blah, :filter
</code>

<code class="brush:ruby">
>> blah 3
4
=> nil
</code>

This approach will work dandily, until you need to start aliasing a method more than once.

Consider what happens when we do the following:

<code class="brush:ruby">
alias_method_chain :blah, :filter
alias_method_chain :blah, :filter
</code>

<code class="brush:ruby">
>> blah 3
SystemStackError: stack level too deep
from /Users/nirvdrum/dev/workspaces/alias_method_chain/blah.rb:8:in `send'
from /Users/nirvdrum/dev/workspaces/alias_method_chain/blah.rb:8:in `blah_without_filter'
from /Users/nirvdrum/dev/workspaces/alias_method_chain/blah.rb:8:in `send'
from /Users/nirvdrum/dev/workspaces/alias_method_chain/blah.rb:8:in `blah_without_filter'
from /Users/nirvdrum/dev/workspaces/alias_method_chain/blah.rb:8:in `send'
from /Users/nirvdrum/dev/workspaces/alias_method_chain/blah.rb:8:in `blah_without_filter'
... 8235 levels...
</code>

That’s the polite way of telling you that you have infinite recursion, not the expected value of 5.  The issue is that no closure is formed.  The following expected execution flow does not occur:

<code class="brush:ruby">
:blah_with_filter -->
:blah_without_filter (:blah_with_filter) -->
:blah_without_filter -->
:blah (original)
</code>

Instead, we have:

<code class="brush:ruby">
:blah_with_filter -->
:blah_without_filter (:blah_with_filter) -->
:blah_without_filter (:blah_with_filter) -->
:blah_without_filter (:blah_with_filter) -->
# And so on, until stack overflows.
</code>

The reasoning goes something like this:

The first <code>alias_method_chain</code> takes the original definition of <code>:blah</code> and aliases it to <code>:blah_without_filter</code>.  It then takes the definition of <code>:blah_with_filter</code> and aliases it as <code>:blah</code>.  So, when the second <code>alias_method_chain</code> call is encountered, we’re actually now aliasing the previous invocation’s <code>:blah_with_filter</code>.  The first <code>:blah_with_filter</code> (aliased as <code>:blah</code>) now gets aliased as <code>:blah_without_filter</code>.  The definition of <code>:blah_with_filter</code> gets aliased as <code>:blah</code>, as before.  So, <code>:blah_with_filter</code> delegates to <code>:blah_without_filter</code>, which incidentally is the same as <code>:blah_with_filter</code> at this stage.

There is no* clean way around this with <code>alias_method_chain</code>.  It’s a classic chicken-and-egg situation.  The best that can be done is for <code>:blah_with_filter</code> to maintain a stack of <code>UnboundMethod</code> objects in a class instance variable.  It is then the responsibility of <code>:blah_with_filter</code> to maintain this stack.

The proposed improvement to <code>alias_method_chain</code> is to yield to a block between the two <code>alias_method</code> calls.  Unfortunately, this is not backwards-compatible with Rails because <code>alias_method_chain</code> yields to a block elsewhere for reasons not quite clear to me (I believe it’s to handle method names with punctuation).

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

We can then do the following:

<code class="brush:ruby">
# Build up a proc that will construct the filtered method.
# Execution of the proc is delayed until we encounter the
# alias_method_chain call.
filtered_method_builder = Proc.new do

  # Get a reference to the unfiltered method or, more accurately,
  # the original method with all previous filters already applied.
  # This new filtered method builds up on the filters already applied.
  unfiltered_method = instance_method :blah_without_filter

  # Define the newly filtered method.
  define_method("blah_with_around_filter_wrapper") do |*args|
    unfiltered_method.bind(self).call(*args)
  end

end

alias_method_chain :blah, :filter, &amp;filtered_method_builder
alias_method_chain :blah, :filter, &amp;filtered_method_builder
</code>

The big difference here is that the "with" method is defined in between the two <code>alias_method</code> calls.  The method is defined dynamically in a block yielded by the new <code>alias_method_chain</code>.  This will allow the "with" method to reference the most recently defined "without" method.

In this admittedly convoluted example, the block is built up as a proc first.  This allows me to make the same <code>alias_method_chain</code> call without needing to duplicate code.  The proc gets a reference to <code>:blah_without_filter</code> calls it within the newly defined <code>:blah_with_filter</code>.  This forms a closure and lets each level of "with" and "without" methods to pair up, subsequently avoiding the infinite recursion problem when using just <code>send</code> alone.


Rationale
=========

At this stage, I'm sure some of you are wondering why anyone would actually want to call <code>alias_method_chain</code> twice.  Unfortunately, in order to illustrate the problem I had to use a bit of a contrived example.  One real world case where this was a problem was with the <a href="http://github.com/haruska/ninja-decorators/">ninja-decorators project</a>.  ninja-decorators gives you <code>before_filter</code>, <code>after_filter</code>, and <code>around_filter</code> functionality without having to use Rails.  With these methods you can handle cross-cutting concerns in a single location.  It is not uncommon to have something like the following:

<code class="brush:ruby">
around_filter :secure_around, [:blah]
around_filter :log_around, [:blah]
</code>

Here, we want :log_around to decorate :blah with <code>:secure_around</code> applied.  Internally, <code>around_filter</code> delegates to alias_method_chain to handle method decoration.  This is only feasible with the modified <code>alias_method_chain</code>.

Conclusion
==========

I began writing this post just to document the changes necessary to alias_method_chain in order to make ninja-decorators work.  If this work could make its way back into Rails core, great.  Otherwise, it serves as a decent rationale document.  If you've run into similar issues yourself, you should now know why and how to work around them.



* I suspect someone much smarter than me knows a way.  After a couple days on the issue, I couldn’t come up with anything.