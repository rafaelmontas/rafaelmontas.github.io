---
layout: post
title: "Implementing Data Structures in Ruby: Arrays"
slug: arrays from scratch in ruby
excerpt: "Let's strengthen our understanding of Arrays by implementing them from scratch
in Ruby. Building a solid foundation in Data Structures and Algorithms."
external_site: dev.to
external_url: https://dev.to/rmontas/implementing-data-structures-in-ruby-arrays-3oed
canonical_url: 'https://dev.to/rmontas/implementing-data-structures-in-ruby-arrays-3oed'
---


Just recently, I transitioned careers to become a Software Developer using mainly Ruby and Ruby on Rails and wanted to strengthen my understanding of Data Structures & Algorithms, which I am sure would allow me to become a better programmer. So with this in mind, I decided to read [The Well-Grounded Rubyist, Third Edition](https://www.manning.com/books/the-well-grounded-rubyist-third-edition) to better understand the basics of the Ruby language and [A Common-Sense Guide to Data Structures and Algorithms, Second Edition](https://pragprog.com/titles/jwdsal2/a-common-sense-guide-to-data-structures-and-algorithms-second-edition/) to start implementing algorithms with Space and Time Complexity in mind.

While working through these books and a few blog posts from amazing people, I learned how to implement data structures from scratch in Ruby and in this post, we will do just that by trying to replicate the inner workings and methods of the [`array`](https://ruby-doc.org/core-2.7.0/Array.html)

## What is an Array?
When a program declares an array, it allocates a contiguous set of empty cells for use in the program (Ruby implements dynamic arrays). This translates to an ordered collection of objects.

Ruby provides us with the class `Array` which we can instantiate, among other ways, by explicitly calling `Array.new` or by using the literal constructor `[]`.

## Building an Array from scratch
Let's begin by creating and initializing our class `MyArray` using a [`Hash`](https://ruby-doc.org/core-2.7.0/Hash.html) to keep track of the elements within the collection. Also, Arrays keep track of the number of objects the collection contains at all times. We will implement an `attr_reader` to be able to query instances of `MyArray` about this information using `MyArray#length`.

```ruby
class MyArray
  attr_reader :length
  
  def initialize(enum = nil)
    @length = 0
    @elements = Hash.new(nil)

    return if enum.nil?

    enum.each do |val|
      @elements[@length] = val
      @length += 1
    end
  end
end
```

We initialize our class with two instance variables, `@length`, set to `0` by default and `@elements` which is set to an empty `Hash`. Also, with this implementation, we can create a new `MyArray` containing the elements of a given enumerable object, increasing the `@length` on each iteration.

Before we move on, let's replicate even more the behavior and terminal output by creating a method that will allow us to instantiate our class like the way we instantiate `arrays` with literal constructors and specially similar to the class [`Set`](https://ruby-doc.org/stdlib-2.7.1/libdoc/set/rdoc/Set.html).

```ruby
def self.[](*ary)
  new(ary)
end

def inspect
  sprintf('#<%s: [%s]>', self.class, @elements.values.inspect[1..-2])
end

alias to_s inspect
```
`self.[](*ary)` defines a class method and thanks to [Ruby's syntactic sugar](https://blog.appsignal.com/2018/02/20/ruby-magic-syntactic-sugar-methods.html), we can now create instances of our class by calling `MyArray['foo', 'bar']`. The implementation of [`inspect`](https://www.rubyguides.com/2018/12/ruby-inspect-method/) was inspired by the way the [`Set`](https://github.com/ruby/set/blob/master/lib/set.rb) class implements it. Resulting in a terminal output as shown below.

```console
irb(main):001:0> MyArray['foo', 'bar']
=> #<MyArray: ["foo", "bar"]>
```

Now, let's start creating the methods we are going to need in order to cover the basic operations of reading, searching, inserting and deleting.

**`#[] (aliased as #slice)`**

When you have objects in an `array` you can retrieve those objects by using the `#[]` or `#slice(index)` methods. Retrieving single elements like this takes constant time in terms of Time Complexity analysis: O(1).

```ruby
# Returns the element at the given index.
def [](index)
  @elements[index]
end

alias slice []
```

**`#[]=(index, element)`**

Let's now implement its setter equivalent `#[]=(index, element)` which allows us to set an element at a given index. In the case the given index is out of range for `MyArray` we set those slots between the given index and the last element to `nil`.

```ruby
def []=(index, element)
  while index > @length
    @elements[@length] = nil
    @length += 1
  end
  @elements[index] = element
  @length += 1 if index == @length
end
```

**`#push(element) (aliased as #<<)`**

To add an element to the end of the collection, you can use [`#push(element)`](https://ruby-doc.org/core-2.7.0/Array.html#method-i-push) or [`#<<`](https://ruby-doc.org/core-2.7.0/Array.html#method-i-3C-3C). Returning the collection itself. Here, knowing `arrays` have zero based indexing, we can implement it by setting the given element at the `@length` index, which is one slot to the right of the last element in the collection.

```ruby
# Constant Time to add element to the end of collection
# Time Complexity: O(1)
def push(element)
  @elements[@length] = element
  @length += 1
  return self
end

alias << push
```

**`#pop`**

Here we remove the last element of the collection, reduce the `@length` instance variable by 1, and return the deleted element back which is accomplished by saving said element in a variable before removing it.

```ruby
# Constant time to remove last element. Time Complexity: O(1)
def pop
  latest_element = @elements[@length -1]
  @elements.delete(@length -1)
  @length -= 1

  return latest_element
end
```

**`#unshift(value)`** and **`#shift`**

As you have seen, appending and removing elements from the end of the collection is fairly straight forward. Now, to accomplish the same operations at the beginning of the collection, we have to shift the remaining elements either to the right (if we are appending [`#unshift(value)`](https://ruby-doc.org/core-2.7.0/Array.html#method-i-unshift)) or to the left (if we are removing [`#shift`](https://ruby-doc.org/core-2.7.0/Array.html#method-i-shift)). So, we will be creating a couple of `private` methods that will help us do just that.

**`#unshift(value)`**

```ruby
class MyArray
  ...
  
  # Linear time to add at the beginning of collection.
  # Moves other values. Time Complexity: O(n)
  def unshift(value)
    self.unshift_elements(0, value)
    return self
  end

  private
  
  def unshift_elements(index, value)
    counter = @length - 1
    while counter >= index
      @elements[counter + 1] = @elements[counter]
      counter -= 1
    end
    @elements[index] = value
    @length += 1
  end
end
```

Here, when we call `#unshift(value)`, we start shifting by 1 index to the right all elements of the collection, starting from the last one so we don't lose any value in the process. Once all elements are shifted, we set the element at index 0 to the value being inserted and increment the collection's `length` by 1.

**`#shift`**

Similarly, to remove the first element from the collection, we need to shift all elements but this time to the left. Basically, we start shifting left from index 1 until we have shifted the last element which will be duplicated, so we remove it and decrease our collection's `length` by 1.

```ruby
class MyArray
  ...

  # Linear time to remove at the beginning of the collection.
  # Shifts other values. Time Complexity: O(n)
  def shift
    first_element = @elements[0]
    self.shift_elements(0)
    return first_element
  end

  private

  ...

  def shift_elements(index)
    counter = index
    while counter < (@length - 1)
      @elements[counter] = @elements[counter + 1]
      counter += 1
    end
    @elements.delete(@length - 1)
    @length -= 1
  end
end
```

## Iterating over `MyArray`

In order to iterate over the collection, we need to implement the [`#each`](https://ruby-doc.org/core-2.7.0/Array.html#method-i-each) method. The idea behind [`#each`](https://ruby-doc.org/core-2.7.0/Array.html#method-i-each) is simple: by calling it on `#MyArray` collection it will yield back each item in it to the code block provided, one at a time. The return value of [`#each`](https://ruby-doc.org/core-2.7.0/Array.html#method-i-each), when it’s given a block, is its receiver, the original collection. When it isn’t given a block, it returns an enumerator, which is what allows us to chain other methods like [`Enumerator#with_index`](https://ruby-doc.org/core-2.5.0/Enumerator.html#method-i-with_index).

```ruby
# Calls the given block once for each element in the collection, passing
# the element as a parameter. Return an enumerator if no block is given.
def each
  return self.enum_for(__method__) unless block_given?

  counter = 0
  while counter < @length
    yield @elements[counter]
    counter += 1
  end

  return self
end
```

Having done that, we can implement the method [`#delete`](https://ruby-doc.org/core-2.7.0/Array.html#method-i-delete), which makes use of `#each` by first finding the index of the given element, removing it from the collection, and then shifting all remaining elements to the left.

**`#delete(element)`**

```ruby
# Linear time to remove at an arbitrary location. Shifts other values.
# Time Complexity: O(n)
def delete(element)
  self.each.with_index do |value, index|
    if value == element
      self.shift_elements(index)
      return value
    end
  end
    
  return nil
end
```
The journey of learning the basics of Ruby and the inner working of collection classes like the `Array` has been an amazing experience and I'm looking forward to share all my learnings in subsequent posts like this one. I'll let you with the code for our entire `MyArray` class below and a link to the [Github repo](https://github.com/rafaelmontas/ds-algorithms-ruby):

```ruby
class MyArray
  attr_reader :length

  def initialize(enum = nil)
    @length = 0
    @elements = Hash.new(nil)

    return if enum.nil?

    enum.each do |val|
      @elements[@length] = val
      @length += 1
    end
  end

  def self.[](*ary)
    new(ary)
  end
  
  def inspect
    sprintf('#<%s: [%s]>', self.class, @elements.values.inspect[1..-2])
  end
  
  alias to_s inspect

  # Returns the element at the given index.
  def [](index)
    @elements[index]
  end

  alias slice []

  def []=(index, element)
    while index > @length
      @elements[@length] = nil
      @length += 1
    end
    @elements[index] = element
    @length += 1 if index == @length
  end

  # Constant Time to add element to the end of collection
  # Time Complexity: O(1)
  def push(element)
    @elements[@length] = element
    @length += 1
    return self
  end

  alias << push

  # Constant time to remove last element. Time Complexity: O(1)
  def pop
    latest_element = @elements[@length -1]
    @elements.delete(@length -1)
    @length -= 1

    return latest_element
  end

  # Linear time to add at the beginning of collection.
  # Moves other values. Time Complexity: O(n)
  def unshift(value)
    self.unshift_elements(0, value)
    return self
  end

  # Linear time to remove at the beginning of the collection.
  # Shifts other values. Time Complexity: O(n)
  def shift
    first_element = @elements[0]
    self.shift_elements(0)
    return first_element
  end

  # Calls the given block once for each element in the collection, passing
  # the element as a parameter. Return an enumerator if no block is given.
  def each
    return self.enum_for(__method__) unless block_given?

    counter = 0
    while counter < @length
      yield @elements[counter]
      counter += 1
    end

    return self
  end

  # Linear time to remove at an arbitrary location. Shifts other values.
  # Time Complexity: O(n)
  def delete(element)
    self.each.with_index do |value, index|
      if value == element
        self.shift_elements(index)
        return value
      end
    end

    return nil
  end

  private

  def unshift_elements(index, value)
    counter = @length - 1
    while counter >= index
      @elements[counter + 1] = @elements[counter]
      counter -= 1
    end
    @elements[index] = value
    @length += 1
  end

  def shift_elements(index)
    counter = index
    while counter < (@length - 1)
      @elements[counter] = @elements[counter + 1]
      counter += 1
    end
    @elements.delete(@length - 1)
    @length -= 1
  end
end
```
## Sources of Inspiration
[Data Structures From Scratch: Array](https://dev.to/torianne02/data-structures-from-scratch-array-24d5)
[Exploring Data Structures From a Ruby Background, Pt. 1: Arrays](https://dev.to/isalevine/exploring-data-structures-from-a-ruby-background-pt-1-arrays-5bma)
[Ruby's `Set` source code](https://github.com/ruby/set/blob/master/lib/set.rb)