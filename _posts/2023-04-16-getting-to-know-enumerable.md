---
layout: post
title: "Getting to know  Ruby's Enumerable Module"
header: "Getting to know  Ruby's Enumerable Module"
slug: "getting to know  ruby enumerable module"
excerpt: "Collection classes in Ruby typically include (or extend) Enumerable.
If it has an #each method it probably mixes in the Enumerable Module."
image: /assets/images/getting-to-know-rubys-enumerable-module.png
---


`Enumerable` is a module in the Ruby standard library. Classes that include it have
to define an instance method called `each`, which yields the elements of the collection
in succession. Once the iterator is defined and `Enumerable` is mixed in, the class
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

You can query which methods `Enumerable` provides by sending the message
`instance_methods` to it and passing the `false` argument so you get only methods
defined in the module itself.

```console
irb(main):001:0> Enumerable.instance_methods(false)
```

In order to understand `Enumerable` at a deeper level, let's write a few of its
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

# (1..5).include?(2)
# => true
```

This works just fine for `Arrays` but if we were querying a `Hash`, we would need to
adjust for the fact that when you iterate through a `Hash` with `each` it yields back
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
# => (1..5).any? { |n| n > 3 }
# => true
```

**`#count`**

Returns the count of elements based on a criteria passed as argument or a block, if
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

# (1..5).count { |value| value > 3 }
# => 2
```

**`#tally`**

Counts the ocurrences of each element. Returning a `Hash` with the collection's elements
as keys and corresponding counts as values.

```ruby
def tally
  tally = Hash.new(0)
  self.each { |element| tally[element] += 1 }

  return tally
end

# [1, 2, 2, 2, 3, 3, 4].tally
# => {1=>1, 2=>3, 3=>2, 4=>1}
```

## Methods for Searching and Filtering

It's common to want to filter a collection of objects based on a selection criteria. We'll
look at several facilities for filtering and searching collections. All of them
expect a code block, where you difine your selection criteria (your tests for inclusion
or exclusion).

**`#find` aliased as `#detect`**

Returns the first element in the collection for which the code block returns `true`. If no
code block is provided, it returns an instance of `Enumerator`.

```ruby
def find
  return self.enum_for(__method__) unless block_given?
  self.each { |element| return element if yield(element) }

  return nil
end

alias :detect :find

# (1..5).find {|value| value > 2}
# => 3
```

**`#find_all` aliased as `#select`**

Returns an `Array` containing all the elements of the original collection that matched
the criteria in the code block. If no matching elements are found, it returns an empty
collection. Also, if no code block is provided, it returns an instance of `Enumerator`.

```ruby
def find_all
  return self.enum_for(__method__) unless block_given?

  matches = []
  self.each do |element|
    matches << element if yield(element)
  end

  return matches
end

alias :select :find_all

# (1..9).find_all {|value| value % 3 == 0 }
# => [3, 6, 9]
```

**`#find_index`**

Returns the index of the first element that meets the specified criteria, or `nil` if
no element matches the criteria. It returns an instance of `Enumerator` if neither a code
block nor an argument are provided.

For this method to work, make sure your `#each` implementation returns an
`Enumerator` when no code block is provided. This will allow us to chain methods, as
in `.each.with_index`.

```ruby
def find_index(value=nil)
  return self.enum_for(__method__) unless value || block_given?

  self.each.with_index do |element, index|
    if value
      return index if element == value
    else
      return index if yield(element)
    end
  end
  
  return nil
end

# (1..5).find_index(2)
# => 1
```

**`#group_by`**

When given a code block, it returns a `Hash`. For each unique value returned by the
block, the results hash gets a key; the value for that key is an `Array` of all the
elements of the collection for which the block returned that value. If no code block is
given, it returns an instance of `Enumerator`.

```ruby
def group_by
  return self.enum_for(__method__) unless block_given?

  groups = Hash.new([])
  self.each do |element|
    groups[yield(element)] << element
  end

  return groups
end

# colors = [{group: 'primary'}, {group: 'secondary'}, {group: 'primary'}]
# colors.group_by { |value| value[:group] }
# => {"primary"=>[{:group=>"primary"}, {:group=>"primary"}], "secondary"=>[{:group=>"secondary"}]}
```

**`#map` aliased as `#collect`**

Returns an `Array` when given a block. Returns and `Enumerator` otherwise. The returned
array is always the same size as the original collection. Its elements are the result
of calling the code block on each element in the original object.

```ruby
def map
  return self.enum_for(__method__) unless block_given?

  collection = []
  self.each { |element| collection << yield(element) }

  return collection
end

alias :collect :map

# (1..5).map { |value| value * value }
# => [1, 4, 9, 16, 25]
```

**`#inject` aliased as `#reduce`**

Works by initializing an accumulator object, performs a calculation on each iteration
and reset the accumulator to the result of that calculation. Returns the return value
from the last block call.

```ruby
def inject(operand=0)
  accumulator = operand
  self.each do |element|
    accumulator = yield(accumulator, element)
  end

  return accumulator
end

alias :reduce :inject

# (1..5).inject { |sum, value| sum + value }
# => 15
```

We've gotten to know `Enumerable` in a more detailed way and it will better
positioned us to handle everyday programming as we’ll use the enumeration-related
facilities of the language virtually every time we write a Ruby program.
