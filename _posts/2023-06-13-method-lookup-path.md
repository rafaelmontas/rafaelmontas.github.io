---
layout: post
title: "Ruby's method-lookup path and Object Individuation"
header: "Ruby's method-lookup path and Object Individuation"
slug: "ruby method-lookup path"
excerpt: "Objects seek their methods in both classes and superclasses,
all the way up the inheritance tree; let's see how this method-lookup
process works when modules are also involved."
image: /assets/images/method-lookup-path.png
---


Objects, in Ruby, don't "have" methods but, rather, find them by searching its
class and that class's superclass, and onward, up to the `Object` or even `BasicObject`
class, or in a module tha has been mixed into any of those classes.

Most of the time, you’ll use the dot operator "`.`" (There's also `send`, `__send__`, and
`public_send` alternatives) to "prompt" the object to go find a method
(send messages to objects). In practice, the message being sent is the name of a method.

## Demonstration of basic module inclusion and inheritance

Let's write some basic classes, modules, and methods so we can easily visualize the logic
and mechanincs of method lookup and hopefully better understand how objects find methods.

```ruby
module M
  def hello
    puts "Hello from module M"
  end
end

class C
  include M
end
class D < C; end

object = D.new
object.hello

# Hello from module M
# => nil
```

The search ends when the method being searched for is found, or with an error condition
if it isn't found. This error condition is triggered by the `method_missing` method.

> The `Kernel` module provides an instance method called `method_missing`. This method
is executed whenever an object receives a message that doesn’t match a method anywhere
in the object’s method-lookup path.

## How far does the method search go?

However many classes and modules it may cross along the way, the search for a method
can always go as far up as `BasicObject`, which has a few instance methods. But to
understand the common behavior of all Ruby objects, you have to look at `Object` or
more precisely, you have to look at `Kernel` where most of Ruby's fundamental methods
are defined. And because `Object` mixes in `Kernel`, all instances of `Object` and
all descendants of it have access to the instance methods in `Kernel`.

The illustration below show the method search path for the code example above all
the way up the chain.

![Method-lookup path](/assets/images/method-lookup-article/method-lookup.svg)

## Overriding methods

Since the method search process passes through multiple classes and modules, an
object can have multiple methods with the same name in its method-lookup path.
Still, if the object's method-lookup path includes two or more same-named
methods, the first one encountered is executed.

```ruby
module Interest
  def calculate_interest
    puts "We are in module Interest"
  end
end

class Account
  include Interest

  def calculate_interest
    puts "We are in class Account"
  end
end

account = Account.new
account.calculate_interest

# We are in class Account
# => nil
```

Also, a class could mix in two or more modules and more than one implements
the same method beign searched for. In that case, the most-recently mixed-in
module is searched first.

## How `prepend` and `extend` work

Eventhough `include` is the most common way of mixing in modules into a
class, Ruby provides two other ways to achive that but with some differences.

**`prepend`**

If you `prepend` a module to a class, the object looks in that module first,
before it looks in the class. it basically inserts the module at the begining of
the ancestors chain.

You can see the difference between `include` and `prepend` reflected when calling
`ancestors` on the `Person` class, which lists all the classes and modules where
an instance of the class will search for methods.

```ruby
module LookHereFirst; end
module LookHereSecond; end

class Person
  prepend LookHereFirst
  include LookHereSecond
end

Person.ancestors

# => [LookHereFirst, Person, LookHereSecond, Object, PP::ObjectMixin, Kernel, BasicObject]
```

You could use `prepend` when you want methods in a module to take precedence over
the versions defined in a given class.

**`extend`**

On the other hand, `extend` is another way of mixing a module into a class. The
difference is that the module's methods will be available as class methods instead
of instance methods.

Had we used `extend` rather than `prepend` in our example above, the `LookHereFirst`
module would not have been inserted into `Person`'s ancestors chain. Instead, Ruby
inserts the module in the ancestors chain of `Person`'s *singlenton class*.

## Where do singlenton methods fit in the method-lookup path?

An object's singleton methods live in the object *singleton class* and so an object
can call instance methods from its class and from its singleton class. It has both.

To solve a message into a method, an object looks in all the instance methods
defined in these two classes, along with methods available through ancestral classes
or through any modules that have been mixed in or prepended to any of these classes.

Let's `prepend` and `include` two modules in an object's singleton class and then
update our diagram to see singleton classes taken into account in the method-lookup
path.

> The `class << object` notation is a common way of opening an object's singleton class.

```ruby
module M
  def hello
    puts "Hello from module M"
  end
end

module X
  def hello
    puts "Hello from module X in object's singleton class"
  end
end

module N; end
module Y; end

class C; end

class D < C
  prepend M
  include N
end

object = D.new

class << object
  prepend X
  include Y
end

object.hello

# Hello from module X in object's singleton class
# => nil
```

In its search for the method `hello`, `object` looks first for any module prepended
to its singleton class; then it looks in the singleton class itself. It then looks
in any modules that the singleton class has included. Finally, the search proceeds
up to the object's original class, and so forth.

![Method-lookup path with singleton classes](/assets/images/method-lookup-article/method-lookup-singleton.svg)

## Class methods are singleton methods

Class methods are singleton methods defined on objects of class `Class`.
Normally, when you define a singleton method on an object, no other object
can serve as the receiver in a call to that method. But, methods defined
as singleton methods of a class object can also be called on subclasses
of that class.

![Relationship among classes in method-lookup path](/assets/images/method-lookup-article/singleton-class-relation.svg)

The diagram above shows the relationship among classes and their singleton classes.

In summary, now we better understand how an object looks for a method when
resolving a message.

I really hope you find this article useful and thank you for taking the time.
