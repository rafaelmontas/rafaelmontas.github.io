---
layout: post
title: "Getting to know  Ruby's Enumerable Module"
header: "Getting to know  Ruby's Enumerable Module"
slug: "getting to know  ruby enumerable module"
excerpt: "Collection classes in Ruby typically include (or extend) Enumerable.
If it has an #each method it probably mixes in the Enumerable Module."
---


`Enumerable` is a module in the Ruby standard library. Classes that include it have
to define an instance method called `each`, which yields the elements of your
collection in succession. Once the iterator is defined and `Enumerable` is mixed in, the class
now supports all sorts of collection-related behavior. Also, keep in mind, that
although `Enumerable` adds common functionality to each of the collection classes
that includes it, each of them actually overrides some of the methods.

```ruby
class MyArray
  include Enumerable

  def each
    # yield elements of the collection
  end
end
```

You can query which methods `Enumerable` provides by sending it the message
`instance_methods` and passing the `false` argument so you get only methods
defined in the module itself.

```console
irb(main):001:0> Enumerable.instance_methods(false)
```

In order to understand `Enumerable` at a deeper level, let's write a few its
methods while trying to cover as many operations as posible.

## Methods for Querying

Some Enumerable methods return information about the collection other than the elements
themselves. Some return either true or false if one or more element matches certain
criteria.

**`#include?`**

Returns `true` if the item yielded to the block is equal the argument passed, `false`
otherwise.

```ruby
def include?(value)
  self.each { |item| return true if item == value }
  return false
end
```

This works just fine for `Arrays` but if we were querying a `Hash`, we would need to
adjust for the fact that when you iterate through a `hash` with `each` it yields back
one key/value pair at a time (two-element arrays). The `Hash#include?` method checks
for key inclusion

**`#any?`**

Returns `true` if any element meets a specified criteria, `false` otherwise.

```ruby
def any?
  return self.length >= 1 unless block_given?

  self.each do |element|
    return true if yield(element)
  end

  return false
end
# => arr.any? { |n| n > 3 }
# => true
```

**`#count`**

Return the count of elements based on a criteria passed as argument or a block, if
given.

```ruby
def count(value=nil)
  counter = 0
  self.each do |element|
    if value
      counter += 1 if element == value
    elsif block_given?
      counter += 1 if yield(element)
    else
      counter += 1
    end
  end

  warn("warning: given block not used") if value && block_given?
  return counter
end
```

**`#tally`**

Counts the ocurrences of each element. Returning a `hash` with the collection's elements
as keys and corresponding counts as values.

```ruby
def tally
  tally = Hash.new(0)
  self.each { |element| tally[element] += 1 }

  return tally
end
```

## Methods for Searching and Filtering

