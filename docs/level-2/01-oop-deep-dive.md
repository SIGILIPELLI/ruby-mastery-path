# 01 · OOP Deep Dive

Level 1 covered classes, `attr_accessor`, and single inheritance. Ruby only
allows a class to have **one** superclass, but real programs need to share
behavior across unrelated classes all the time — a `Dog` and a `Duck` might
both need to "swim," even though neither is a subclass of the other. Ruby
solves this with **modules**: namespaces and mixins that let you share code
without forcing an inheritance relationship.

## Modules as namespaces

A module can group related classes/constants/methods under one name, purely
to avoid naming collisions:

```ruby
module Shapes
  class Circle
    def initialize(radius)
      @radius = radius
    end

    def area
      (Math::PI * @radius**2).round(2)
    end
  end
end

circle = Shapes::Circle.new(4)
puts circle.area   # 50.27
```

`Shapes::Circle` uses `::` to reach inside the module — this is the same
namespacing Ruby itself uses for things like `Math::PI`.

## Modules as mixins — include

A module can also hold plain methods that get mixed **into** a class as
instance methods, via `include`:

```ruby
module Swimmable
  def swim
    "#{name} is swimming!"
  end
end

class Fish
  include Swimmable
  attr_reader :name

  def initialize(name)
    @name = name
  end
end

class Athlete
  include Swimmable
  attr_reader :name

  def initialize(name)
    @name = name
  end
end

puts Fish.new("Nemo").swim        # Nemo is swimming!
puts Athlete.new("Michael").swim   # Michael is swimming!
```

`Fish` and `Athlete` share zero inheritance, but both gained `swim` for
free. This is Ruby's answer to "multiple inheritance" — a class can
`include` as many modules as it wants, while only ever having one
superclass.

## extend — mixing in as class methods instead

`include` adds *instance* methods. `extend` adds the module's methods as
**class** methods (or as singleton methods on whatever object you extend):

```ruby
module Describable
  def describe
    "I am #{self}"
  end
end

class Robot
  extend Describable
end

puts Robot.describe   # I am Robot -- describe became a CLASS method
```

## Where include actually puts the module: the ancestor chain

`include` doesn't copy methods into the class — it inserts the module into
the class's **ancestor chain**, just above the class itself. Ruby method
lookup walks this chain in order, which is why a method defined directly on
the class overrides one pulled in from an included module:

```ruby
module Greetable
  def greet
    "Hi from the module"
  end
end

class Person
  include Greetable
end

puts Person.ancestors
# [Person, Greetable, Object, Kernel, BasicObject]

puts Person.new.greet   # Hi from the module
```

If `Person` defined its own `greet`, that would win, because `Person` comes
before `Greetable` in the chain. This lookup order is also why include order
matters when two modules define the same method — the *last* one `include`d
sits closer to the class and wins.

## prepend — inserting before the class

`prepend` is the less common cousin of `include`: it inserts the module
**before** the class in the ancestor chain, so the module's method runs
first and can call `super` to reach the class's own method. This is the
idiomatic way to wrap/decorate an existing method:

```ruby
module LoudLogger
  def greet
    "#{super.upcase}!!!"
  end
end

class Person
  prepend LoudLogger

  def greet
    "hello"
  end
end

puts Person.ancestors.first(2)   # [LoudLogger, Person]
puts Person.new.greet             # HELLO!!!
```

## Comparable and Enumerable are just modules

Ruby's own standard library leans hard on this pattern — `Comparable` and
`Enumerable` (covered in depth in
[Enumerable & Comparable](09-enumerable-comparable.md)) are ordinary modules
you `include`, not special language features.

## method_missing — intercepting unknown method calls

Every object responds to `method_missing` whenever a method is called that
isn't defined anywhere in its ancestor chain. By default it raises
`NoMethodError`, but you can override it to handle calls dynamically:

```ruby
class DynamicRecord
  def initialize(attributes)
    @attributes = attributes
  end

  def method_missing(name, *args)
    key = name.to_s.chomp("=").to_sym

    if name.to_s.end_with?("=")
      @attributes[key] = args.first
    elsif @attributes.key?(key)
      @attributes[key]
    else
      super   # important! see the footgun below
    end
  end

  def respond_to_missing?(name, include_private = false)
    key = name.to_s.chomp("=").to_sym
    @attributes.key?(key) || super
  end
end

record = DynamicRecord.new(title: "Refactoring", year: 1999)
puts record.title                  # Refactoring
record.year = 2018
puts record.year                    # 2018
puts record.respond_to?(:title)   # true
```

## The method_missing footguns

`method_missing` is powerful but easy to misuse:

- **Always call `super` for names you don't recognize.** Skipping the
  `super` call above would silently return `nil` for typos like
  `record.tiel` instead of raising `NoMethodError` — turning obvious bugs
  into confusing `nil` values downstream.
- **Always define `respond_to_missing?` alongside it.** Without it,
  `respond_to?` and `.methods` lie about what the object can actually do,
  which breaks duck-typing checks and tools like `method(:foo)`.
  `respond_to_missing?` is *why* `record.respond_to?(:title)` correctly
  returns `true` above.
- **It's slower** than a real method, because Ruby has to fail normal
  lookup first before falling into `method_missing`. Reach for
  `define_method` (see [Metaprogramming Basics](07-metaprogramming-basics.md))
  instead when you know the method names up front — it defines a real
  method once instead of intercepting every call forever.

## Monkey-patching: reopening classes (and why it's risky)

Ruby classes are never "closed" — you can reopen `String`, `Integer`, or any
class, including ones from the standard library, and add or redefine
methods on them:

```ruby
class String
  def shout
    upcase + "!"
  end
end

puts "watch out".shout   # WATCH OUT!
```

This is called **monkey-patching**. It's tempting because it's global and
convenient, but it's risky:

- **Global mutation** — every `String` in the entire process gets `shout`,
  including ones inside gems you didn't write. Two gems patching the same
  method differently causes silent, hard-to-debug conflicts.
- **Version fragility** — if a future Ruby version adds a real `String#shout`
  with different behavior, your patch silently shadows it.
- **Safer alternatives**: wrap the behavior in your own module and
  `include`/`extend` it only where you need it, or use `refine` (Ruby's
  scoped alternative to monkey-patching) if you truly need to change
  built-in behavior.

## Cheat sheet

| Goal | Tool |
|------|------|
| Share instance methods across unrelated classes | `module M; end` + `include M` |
| Add class-level (or singleton) methods | `extend M` |
| Wrap/override a method and still call the original | `prepend M` + `super` |
| Inspect method lookup order | `SomeClass.ancestors` |
| Handle undefined method calls dynamically | `method_missing` + `respond_to_missing?` |
| Add methods to an existing class | reopen it (monkey-patch) — use sparingly |

## Exercise

Write a module `Trackable` with a method `track_change(field, value)` that
appends `"#{field} changed to #{value}"` to an internal `@history` array
(initialize it lazily with `@history ||= []`), plus a `history` reader.
Mix `Trackable` into a `Product` class that calls `track_change` from its
`price=` setter whenever the price changes. Then write a tiny
`DynamicConfig` class using `method_missing` (with `respond_to_missing?`)
that lets you write `config.timeout = 30` and read it back with
`config.timeout`.
