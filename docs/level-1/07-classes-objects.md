---
description: "Classes & Objects Basics — initialize is the method Ruby calls automatically when you write Person.new(...) — it's the constructor, but it's just a…"
---

# 07 · Classes & Objects Basics

## Defining a class

```ruby
class Person
  def initialize(name, age)   # the constructor
    @name = name                # @ prefix = instance variable
    @age = age
  end

  def greet
    "Hi, I'm #{@name} and I'm #{@age} years old."
  end
end

ada = Person.new("Ada", 30)
puts ada.greet   # Hi, I'm Ada and I'm 30 years old.
```

`initialize` is the method Ruby calls automatically when you write
`Person.new(...)` — it's the constructor, but it's just a regular method
under the hood.

## Instance variables need explicit accessor methods

```ruby
class Person
  def initialize(name, age)
    @name = name
    @age = age
  end

  def name
    @name
  end

  def age=(new_age)   # a "setter" method -- Ruby lets you call it as ada.age = 31
    @age = new_age
  end
end

ada = Person.new("Ada", 30)
puts ada.name    # Ada
# ada.age        # error: private method `age' called -- no getter defined for age!
ada.age = 31       # this works -- age=(new_age) was defined
```

Instance variables (`@name`) are **not** accessible from outside the class
unless you define methods that expose them — there's no automatic public
field access like in Python or JavaScript.

## attr_accessor, attr_reader, attr_writer

```ruby
class Person
  attr_accessor :name    # generates both a getter AND setter for :name
  attr_reader :age         # generates only a getter for :age

  def initialize(name, age)
    @name = name
    @age = age
  end
end

ada = Person.new("Ada", 30)
puts ada.name     # Ada       -- getter, from attr_accessor
ada.name = "Ada L." # works    -- setter, from attr_accessor
puts ada.age       # 30        -- getter, from attr_reader
# ada.age = 31     # error: no setter -- attr_reader only generates a getter
```

`attr_accessor`, `attr_reader`, and `attr_writer` are the idiomatic shortcut
for the boilerplate getter/setter methods shown above — almost every Ruby
class uses one of these instead of writing them by hand.

## Class methods vs instance methods

```ruby
class Person
  attr_reader :name

  def initialize(name)
    @name = name
  end

  # `self.` prefix makes this a CLASS method, called as Person.count_greeting
  def self.default
    new("Anonymous")
  end

  def greet
    "Hi, I'm #{@name}"
  end
end

anon = Person.default
puts anon.greet   # Hi, I'm Anonymous
```

## Inheritance

```ruby
class Animal
  def initialize(name)
    @name = name
  end

  def speak
    "..."
  end
end

class Dog < Animal
  def speak
    "#{@name} says Woof!"
  end
end

class Cat < Animal
  def speak
    "#{@name} says Meow!"
  end
end

[Dog.new("Rex"), Cat.new("Whiskers")].each do |animal|
  puts animal.speak
end
# Rex says Woof!
# Whiskers says Meow!
```

## super — calling the parent's version

```ruby
class Animal
  def initialize(name)
    @name = name
  end
end

class Dog < Animal
  def initialize(name, breed)
    super(name)   # calls Animal#initialize with `name`
    @breed = breed
  end

  def describe
    "#{@name} is a #{@breed}"
  end
end

rex = Dog.new("Rex", "Labrador")
puts rex.describe   # Rex is a Labrador
```

## How It Actually Works

Every object in Ruby carries an invisible pointer to its class, and every
class is itself an object — an instance of `Class`, which is itself an
instance of `Class`. When you write `class Dog; end`, MRI allocates a
`Class` object and stores a method table (a hash from method name to
bytecode/C-function pointer) directly on it. Calling `dog.bark` doesn't
search `dog` itself for a `bark` method — instance variables and methods
live in different places: `@name` is stored in a slot table on the
*instance*, while `bark`'s implementation lives in the *class's* method
table, found via the ancestor-chain walk described elsewhere in this site.
`attr_accessor :name` is just Ruby code that defines two methods
(`name` and `name=`) on the class at load time using `define_method`
internally — there's no special compiler magic, you could write those two
`def`s by hand and get identical behavior.

## Cheat sheet

| Feature | Syntax |
|---------|--------|
| Define a class | `class Name ... end` |
| Constructor | `def initialize(args) ... end` |
| Instance variable | `@name` |
| Getter + setter | `attr_accessor :name` |
| Getter only | `attr_reader :name` |
| Class method | `def self.method_name ... end` |
| Inheritance | `class Sub < Base ... end` |
| Call parent method | `super(args)` |

## 🔀 See this in another language

- [PHP — Classes & Objects Basics](https://sigilipelli.github.io/php-mastery-path/level-1/07-classes-objects/)
- [MATLAB — Working with Data Files](https://sigilipelli.github.io/matlab-mastery-path/level-1/07-data-files/)
- [JavaScript — DOM Basics & Events](https://sigilipelli.github.io/javascript-mastery-path/level-1/07-dom-events/)

## Exercise

Write a class `BankAccount` with `attr_reader :owner, :balance`, an
`initialize(owner, balance)`, and methods `deposit(amount)` and
`withdraw(amount)` (the latter should return `"Insufficient funds"` as a
string instead of actually withdrawing if the amount exceeds the balance).
Then write a subclass `SavingsAccount < BankAccount` that adds an
`interest_rate` and a method `apply_interest` that increases the balance by
that rate.
