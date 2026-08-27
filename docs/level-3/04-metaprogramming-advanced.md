# 04 · Metaprogramming Advanced

Level 1/2 metaprogramming touched `method_missing` and `send` in passing.
This module goes deeper into the tools Ruby gives you to write code that
writes code: reopening classes at runtime (`class_eval`), running a block
in the context of a specific object (`instance_eval`), and generating
methods programmatically (`define_method`). These are the mechanisms
behind DSLs like RSpec's `describe`/`it` and Rails' `has_many`.

## class_eval — reopening a class dynamically

Ruby classes are never "closed" — you can reopen `Widget` and add methods
to it at any point, including from a variable holding the class itself
(useful when the class name is computed, not a literal):

```ruby
class Widget
end

Widget.class_eval do
  def greet
    "Hi from #{self.class}"
  end
end

puts Widget.new.greet
```

```text
Hi from Widget
```

`class_eval` runs the block as if it were written directly inside the
class body — `def` inside it defines an instance method exactly like a
normal `class Widget ... end` block would. This is how gems patch or
extend classes they don't own (monkey-patching), and how ORMs like
ActiveRecord inject association methods into your model classes at load
time.

## method_missing — intercepting unknown calls

Every object has `method_missing`, called whenever a method lookup
fails. Overriding it lets an object respond to method names it never
explicitly defined:

```ruby
class Config
  def initialize
    @settings = {}
  end

  def method_missing(name, *args)
    name_str = name.to_s
    if name_str.end_with?("=")
      @settings[name_str.chomp("=").to_sym] = args.first
    elsif @settings.key?(name)
      @settings[name]
    else
      super
    end
  end

  def respond_to_missing?(name, include_private = false)
    true
  end
end

c = Config.new
c.timeout = 30
puts c.timeout
```

```text
30
```

`c.timeout = 30` doesn't call a real `timeout=` method — Ruby fails the
lookup, then calls `method_missing(:timeout=, 30)`, which stores the
value in `@settings`. Reading `c.timeout` triggers the same path in
reverse.

## define_method — generating methods from data

`method_missing` intercepts every failed call at runtime, which is
flexible but slower and harder to introspect. When you know the method
names ahead of time (even if only as a list computed at class-definition
time), `define_method` is the better tool — it creates a real, normal
method:

```ruby
class Person
  [:name, :age].each do |attr|
    define_method(attr) { instance_variable_get("@#{attr}") }
    define_method("#{attr}=") { |val| instance_variable_set("@#{attr}", val) }
  end
end

p = Person.new
p.name = "Ada"
p.age = 36
puts "#{p.name}, #{p.age}"
```

```text
Ada, 36
```

This is exactly what `attr_accessor` does under the hood, and it's the
pattern used to generate the association methods (`has_many :books`
producing a real `books` method) that `class_eval` sets up on your
behalf in a real ORM.

## instance_eval — running a block against one object

`instance_eval` changes `self` for the duration of a block to be the
receiver, giving the block direct access to that object's instance
variables and private methods — without needing accessor methods:

```ruby
obj = Object.new
obj.instance_eval do
  @secret = 42
end
puts obj.instance_variable_get(:@secret)
```

```text
42
```

This is the mechanism behind builder-style DSLs:

```ruby
class HTMLBuilder
  def initialize(&block)
    @output = +""
    instance_eval(&block)
  end

  def tag(name, text)
    @output << "<#{name}>#{text}</#{name}>"
  end

  def to_s = @output
end

page = HTMLBuilder.new do
  tag "h1", "Title"
  tag "p", "Body text"
end
puts page.to_s
# <h1>Title</h1><p>Body text</p>
```

Inside the block, `tag` resolves against `HTMLBuilder`'s instance
(because `self` was reassigned there), not against wherever the block was
originally written — which is exactly what lets you write `tag "h1",
"Title"` instead of `builder.tag("h1", "Title")`.

## class_eval vs instance_eval vs define_method — which one

- **`class_eval`**: add/redefine *instance methods* on a class you have a
  reference to, especially one whose name you don't know until runtime.
- **`instance_eval`**: run a block with `self` pointed at one specific
  object — the core trick behind fluent/builder DSLs.
- **`define_method`**: generate methods programmatically from a known
  list of names, as a faster, more introspectable alternative to
  `method_missing`.

## Metaprogramming-specific traps

- **Forgetting `respond_to_missing?`** alongside `method_missing` breaks
  `respond_to?`, `.methods`, and duck-typing checks — code that asks
  "does this object handle `foo`?" gets a wrong `false` even though
  `method_missing` would happily handle it.
- **`method_missing` without eventually calling `super`** for truly
  unhandled names swallows real `NoMethodError`s, turning typos into
  silent `nil`s or confusing custom errors instead of Ruby's normal,
  greppable exception.
- **`define_method` block scope captures the enclosing scope** (a
  closure) — this is a feature for the accessor pattern above, but it
  means a loop variable reused across iterations can leak into every
  generated method if you're not careful about what the block closes
  over.
- **`class_eval` on a class you don't own is monkey-patching** — it can
  silently override a method the gem author relies on internally,
  breaking in a way that's very hard to trace back to "line 40 of my
  initializer." Prefer refinements or composition when you don't have to
  reach into someone else's class.
- **`instance_eval` changes `self` but not local-variable scope** — a
  block passed to `instance_eval` still closes over locals from where it
  was *written*, so `tag` methods resolve against the new `self`, but a
  plain local variable referenced inside still means whatever it meant
  outside.

## Cheat sheet

| Task | Code |
|---|---|
| Reopen a class and add a method | `SomeClass.class_eval { def m; end }` |
| Run a block with a different `self` | `obj.instance_eval { ... }` |
| Intercept unknown method calls | `def method_missing(name, *args); ...; end` |
| Keep `respond_to?` honest | `def respond_to_missing?(name, priv); ...; end` |
| Generate a method from a name | `define_method(:name) { ... }` |
| Generate a method with args | `define_method(:name) { \|x\| ... }` |
| Call a private method externally | `obj.send(:private_method)` |
| Check defined methods | `SomeClass.instance_methods(false)` |

## Exercise

Build a small `AttributeBuilder` module:

1. A module `Attributable` with a class method `attribute(name, default:
   nil)` that, when `extend`ed into a class, uses `define_method` to
   create a getter (returning the stored value or `default`) and setter
   for that attribute name.
2. A class `Task` that `extend Attributable` and declares `attribute
   :title` and `attribute :priority, default: "normal"`.
3. Demonstrate: creating a `Task.new`, reading `.priority` before setting
   it (should print `"normal"`), then setting and reading `.title`.
4. Add a `method_missing`-based fallback on `Task` that raises a custom
   `NoSuchAttributeError` for any unknown attribute-style call, and show
   it triggering.

Run the script and confirm the printed values match what `define_method`
generated.
