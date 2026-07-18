# 04 · Methods

## Defining methods

```ruby
def add(a, b)
  a + b   # the last evaluated expression is returned automatically
end

def greet(name = "friend")   # default argument
  "Hello, #{name}!"
end

puts add(2, 3)     # 5
puts greet          # Hello, friend!
puts greet("Ada")    # Hello, Ada!
```

Ruby methods return the value of their **last evaluated expression** by
default — an explicit `return` is only needed to exit early.

## Explicit return

```ruby
def safe_divide(a, b)
  return "Cannot divide by zero" if b == 0   # early return, guard clause
  a / b
end

puts safe_divide(10, 2)   # 5
puts safe_divide(10, 0)   # Cannot divide by zero
```

## The question mark and bang naming conventions

```ruby
def even?(n)
  n % 2 == 0
end

puts even?(4)   # true
puts even?(5)   # false
```

Methods ending in `?` are a strong convention (not enforced by the language)
for methods returning `true`/`false`. Methods ending in `!` conventionally
indicate a "dangerous" variant — usually one that mutates the receiver in
place or could raise, as opposed to a safer non-bang counterpart:

```ruby
name = "ada"
puts name.upcase    # "ADA" -- returns a NEW string, `name` itself is unchanged
puts name            # "ada" -- still lowercase

name.upcase!         # mutates `name` itself, in place
puts name            # "ADA"
```

## Splat args (*args) and keyword args

```ruby
def total(*numbers)     # collects any number of positional args into an array
  numbers.sum
end

puts total(1, 2, 3, 4)   # 10

def make_profile(name:, city: "Unknown")   # keyword args, `city` has a default
  "#{name} from #{city}"
end

puts make_profile(name: "Ada", city: "London")   # Ada from London
puts make_profile(name: "Grace")                  # Grace from Unknown
```

Keyword arguments must be passed by name (`make_profile(name: "Ada")`), which
makes call sites self-documenting compared to positional arguments — very
common in idiomatic Ruby method signatures with more than 1-2 parameters.

## Double splat (**kwargs) — collecting arbitrary keyword args

```ruby
def describe(**options)
  options.each { |key, value| puts "#{key}: #{value}" }
end

describe(name: "Ada", role: "Engineer")
# name: Ada
# role: Engineer
```

## Methods are called on objects (even top-level ones, implicitly on `self`)

```ruby
def square(x)
  x * x
end

puts square(5)   # 25 -- implicitly called on the top-level `self`

# Methods can be referenced as objects with `method(...)`
op = method(:square)
puts op.call(4)   # 16
```

## Blocks as implicit method arguments (preview — full coverage in Module 8)

```ruby
def apply_twice
  yield(yield(5))   # `yield` calls the block passed to this method
end

result = apply_twice { |n| n * n }
puts result   # 625 (5*5=25, then 25*25=625)
```

`yield` is how a method invokes a block passed to it without needing an
explicit parameter for it — this is the foundation for `each`, `map`, and
most of Ruby's iteration idioms, covered fully in
[Module 8](08-blocks-procs-lambdas.md).

## Cheat sheet

| Feature | Syntax |
|---------|--------|
| Basic method | `def name(a, b) ... end` |
| Default argument | `def name(a, b = 10) ... end` |
| Keyword argument | `def name(a:, b: 10) ... end` |
| Splat (variadic) | `def name(*args) ... end` |
| Double splat | `def name(**kwargs) ... end` |
| Implicit return | Last expression's value, or explicit `return` |
| Predicate convention | `name?` returns true/false |
| Dangerous/mutating convention | `name!` |

## Exercise

Write a method `summarize(*values, precision: 2)` that returns a hash with
`:min`, `:max`, and `:average` (rounded to `precision` decimal places) of the
given values. Then write a method `shout!(s)` that mutates a string in place
to be uppercase with an exclamation point appended (hint: use `upcase!` and
`concat`, or reassign via `replace`).
