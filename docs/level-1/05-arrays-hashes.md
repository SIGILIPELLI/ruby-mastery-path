---
description: "Arrays & Hashes — A Ruby Array is a contiguous, growable buffer of object references (like a Vec in the C source), not a linked list — that's why arr[5]…"
---

# 05 · Arrays & Hashes

## 🎥 Video walkthrough

<iframe width="100%" height="400" style="max-width:720px;aspect-ratio:16/9;height:auto;" src="https://www.youtube.com/embed/ZuzQfrLNh5E" title="Video walkthrough" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

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

## How It Actually Works

A Ruby `Array` is a contiguous, growable buffer of object references (like
a `Vec<VALUE>` in the C source), not a linked list — that's why `arr[5]` is
O(1) but `arr.unshift(x)` is O(n) (everything has to shift right). Ruby
over-allocates capacity on growth (similar to how `ArrayList`/`Vec` grow in
other languages), so repeated `push` calls are amortized O(1) even though
occasional reallocations happen. A `Hash` is a genuine hash table: MRI calls
`#hash` and `#eql?` on each key to place it in a bucket, and since Ruby 1.9
hashes also maintain a doubly-linked **insertion order** list alongside the
buckets — which is why iterating a Hash always yields keys in the order
they were first inserted, a guarantee (not an implementation accident) since
Ruby 1.9. Using a mutable object like an `Array` as a hash key is dangerous
precisely because its `#hash` value can change after insertion, breaking
the bucket it lives in — symbols and frozen strings are safe because their
hash value never changes.

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

## 🔀 See this in another language

- [PHP — Arrays](https://sigilipelli.github.io/php-mastery-path/level-1/05-arrays/)
- [MATLAB — Functions in MATLAB](https://sigilipelli.github.io/matlab-mastery-path/level-1/05-functions/)
- [JavaScript — Arrays & Objects](https://sigilipelli.github.io/javascript-mastery-path/level-1/05-arrays-objects/)

## Exercise

Given an array of words, use `select` to keep only words longer than 4
characters, `map` to convert them to uppercase, and `reduce` to join them all
into one comma-separated string. Then, given a hash of `{name: "Ada", role:
"Engineer", team: "Platform"}`, use `transform_values` to uppercase every
value in the hash.
