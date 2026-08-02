# 07 · Metaprogramming Basics

Metaprogramming is writing code that writes (or modifies) code at runtime.
Ruby is exceptionally good at this — classes stay open, methods can be
defined dynamically, and objects can be inspected and reshaped while the
program is running. This is how gems like ActiveRecord generate accessor
methods for database columns nobody hand-wrote. Used well, it eliminates
repetitive boilerplate; used carelessly, it produces code that's nearly
impossible to trace. This lesson covers the three core tools:
`define_method`, `method_missing`, and `respond_to_missing?`.

## define_method — defining methods programmatically

`define_method` defines a real, ordinary instance method, but lets you
compute its name and body in a loop instead of writing `def` by hand for
each one:

```ruby
class Product
  ATTRIBUTES = [:name, :price, :sku]

  def initialize(data)
    @data = data
  end

  ATTRIBUTES.each do |attribute|
    define_method(attribute) do
      @data[attribute]
    end
  end
end

product = Product.new(name: "Widget", price: 9.99, sku: "W-100")
puts product.name    # Widget
puts product.price    # 9.99
```

This is exactly what `attr_reader` does internally — it's `define_method`
under the hood. Writing your own version like this is useful whenever the
attribute list itself is dynamic (loaded from a config file or a database
schema) rather than known when you write the class.

## define_method vs method_missing — prefer define_method

Both can create "virtual" methods, but they behave very differently:

```ruby
# method_missing: intercepts every unknown call, every time, forever
class LazyRecord
  def initialize(data)
    @data = data
  end

  def method_missing(name, *)
    @data.key?(name) ? @data[name] : super
  end

  def respond_to_missing?(name, include_private = false)
    @data.key?(name) || super
  end
end

# define_method: pays the cost once, up front, then behaves like a real method
class EagerRecord
  def initialize(data)
    @data = data
    data.each_key { |key| self.class.define_method(key) { @data[key] } }
  end
end
```

`respond_to?`, `.methods`, and tools like `method(:name)` all work
correctly and automatically with `define_method` — because it creates a
genuine method. With `method_missing`, you must maintain
`respond_to_missing?` by hand to keep those tools honest (see
[OOP Deep Dive](01-oop-deep-dive.md) for the full footgun list). **Rule of
thumb: reach for `define_method` whenever you know the possible method
names ahead of time; reserve `method_missing` for when the names are
genuinely open-ended** (like a schemaless data wrapper that must accept
*any* key).

## instance_variable_get / instance_variable_set

Ruby lets you read and write instance variables by name, even ones the
class didn't explicitly expose:

```ruby
class Point
  def initialize(x, y)
    @x = x
    @y = y
  end
end

point = Point.new(3, 4)
puts point.instance_variable_get(:@x)   # 3
point.instance_variable_set(:@x, 10)
puts point.instance_variable_get(:@x)   # 10
puts point.instance_variables.inspect     # [:@x, :@y]
```

This is how generic serialization libraries (converting any object to a
hash) work without each class needing to implement a `to_h` method itself
— though bypassing a class's own accessors like this skips any validation
those accessors might do, so use it sparingly, mostly for tooling.

## send — calling a method by name (including private ones)

```ruby
class Account
  def initialize(balance)
    @balance = balance
  end

  private

  def apply_interest(rate)
    @balance *= (1 + rate)
  end
end

account = Account.new(100)
account.send(:apply_interest, 0.05)   # bypasses the `private` keyword
puts account.instance_variable_get(:@balance)   # 105.0
```

`send` calls a method by its symbol/string name and can call private
methods, which makes it powerful for tooling and testing but dangerous in
application code — it silently breaks encapsulation. Prefer `public_send`
when the method name comes from outside your program (user input,
config), since it respects `private`/`protected` and raises `NoMethodError`
instead of quietly executing code that was meant to be internal-only.

## define_method with metaprogrammed method names

A common pattern: generating both a getter and a "predicate" (`?`) method
from the same data in one pass:

```ruby
class FeatureFlags
  def initialize(flags)
    flags.each do |name, enabled|
      define_singleton_method("#{name}?") { enabled }
    end
  end
end

flags = FeatureFlags.new(dark_mode: true, beta_ui: false)
puts flags.dark_mode?   # true
puts flags.beta_ui?       # false
```

`define_singleton_method` defines the method on this **one object only**,
not on the whole class — useful when different instances legitimately need
different dynamic methods.

## class_eval and instance_eval — evaluating code in another context

`class_eval` runs a block as if it were written inside the class body,
letting you reopen and modify a class from outside its definition:

```ruby
String.class_eval do
  def blank?
    strip.empty?
  end
end

puts "   ".blank?   # true
puts "hi".blank?     # false
```

This is (again) monkey-patching — see the risks discussed in
[OOP Deep Dive](01-oop-deep-dive.md). `class_eval` is how gems like Rails
add methods to core classes; it's the same mechanism as reopening the
class with `class String ... end`, just usable when you only have a
reference to the class object, not its literal name.

## The core metaprogramming footgun: debuggability

Every technique on this page trades a little bit of stack-trace clarity
for less boilerplate. A `NoMethodError` on a `define_method`-generated
method points at the `define_method` call, not a named `def` line; a bug
inside `method_missing` can be genuinely confusing to track down because
the method "doesn't exist" anywhere you can `grep` for it. Use these tools
when the boilerplate they eliminate is large and repetitive — not as a
default way to write every class.

## Cheat sheet

| Tool | What it does |
|---|---|
| `define_method(name) { ... }` | defines a real method dynamically |
| `method_missing` | intercepts calls to undefined methods |
| `respond_to_missing?` | keeps `respond_to?` honest alongside `method_missing` |
| `instance_variable_get/set` | read/write an ivar by name |
| `send` / `public_send` | call a method by name; `send` bypasses `private` |
| `define_singleton_method` | defines a method on one specific object |
| `class_eval` | runs a block as if inside the class body (reopens it) |

## Exercise

Write a class `Struct2` (don't worry about the name clash with Ruby's real
`Struct`) whose `initialize(attributes)` takes a hash and uses
`define_method` to create a getter for every key, plus a `to_h` method that
returns `@attributes` unchanged. Then add a class method
`Struct2.with_validations(*required_fields)` that raises `ArgumentError` in
`initialize` if any required field is missing from the hash.
