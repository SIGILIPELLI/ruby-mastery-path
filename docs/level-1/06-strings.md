# 06 · Strings & String Methods

## String basics

```ruby
s = "Hello, World!"

puts s.downcase       # hello, world!
puts s.upcase          # HELLO, WORLD!
puts s.length          # 13
puts s[7..11]           # World  -- inclusive range slice
puts s.reverse          # !dlroW ,olleH
puts s.split(", ")      # ["Hello", "World!"]
puts ["a", "b", "c"].join(" ")   # a b c
puts "  padded  ".strip # "padded" -- removes leading/trailing whitespace
```

## String interpolation vs concatenation

```ruby
name = "Ada"
age = 30

# Interpolation -- the idiomatic Ruby way, only works in double-quoted strings
puts "#{name} is #{age} years old"

# Concatenation
puts name + " is " + age.to_s + " years old"

# << appends in place (mutates the original string)
greeting = "Hello"
greeting << ", " << name << "!"
puts greeting   # Hello, Ada!
```

`'single quotes'` do **not** interpolate — `'#{name}'` prints literally as
`#{name}`. Always use double quotes when you need `#{...}` interpolation.

## Formatting numbers into strings

```ruby
pi = 3.14159265

puts "pi rounded: %.2f" % pi          # pi rounded: 3.14
puts format("%.2f", pi)                # 3.14 -- equivalent, more explicit
puts "%5d" % 42                         # "   42" -- right-aligned in width 5

name = "Ada"
puts "Name: #{name.ljust(10)}|"          # "Name: Ada       |"
```

## Multi-line and heredoc strings

```ruby
paragraph = "This spans
multiple lines."

# Heredoc -- common for larger blocks of text
sql = <<~SQL
  SELECT *
  FROM users
  WHERE active = true
SQL

puts sql
```

`<<~` (squiggly heredoc) automatically strips the leading indentation from
each line, which keeps the heredoc readable even when nested inside
indented code.

## Common string checks

```ruby
"42".match?(/\A\d+\z/)      # true -- regex match
"hello".start_with?("he")     # true
"hello".end_with?("lo")       # true
"  ".strip.empty?              # true
"Hello, World!".include?("World")   # true
```

## String immutability (mostly)

```ruby
s = "hello"
s.upcase          # returns "HELLO" but doesn't change s
puts s             # hello -- unchanged

s.upcase!          # mutates s in place (bang version)
puts s              # HELLO
```

Ruby strings are technically mutable objects (unlike Python/Java strings),
but non-bang methods like `.upcase` return a *new* string rather than
modifying the receiver — you must call the `!` version, or reassign, to
actually change the original.

## Cheat sheet

| Task | Method |
|------|--------|
| Interpolate a value | `"text #{expr} text"` |
| Format a float | `"%.2f" % value` or `format("%.2f", value)` |
| Multi-line literal text | `<<~LABEL ... LABEL` (heredoc) |
| Check substring | `.include?`, `.start_with?`, `.end_with?` |
| Regex match | `.match?(/pattern/)` |
| Mutate in place | Bang methods: `.upcase!`, `.strip!`, etc. |

## Exercise

Write a method `slugify(title)` that converts `"Hello, World!  "` into
`"hello-world"` — lowercase, punctuation stripped (hint: `gsub` with a
regex), spaces replaced with hyphens, and any leading/trailing hyphens
stripped.
