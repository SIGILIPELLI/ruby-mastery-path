# 02 · Variables & Types

## Variables

Ruby variables need no type declaration — they're just names bound to
objects, and the binding can change to point at a different type entirely.

```ruby
age = 25
name = "Ada"
height = 1.68
is_student = false

puts age.class    # Integer
```

## Core built-in types

```ruby
whole_number = 42          # Integer
pi_ish = 3.14159             # Float
message = "hi there"         # String
flag = true                  # TrueClass (and `false` is FalseClass)
nothing = nil                 # NilClass

puts whole_number.class   # Integer
```

## Numeric operators

```ruby
a = 7
b = 2

puts a + b     # 9
puts a - b     # 5
puts a * b     # 14
puts a / b     # 3    -- integer division truncates when both operands are Integer
puts a.to_f / b # 3.5  -- force float division by converting one operand
puts a % b     # 1    modulo
puts a ** b    # 49   exponentiation
```

Integer `/` division truncates toward zero when both operands are integers —
a common surprise coming from Python 3 or JavaScript, where `/` always
produces a float.

## Comparison and boolean operators

```ruby
puts 5 > 3            # true
puts 5 == 5.0          # true -- value equality across types
puts 5.equal?(5)       # true for small integers (they're cached/identical objects)

puts true && false     # false
puts true || false     # true
puts !true              # false
```

## Ruby's truthiness: only nil and false are falsy

```ruby
puts "truthy" if 0        # prints "truthy" -- 0 IS truthy in Ruby!
puts "truthy" if ""        # prints "truthy" -- empty string IS truthy!
puts "falsy" unless nil    # prints "falsy"
puts "falsy" unless false  # prints "falsy"
```

This differs sharply from Python, JavaScript, and PHP, where `0` and `""` are
falsy. In Ruby, **only `nil` and `false` are falsy** — everything else,
including `0`, `""`, and empty arrays/hashes, is truthy.

## Type conversion

```ruby
42.to_s          # "42"
"42".to_i        # 42
"3.5".to_f       # 3.5
3.9.to_i         # 3 -- truncates, doesn't round
0.to_s.empty?    # false
```

## Constants and naming conventions

```ruby
MAX_RETRIES = 3       # constants start with an UPPERCASE letter
user_age = 25          # variables use snake_case

# Ruby warns (but doesn't prevent) reassigning a constant:
MAX_RETRIES = 5
# warning: already initialized constant MAX_RETRIES
```

Unlike some languages, Ruby constants are a *language-level* concept
(anything starting with an uppercase letter is treated as one), not just a
naming convention — though Ruby only warns, rather than errors, if you
reassign one.

## Symbols — a lightweight, immutable identifier

```ruby
status = :active

puts status.class      # Symbol
puts :active == :active  # true -- same symbol is always the same object

# Common use: hash keys
person = { name: "Ada", role: :admin }
puts person[:role]      # admin
```

Symbols look similar to strings but are immutable and memory-efficient —
Ruby code idiomatically uses them for things like hash keys and fixed status
values rather than plain strings.

## Cheat sheet

| Concept | Example |
|---------|---------|
| Falsy values | `nil`, `false` — that's the entire list |
| Integer division | `7 / 2` → `3` (use `.to_f` for float division) |
| String conversion | `.to_s`, `.to_i`, `.to_f` |
| Symbol | `:name` — lightweight, immutable identifier |
| Constant | `UPPER_SNAKE_CASE`, warns (not errors) on reassignment |

## Exercise

Write a script that stores a rectangle's `width` and `height` as variables,
computes its area and perimeter, and prints both formatted to 2 decimal
places using `"%.2f" % value`. Then experiment in `irb` with `if 0`, `if ""`,
and `if []` to confirm all three are truthy in Ruby.
