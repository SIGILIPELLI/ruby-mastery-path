# 05 · Arrays & Hashes

## Arrays — ordered, mutable collections

```ruby
fruits = ["apple", "banana", "cherry"]

fruits << "date"           # append (shovel operator)
fruits.push("elderberry")   # also appends
fruits.unshift("avocado")   # prepend

puts fruits[0]     # avocado
puts fruits[-1]    # elderberry -- negative indices count from the end
puts fruits[1..3]  # ["apple", "banana", "cherry"] -- inclusive range slice
puts fruits.length # 6

fruits.delete("banana")   # removes by value
```

## Common array methods

```ruby
numbers = [5, 3, 8, 1, 9]

puts numbers.sort            # [1, 3, 5, 8, 9] -- returns a NEW sorted array
puts numbers.sort!             # sorts numbers IN PLACE (bang version)
puts numbers.map { |n| n * 2 }   # [2, 6, 10, 16, 18]
puts numbers.select { |n| n.even? }   # [8] -- keeps elements where block is true
puts numbers.reject { |n| n.even? }   # [1, 3, 5, 9] -- opposite of select
puts numbers.reduce(:+)          # 26 -- sum via reduce/inject with a symbol
puts numbers.reduce(0) { |sum, n| sum + n }   # 26 -- same, with a block
puts numbers.include?(8)         # true
puts numbers.find { |n| n > 5 }   # 8 -- first element matching the block
```

## Hashes — key/value pairs

```ruby
person = { name: "Ada", age: 30 }   # symbol keys, modern idiomatic syntax

person[:email] = "ada@example.com"   # add/update a key
puts person[:age]                     # 30
puts person.fetch(:age, 0)           # 30 -- safe lookup with a default
puts person.fetch(:missing, 0)       # 0

person.delete(:age)

person.each do |key, value|
  puts "#{key}: #{value}"
end
```

```ruby
# Older string-key / "hash rocket" syntax, still valid and needed for
# non-symbol keys:
prices = { "apple" => 1.50, "banana" => 0.75 }
puts prices["apple"]   # 1.5
```

## Transforming hashes

```ruby
prices = { apple: 1.50, banana: 0.75, cherry: 3.00 }

expensive = prices.select { |name, price| price > 1.0 }
puts expensive   # {apple: 1.5, cherry: 3.0}

doubled = prices.transform_values { |price| price * 2 }
puts doubled      # {apple: 3.0, banana: 1.5, cherry: 6.0}

names = prices.keys      # [:apple, :banana, :cherry]
values = prices.values   # [1.5, 0.75, 3.0]
```

## Ranges

```ruby
range = (1..5)          # inclusive: 1, 2, 3, 4, 5
exclusive = (1...5)     # exclusive: 1, 2, 3, 4

puts range.to_a          # [1, 2, 3, 4, 5]
puts range.include?(3)   # true
puts (1..5).sum          # 15
```

## Choosing the right structure

| Need | Use |
|------|-----|
| Ordered, allow duplicates, changeable | `Array` |
| Fast lookup by named key | `Hash` |
| A sequence of numbers/letters | `Range` |
| Unique items only | `Array#uniq` or convert to a `Set` (from the `set` library) |

## Cheat sheet

| Task | Method |
|------|--------|
| Append | `arr << x` or `arr.push(x)` |
| Transform each element | `arr.map { \|x\| ... }` |
| Keep matching elements | `arr.select { \|x\| ... }` |
| Remove matching elements | `arr.reject { \|x\| ... }` |
| Sum/combine | `arr.reduce(:+)` or `arr.sum` |
| First match | `arr.find { \|x\| ... }` |
| Safe hash lookup | `hash.fetch(:key, default)` |
| Iterate a hash | `hash.each { \|k, v\| ... }` |

## Exercise

Given an array of words, use `select` to keep only words longer than 4
characters, `map` to convert them to uppercase, and `reduce` to join them all
into one comma-separated string. Then, given a hash of `{name: "Ada", role:
"Engineer", team: "Platform"}`, use `transform_values` to uppercase every
value in the hash.
