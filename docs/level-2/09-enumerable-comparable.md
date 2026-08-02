# 09 · Enumerable & Comparable

Level 1 used `each`, `map`, `select`, and friends on `Array` and `Hash`
without asking where they came from. The answer: `Enumerable`, a module
that both classes `include`. This lesson shows how to get all of those
methods "for free" on your **own** classes by implementing just one method
— plus `Comparable`, `Enumerable`'s equally minimal cousin for ordering.

## Where map/select/reduce actually live

```ruby
puts Array.ancestors.include?(Enumerable)   # true
puts Hash.ancestors.include?(Enumerable)     # true
```

`Enumerable` defines dozens of methods (`map`, `select`, `reject`, `sort`,
`reduce`, `min`, `max`, `count`, `include?`, `first`, `to_a`, `sum`, and
many more) all in terms of a **single** method: `each`. Any class that
defines `each` and `include`s `Enumerable` gets the entire rest of the
list automatically.

## Making your own class Enumerable

```ruby
class Playlist
  include Enumerable

  def initialize
    @songs = []
  end

  def add(song)
    @songs << song
    self
  end

  def each
    return enum_for(:each) unless block_given?   # supports Playlist.new.each with no block

    @songs.each { |song| yield song }
  end
end

playlist = Playlist.new
playlist.add("Bohemian Rhapsody").add("Imagine").add("Hotel California")

# each is the only method WE wrote -- everything below comes from Enumerable
puts playlist.map(&:upcase).inspect
# ["BOHEMIAN RHAPSODY", "IMAGINE", "HOTEL CALIFORNIA"]

puts playlist.select { |song| song.start_with?("H") }.inspect
# ["Hotel California"]

puts playlist.count            # 3
puts playlist.first              # Bohemian Rhapsody
puts playlist.include?("Imagine")   # true
puts playlist.sort.inspect        # ["Bohemian Rhapsody", "Hotel California", "Imagine"]
```

`Playlist` never defined `map`, `select`, `count`, `first`, `include?`, or
`sort` — `Enumerable` implemented every one of them in terms of the `each`
we wrote. This is the same technique
[OOP Deep Dive](01-oop-deep-dive.md) introduced with mixins in general —
`Enumerable` is simply the standard library's own module, built on top of
that mechanism.

## The enum_for detail

`return enum_for(:each) unless block_given?` isn't strictly required for
the examples above, but it's what makes `playlist.each` (with **no**
block) return an `Enumerator` instead of raising — which in turn is what
lets methods like `each_with_index` or chaining
(`playlist.each.with_index`) work correctly. It's a one-line idiom worth
including in every custom `each`.

## Comparable — implementing <=> once, getting <, >, ==, between? for free

`Comparable` works the same way but for ordering: implement the spaceship
operator `<=>` (which returns `-1`, `0`, or `1`), and `Comparable` builds
`<`, `<=`, `==`, `>`, `>=`, `between?`, and `clamp` on top of it.

```ruby
class Money
  include Comparable
  attr_reader :cents

  def initialize(cents)
    @cents = cents
  end

  def <=>(other)
    cents <=> other.cents
  end

  def to_s
    format("$%.2f", cents / 100.0)
  end
end

a = Money.new(500)    # $5.00
b = Money.new(1000)   # $10.00

puts a < b            # true
puts a == Money.new(500)   # true
puts [b, a].sort.map(&:to_s).inspect   # ["$5.00", "$10.00"]
puts a.between?(Money.new(0), Money.new(1000))   # true
puts a.clamp(Money.new(600), Money.new(2000)).to_s   # $6.00 -- clamped up to the minimum
```

Writing one method (`<=>`) gave `Money` a full set of comparison operators,
`sort`, `min`, `max`, `between?`, and `clamp` — all consistent with each
other by construction, since they all defer to the same `<=>`.

## Combining both — a class that's Enumerable AND Comparable

```ruby
class Team
  include Comparable

  attr_reader :name, :wins

  def initialize(name, wins)
    @name = name
    @wins = wins
  end

  def <=>(other)
    wins <=> other.wins
  end

  def to_s
    "#{name} (#{wins} wins)"
  end
end

class League
  include Enumerable

  def initialize
    @teams = []
  end

  def add(team)
    @teams << team
    self
  end

  def each(&block)
    @teams.each(&block)
  end
end

league = League.new
league.add(Team.new("Ravens", 10))
     .add(Team.new("Falcons", 14))
     .add(Team.new("Wolves", 7))

puts league.max.to_s          # Falcons (14 wins) -- Enumerable#max uses Team#<=> automatically
puts league.sort.map(&:to_s)   # ["Wolves (7 wins)", "Ravens (10 wins)", "Falcons (14 wins)"]
puts league.min_by(&:name).to_s   # Falcons (14 wins) -- alphabetically first name, NOT fewest wins
```

Note: `min_by(&:name)` picks whichever team sorts first by *name*
(`"Falcons"` beats `"Ravens"`/`"Wolves"` alphabetically) — it has nothing
to do with `wins`. That's the key distinction: `min`/`max`/`sort` (no
`_by`) use `<=>`, while the `_by` variants compare on whatever block you
give them instead.

## sort vs sort_by — a performance note

```ruby
words = ["banana", "kiwi", "fig", "watermelon"]

# sort_by computes the comparison key ONCE per element (via Schwartzian transform)
puts words.sort_by(&:length).inspect
# ["fig", "kiwi", "banana", "watermelon"]

# sort with a block recomputes .length on every COMPARISON, not every element
puts words.sort { |a, b| a.length <=> b.length }.inspect
# ["fig", "kiwi", "banana", "watermelon"]
```

Both give the same result here, but `sort_by` is generally faster when the
comparison key is expensive to compute, because it's computed once per
element instead of on every pairwise comparison during the sort.

## Cheat sheet

| To get... | Implement... | Then you get for free |
|---|---|---|
| Iteration methods (`map`, `select`, `reduce`, ...) | `each` + `include Enumerable` | `to_a`, `count`, `sort`, `min`, `max`, `include?`, `first`, `sum`, ... |
| Comparison operators | `<=>` + `include Comparable` | `<`, `<=`, `==`, `>=`, `>`, `between?`, `clamp` |
| Sort by a computed key efficiently | n/a | `sort_by { \|x\| key }` over `sort { \|a,b\| ... }` |

## Exercise

Write a class `Inventory` that `include`s `Enumerable`, backed internally
by an array of `Item` structs (`name`, `price`). Implement `each` so that
`inventory.select { |item| item.price > 10 }` and
`inventory.sort_by(&:price)` both work. Then make `Item` itself
`include Comparable`, ordering by `price` via `<=>`, and confirm
`inventory.min` and `inventory.max` (with no block) work correctly because
of it.
