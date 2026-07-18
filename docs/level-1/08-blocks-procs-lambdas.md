# 08 · Blocks, Procs & Lambdas Introduction

Blocks are one of Ruby's defining features — a chunk of code you can pass to
a method, which the method can invoke (`yield`) zero or more times. Nearly
every Ruby iteration idiom (`each`, `map`, `select`) is built on top of them.

## Blocks with do...end or { }

```ruby
[1, 2, 3].each do |n|
  puts n * 2
end
# 2 4 6

[1, 2, 3].each { |n| puts n * 2 }   # same thing, single-line style
```

Convention: `{ }` for short, single-line blocks; `do...end` for multi-line
blocks. Both are functionally identical.

## yield — a method invoking its own block

```ruby
def three_times
  yield
  yield
  yield
end

three_times { puts "Hi!" }
# Hi!
# Hi!
# Hi!
```

```ruby
def apply_to_five
  yield(5)   # pass an argument to the block
end

result = apply_to_five { |n| n * n }
puts result   # 25
```

## block_given? — making a block optional

```ruby
def greet(name)
  if block_given?
    yield(name)
  else
    puts "Hello, #{name}!"
  end
end

greet("Ada")                                # Hello, Ada!
greet("Ada") { |name| puts "Hey #{name}!" }   # Hey Ada!
```

## Procs — a block turned into an object

```ruby
square = Proc.new { |n| n * n }
puts square.call(5)   # 25
puts square.(5)         # 25 -- shorthand for .call
puts square[5]           # 25 -- another shorthand

# Passing a block explicitly with &
def apply(value, &block)
  block.call(value)
end

puts apply(5, &square)   # 25
```

## Lambdas — a stricter kind of Proc

```ruby
multiply = ->(a, b) { a * b }   # "stabby lambda" syntax
puts multiply.call(3, 4)         # 12
puts multiply.(3, 4)              # 12

add = lambda { |a, b| a + b }     # equivalent, more explicit syntax
puts add.call(2, 3)                # 5
```

## Proc vs Lambda — the two key differences

```ruby
def proc_return
  my_proc = Proc.new { return 10 }   # `return` inside a Proc exits the ENCLOSING method
  my_proc.call
  20   # never reached if the proc runs
end

def lambda_return
  my_lambda = -> { return 10 }        # `return` inside a lambda exits only the lambda
  my_lambda.call
  20   # DOES run -- execution continues normally
end

puts proc_return     # 10
puts lambda_return    # 20
```

```ruby
strict_lambda = ->(a, b) { a + b }
loose_proc = Proc.new { |a, b| [a, b] }

# strict_lambda.call(1)          # ArgumentError -- lambdas check argument count strictly
puts loose_proc.call(1).inspect   # [1, nil] -- procs silently fill missing args with nil
```

| | Proc | Lambda |
|---|------|--------|
| Argument count checking | Lenient (missing args become `nil`) | Strict (raises `ArgumentError`) |
| `return` behavior | Returns from the enclosing method | Returns from just the lambda |

## Blocks vs Procs vs Lambdas — when to use which

- **Block**: the default for one-off iteration (`each`, `map`) — no need to
  name or reuse it.
- **Proc**: when you need to store a block as a variable or pass it around,
  and don't need strict argument checking.
- **Lambda**: when you want Proc-like reusability *and* method-like
  behavior (strict arity, `return` scoped to itself) — the safer default
  when in doubt.

## Cheat sheet

| Feature | Syntax |
|---------|--------|
| Block (multi-line) | `method do \|arg\| ... end` |
| Block (single-line) | `method { \|arg\| ... }` |
| Invoke a passed block | `yield(args)` |
| Check for a block | `block_given?` |
| Named parameter for a block | `def m(&block)` |
| Create a Proc | `Proc.new { \|args\| ... }` |
| Create a lambda | `->(args) { ... }` or `lambda { \|args\| ... }` |
| Call a Proc/lambda | `.call(args)`, `.(args)`, or `[args]` |

## Exercise

Write a method `repeat(n)` that yields to its block `n` times, passing the
current iteration index (0-based) each time. Then write a lambda
`is_positive` that checks whether a number is greater than 0, and use it with
`Array#select` to filter a mixed array of positive and negative numbers.
