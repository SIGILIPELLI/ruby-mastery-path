# 02 · Blocks/Procs/Lambdas Advanced

[Level 1](../level-1/08-blocks-procs-lambdas.md) covered the basics: blocks,
`yield`, and the argument/return differences between `Proc` and `lambda`.
This lesson goes deeper into *how* blocks actually work under the hood —
closures, explicit block capture with `&`, converting between the three
callable types, and a few patterns you'll see constantly in real Ruby code.

## Closures — blocks remember their surrounding variables

A block isn't just code — it's code bundled with a reference to the local
variables that were in scope where it was created. This bundle is called a
**closure**, and it keeps working even after that surrounding method has
returned:

```ruby
def make_counter
  count = 0
  increment = lambda { count += 1 }   # captures `count` by reference
  increment
end

counter = make_counter
puts counter.call   # 1
puts counter.call   # 2
puts counter.call   # 3
```

`make_counter` already returned, yet `counter` still has access to `count`
and can keep mutating it. Each call to `make_counter` creates a *new*
`count`, so two counters never share state:

```ruby
counter_a = make_counter
counter_b = make_counter
counter_a.call
counter_a.call
puts counter_a.call   # 3
puts counter_b.call   # 1 -- independent closure, independent `count`
```

## Explicit block capture with &block

Every method can implicitly receive a block and invoke it with `yield`, but
naming it explicitly with `&block` turns it into a real `Proc` object you
can store, pass along, or call conditionally:

```ruby
def process(items, &transformer)
  items.map { |item| transformer.call(item) }
end

doubled = process([1, 2, 3]) { |n| n * 2 }
puts doubled.inspect   # [2, 4, 6]
```

The `&` also works in reverse — prefixing an existing `Proc`/`lambda`/symbol
with `&` when *calling* a method converts it back into a block:

```ruby
square = ->(n) { n * n }
puts [1, 2, 3].map(&square).inspect   # [1, 4, 9]

# The famous &:symbol idiom is the same mechanism: Symbol#to_proc
puts ["ruby", "rails"].map(&:upcase).inspect   # ["RUBY", "RAILS"]
```

`&:upcase` works because `Symbol#to_proc` turns `:upcase` into
`->(obj) { obj.upcase }` automatically — that's the entire trick behind
Ruby's shortest, most idiomatic `map` calls.

## yield vs calling a captured block — they're not quite the same cost

`yield` is faster than `&block.call` because Ruby doesn't have to allocate a
`Proc` object unless you explicitly ask for one by naming it:

```ruby
def fast_version
  yield 5          # no Proc object created
end

def flexible_version(&block)
  block.call(5)     # Proc object created up front, but block is a real value
end
```

Prefer plain `yield` unless you actually need the block as an object (to
pass it elsewhere, store it, or check `respond_to?(:call)` on it).

## Multiple yields and yielding multiple values

A method can `yield` more than once, and can yield more than one value at a
time — the block destructures them the same way multiple assignment does:

```ruby
def each_pair(hash)
  hash.each do |key, value|
    yield key, value
  end
end

each_pair({ a: 1, b: 2 }) do |k, v|
  puts "#{k} => #{v}"
end
# a => 1
# b => 2
```

## Curry — partially applying a lambda

`curry` transforms a multi-argument lambda into a chain of single-argument
lambdas, letting you supply arguments in stages:

```ruby
multiply = ->(a, b, c) { a * b * c }
curried = multiply.curry

double = curried[2]           # fixes a = 2, returns a new lambda
double_and_triple = double[3]   # fixes b = 3

puts double_and_triple[4]   # 24  (2 * 3 * 4)
puts curried[2][3][4]         # 24  -- same thing, all at once
```

This is handy for building specialized versions of a general function
without writing a wrapper method by hand.

## Method objects — turning a method into a callable

Every object method can be extracted as a standalone `Method` object with
`.method(:name)`, and converted to a `Proc` with `.to_proc` (or passed
directly with `&`):

```ruby
def shout(text)
  text.upcase + "!"
end

shout_method = method(:shout)
puts shout_method.call("hello")   # HELLO!

puts ["a", "b"].map(&shout_method).inspect   # ["A!", "B!"]
```

This is useful when you already have a named method and want to reuse it
as a block without redefining it as a lambda.

## Proc.new vs proc vs lambda vs -> — which to reach for

```ruby
p1 = Proc.new { |x| x }   # explicit Proc -- lenient arity, `return` exits enclosing method
p2 = proc { |x| x }         # shorthand for Proc.new -- identical behavior
l1 = lambda { |x| x }       # strict arity, `return` exits only the lambda
l2 = ->(x) { x }             # stabby lambda -- identical to l1, just terser syntax
```

In modern Ruby style: use `proc`/`Proc.new` rarely (mostly for `&block`
capture from an actual block argument), and default to the stabby lambda
`->(...)` for anything you're constructing directly — its strict arity
checking catches bugs that a lenient proc would silently swallow.

## The return-from-a-proc footgun, revisited

This is worth repeating because it causes a real, hard-to-diagnose bug: a
`Proc`'s `return` unwinds the *method that created it*, not just the proc.
If that method has already finished, Ruby raises `LocalJumpError`:

```ruby
def get_proc
  Proc.new { return "done" }
end

stray_proc = get_proc
begin
  stray_proc.call   # get_proc already returned -- nothing to return FROM
rescue LocalJumpError => e
  puts "Error: #{e.message}"
end
```

A lambda created the same way is perfectly safe to call later, because its
`return` only ever exits the lambda itself — one more reason lambdas are the
safer default when a callable is going to be stored or passed around rather
than used immediately.

## Cheat sheet

| Feature | Syntax | Notes |
|---|---|---|
| Capture the block as a Proc | `def m(&block)` | costs a Proc allocation |
| Convert Proc/lambda back to a block | `method(&my_proc)` | |
| Symbol shorthand block | `&:method_name` | via `Symbol#to_proc` |
| Extract a method as a callable | `method(:name)` | returns a `Method` object |
| Partially apply a lambda | `my_lambda.curry` | |
| Closures capture by reference | shared local variables | each closure-creating call gets fresh locals |

## Exercise

Write a method `make_multiplier(factor)` that returns a lambda closing over
`factor`, so `triple = make_multiplier(3)` lets you call `triple.call(7)` to
get `21`. Then write a method `retry_with_delay(&block)` that calls the
given block, and if it raises, waits (just `puts "retrying..."` instead of
an actual `sleep`) and calls it exactly once more before giving up — you'll
build the exception-handling half of this properly in
[Exception Handling](03-exception-handling.md).
