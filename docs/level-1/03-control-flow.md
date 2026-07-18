# 03 · Control Flow

## if / elsif / else

```ruby
score = 82

if score >= 90
  grade = "A"
elsif score >= 80
  grade = "B"
elsif score >= 70
  grade = "C"
else
  grade = "F"
end

puts grade   # B
```

Notice `if`/`elsif`/`else` blocks are closed with `end`, not curly braces —
this is true of every block construct in Ruby (methods, loops, classes).

## if as an expression

```ruby
score = 82

# if returns the value of whichever branch ran -- can assign directly
grade = if score >= 90
          "A"
        elsif score >= 80
          "B"
        else
          "F"
        end

puts grade   # B
```

## unless — the inverse of if

```ruby
age = 15

unless age >= 18
  puts "Not an adult yet"
end
# Not an adult yet

# Often used as a trailing modifier for short conditions:
puts "Not an adult yet" unless age >= 18
```

## Trailing (modifier) conditionals

```ruby
puts "positive" if 5 > 0
puts "empty" if [].empty?
```

Any single-line statement can be followed by `if condition` or `unless
condition` — extremely common in idiomatic Ruby for short guard clauses.

## case/when — Ruby's multi-branch match

```ruby
day = "Wed"

case day
when "Mon", "Tue", "Wed", "Thu", "Fri"
  puts "Weekday"
when "Sat", "Sun"
  puts "Weekend"
else
  puts "Not a valid day"
end
# Weekday
```

```ruby
score = 82

grade = case score
        when 90..100 then "A"
        when 80...90 then "B"
        when 70...80 then "C"
        else "F"
        end

puts grade   # B
```

`case/when` uses `===` under the hood (the "case equality" operator), which
is why ranges like `80...90` work directly as `when` conditions — `Range#===`
checks whether the value falls inside the range.

## while and until

```ruby
count = 0
while count < 3
  puts "count is #{count}"
  count += 1
end
# count is 0 / count is 1 / count is 2

count = 0
until count >= 3
  puts "count is #{count}"
  count += 1
end
# same output -- until is just "while not"
```

## for and each — each is the idiomatic choice

```ruby
# for loop (works, but rarely used in idiomatic Ruby)
for i in 1..3
  puts i
end

# each -- the idiomatic way to iterate in Ruby
(1..3).each do |i|
  puts i
end
```

`for` leaks its loop variable into the surrounding scope and is considered
un-idiomatic; `each` with a block is what virtually all Ruby code uses
instead — you'll see it constantly once we cover blocks in Module 8.

## break, next, redo

```ruby
(1..10).each do |n|
  break if n == 4      # exits the loop entirely
  puts n
end
# 1 2 3

(1..6).each do |n|
  next if n.even?       # skips to the next iteration
  puts n
end
# 1 3 5
```

## Cheat sheet

| Construct | Use when |
|-----------|----------|
| `if / elsif / else` | General branching |
| `unless` | Branching on a negative condition, reads more naturally |
| Trailing `if`/`unless` | Short single-line guard clauses |
| `case / when` | Comparing one value against several possibilities/ranges |
| `while` / `until` | Loop while/until a condition holds |
| `each` | Idiomatic iteration (preferred over `for`) |
| `break` | Exit the loop entirely |
| `next` | Skip to the next iteration |

## Exercise

Write a script that prints FizzBuzz for numbers 1–30 using `each` and
`case/when`: multiples of 3 print `"Fizz"`, multiples of 5 print `"Buzz"`,
multiples of both print `"FizzBuzz"`, otherwise the number itself.
