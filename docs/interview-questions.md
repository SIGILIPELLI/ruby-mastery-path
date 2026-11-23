---
title: "Ruby Interview Questions and Answers (179 Questions)"
description: "179 Ruby interview questions with answers and explanations, from beginner to advanced, organised by level. Free, from the Ruby Mastery Path."
---

# Ruby Interview Questions and Answers

**179 questions** with the correct answer and a short explanation, organised from beginner (Level 1) to advanced (Level 4). Every question comes from a lesson in the free Ruby Mastery Path, so you can open the lesson to go deeper.

!!! tip "Practise, don't just read"
    Each lesson also ends with an interactive quiz, and the [final quiz](quiz.md) tests all four levels. Try to answer each question below before reading the answer.

## Level 1 · Entry

### [01 · Setup &amp; First Program](level-1/01-setup.md)

**Q1.** What is Ruby's design emphasis?

**Answer:** Programmer happiness and readable, expressive code where everything is an object

Ruby was designed by Yukihiro Matsumoto with developer happiness in mind.

**Q2.** What does irb give you?

**Answer:** An interactive Ruby shell to try code line by line

It is handy for experiments.

**Q3.** How do you run a script named hello.rb?

**Answer:** ruby hello.rb

The ruby command runs a file.

**Q4.** Why use a version manager such as rbenv or asdf?

**Answer:** To install and switch between Ruby versions per project

Projects often need different versions.

### [02 · Variables &amp; Types](level-1/02-variables-types.md)

**Q5.** What does this print?

```ruby
a = 'hi'
b = a
b << ' there'
puts a
puts a.equal?(b)
puts 5.class, 5.5.class, nil.class, :sym.class
```

**Answer:** hi there
true
Integer
Float
NilClass
Symbol

Variables reference objects; &lt;&lt; mutates the shared string in place. Verified by compiling and running it.

**Q6.** What does this print?

```ruby
puts 7 / 2
puts 7 % 3
puts 7.0 / 2
puts 2 ** 10
puts 10.divmod(3).inspect
```

**Answer:** 3
1
3.5
1024
[3, 1]

Integer division truncates. Verified by compiling and running it.

**Q7.** Which values are falsy in Ruby?

**Answer:** Only nil and false

Everything else, including 0 and '', is truthy.

**Q8.** What does this print?

```ruby
s = :a
puts s.object_id == :a.object_id
puts 'a'.object_id == 'a'.object_id
puts s.to_proc.call('x') rescue puts 'err'
```

**Answer:** true
false
err

Symbols are unique per name; separate string literals are separate objects (unless frozen literals are enabled). Verified by compiling and running it.

### [03 · Control Flow](level-1/03-control-flow.md)

**Q9.** What does this print?

```ruby
x = 15
r = if x > 10 then 'big' elsif x > 5 then 'mid' else 'small' end
puts r
puts (x.even? ? 'even' : 'odd')
case x
when 1..9 then puts 'single'
when 10..99 then puts 'double'
end
```

**Answer:** big
odd
double

if is an expression and returns a value; case uses ===, so ranges match. Verified by compiling and running it.

**Q10.** What does this print?

```ruby
puts (1..5).select(&:even?).inspect
i = 0
i += 1 while i < 5
puts i
3.times { |n| print n }
puts
puts (1..Float::INFINITY).lazy.map { |n| n * 2 }.first(3).inspect
```

**Answer:** [2, 4]
5
012
[2, 4, 6]

Modifier while, times and lazy enumerators. Verified by compiling and running it.

**Q11.** What does unless do?

**Answer:** Runs the code when the condition is false

It reads as 'if not'.

**Q12.** What does the safe navigation operator &amp;. do?

**Answer:** Returns nil instead of raising when the receiver is nil

user&amp;.name.

### [04 · Methods](level-1/04-methods.md)

**Q13.** What does this print?

```ruby
def greet(name, greeting: 'Hello', punct: '!') = "#{greeting}, #{name}#{punct}"
puts greet('Ana')
puts greet('Bo', greeting: 'Hi')
def sum(*nums) = nums.sum
puts sum(1, 2, 3)
def opts(a, b = a * 2) = [a, b]
puts opts(4).inspect
```

**Answer:** Hello, Ana!
Hi, Bo!
6
[4, 8]

Keyword arguments, splats and defaults referring to earlier parameters. Verified by compiling and running it.

**Q14.** What does a Ruby method return when there is no explicit return?

**Answer:** The value of the last expression evaluated

Explicit return is rarely needed.

**Q15.** By convention, what does a method name ending in ? do?

**Answer:** It returns a boolean (predicate)

A trailing ! often signals a mutating or dangerous variant.

**Q16.** Which is true about method arguments and object mutation?

**Answer:** Ruby passes object references by value, so a method can mutate a passed-in array or string

Use dup to avoid surprising callers.

### [05 · Arrays &amp; Hashes](level-1/05-arrays-hashes.md)

**Q17.** What does this print?

```ruby
a = [3, 1, 2]
puts a.sort.inspect, a.map { |x| x * 2 }.inspect, a.inject(:+)
h = { a: 1, 'b' => 2 }
puts h[:a], h['b'], h[:c].inspect
puts h.fetch(:c, 0)
puts a.each_slice(2).to_a.inspect
```

**Answer:** [1, 2, 3]
[6, 2, 4]
6
1
2
nil
0
[[3, 1], [2]]

sort returns a new array; missing hash keys give nil; fetch takes a default. Verified by compiling and running it.

**Q18.** What does this print?

```ruby
h = Hash.new { |hash, k| hash[k] = [] }
h[:x] << 1
h[:x] << 2
h[:y] << 3
puts h.inspect
puts h.sum { |_, v| v.size }
puts h.transform_values(&:sum).inspect
```

**Answer:** {x: [1, 2], y: [3]}
3
{x: 3, y: 3}

A default block can create missing entries. Verified by compiling and running it.

**Q19.** What is the difference between map and each?

**Answer:** map returns a new array of block results; each returns the receiver and is used for side effects

Choose by intent.

**Q20.** Which method returns only the unique elements of an array?

**Answer:** uniq

uniq! mutates.

### [06 · Strings &amp; String Methods](level-1/06-strings.md)

**Q21.** What does this print?

```ruby
s = 'Ruby'
puts s.upcase, s.reverse, s * 2, s[1..2], s.length
puts "#{s} has #{s.chars.count { |c| 'aeiou'.include?(c.downcase) }} vowel(s)"
puts 'a-b-c'.split('-').inspect
puts 'hello world'.capitalize
```

**Answer:** RUBY
ybuR
RubyRuby
ub
4
Ruby has 1 vowel(s)
["a", "b", "c"]
Hello world

Only u is a vowel in Ruby (y is not counted). Verified by compiling and running it.

**Q22.** What is the difference between single and double quoted strings?

**Answer:** Only double quotes interpolate #{} and process escapes like \n

Use single quotes for plain text.

**Q23.** What does this print?

```ruby
a = 'x'.dup
b = a
b << 'y'
c = a + 'z'
puts a, c
puts a.frozen?
d = 'lit'.freeze
begin; d << 'x'; rescue => e; puts e.class; end
```

**Answer:** xy
xyz
false
FrozenError

&lt;&lt; mutates, + builds a new string, frozen strings raise FrozenError. Verified by compiling and running it.

**Q24.** Which method formats a number as a padded string?

**Answer:** format or sprintf, for example format('%05.1f', 3.14159)

String#% also works.

### [07 · Classes &amp; Objects Basics](level-1/07-classes-objects.md)

**Q25.** What does this print?

```ruby
class Dog
  attr_reader :name
  @@count = 0
  def initialize(name); @name = name; @@count += 1; end
  def self.count = @@count
  def to_s = "Dog(#{@name})"
end
d = Dog.new('Rex'); Dog.new('Fido')
puts d, Dog.count
puts d.respond_to?(:name=)
puts d.instance_variables.inspect
```

**Answer:** Dog(Rex)
2
false
[:@name]

attr_reader defines only a getter; class variables are shared. Verified by compiling and running it.

**Q26.** What does attr_accessor define?

**Answer:** A getter and a setter for an instance variable

attr_reader and attr_writer define one each.

**Q27.** What do public, private and protected control?

**Answer:** Who may call a method: anyone, only the object itself, or instances of the same class family

private methods are called without an explicit receiver.

**Q28.** What is self inside an instance method?

**Answer:** The object receiving the call

Inside a class body self is the class.

### [08 · Blocks, Procs &amp; Lambdas Introduction](level-1/08-blocks-procs-lambdas.md)

**Q29.** What does this print?

```ruby
add = ->(a, b) { a + b }
double = proc { |x| (x || 0) * 2 }
puts add.call(1, 2), double.call(4), double.call, double.lambda?
puts add.curry[1][2]
def twice; yield; yield; end
n = 0
twice { n += 5 }
puts n
```

**Answer:** 3
8
0
false
3
10

Procs are lenient about arguments; lambdas are strict. Verified by compiling and running it.

**Q30.** What does this print?

```ruby
def run
  [1, 2, 3].each { |x| return x * 10 if x == 2 }
  :done
end
puts run
l = lambda { return 1; 2 }
puts l.call
puts [1, 2, 3].map(&:to_s).inspect
```

**Answer:** 20
1
["1", "2", "3"]

return in a block returns from the enclosing method; in a lambda it returns from the lambda. Verified by compiling and running it.

**Q31.** What does the &amp; in &amp;:upcase do?

**Answer:** Converts the symbol to a block via Symbol#to_proc

%w[a b].map(&amp;:upcase).

**Q32.** How does a method receive a block?

**Answer:** Implicitly with yield, or explicitly as &amp;block

block_given? checks.

### [09 · Gems &amp; Bundler Basics](level-1/09-gems-bundler.md)

**Q33.** What does Bundler do?

**Answer:** Installs and locks the gem versions listed in a Gemfile

Gemfile.lock pins exact versions.

**Q34.** Which command runs a script using the Gemfile's gems?

**Answer:** bundle exec ruby script.rb

It ensures the locked versions load.

**Q35.** What should you commit to version control for an application?

**Answer:** Gemfile and Gemfile.lock

The lockfile makes builds reproducible.

**Q36.** What does gem 'rack', '~&gt; 3.0' allow?

**Answer:** Any 3.x version at or above 3.0 but below 4.0

The pessimistic operator.

### [10 · Project — CLI To-Do App](level-1/10-project-todo-app.md)

**Q37.** What does this print?

```ruby
Todo = Struct.new(:title, :done) do
  def to_s = "[#{done ? 'x' : ' '}] #{title}"
end
list = [Todo.new('a', false), Todo.new('b', true)]
list[0].done = true
puts list
puts list.count(&:done)
puts list.reject(&:done).size
```

**Answer:** [x] a
[x] b
2
0

Struct gives accessors and equality for free. Verified by compiling and running it.

**Q38.** How would you persist the todo list between runs?

**Answer:** Write it as JSON (or YAML) to a file and load it at startup

The project saves to a file.

**Q39.** How do you read a line from the user in a CLI?

**Answer:** gets

gets.chomp removes the newline.

**Q40.** Why keep the todo logic in a class separate from the CLI loop?

**Answer:** So the logic can be tested without input and output

Separate concerns.

## Level 2 · Intermediate

### [01 · OOP Deep Dive](level-2/01-oop-deep-dive.md)

**Q41.** What does this print?

```ruby
module Greet
  def hi = "hi from #{self.class}"
end
class A
  include Greet
end
class B < A
  def hi = super + '!'
end
puts A.new.hi, B.new.hi
puts B.ancestors.first(3).inspect
puts B.instance_method(:hi).owner
```

**Answer:** hi from A
hi from B!
[B, A, Greet]
B

include inserts the module after the class in the ancestor chain; super calls the next method up. Verified by compiling and running it.

**Q42.** What is the difference between include and extend?

**Answer:** include adds module methods as instance methods; extend adds them as singleton (class-level) methods

Both mix in behaviour.

**Q43.** What does Comparable need to work?

**Answer:** A &lt;=&gt; method that returns -1, 0 or 1

Then &lt;, &gt;, between? and clamp work.

**Q44.** Why prefer composition over inheritance?

**Answer:** It is more flexible and avoids deep, rigid hierarchies

Use modules and collaborating objects.

### [02 · Blocks/Procs/Lambdas Advanced](level-2/02-blocks-procs-lambdas-advanced.md)

**Q45.** What does this print?

```ruby
counter = 0
inc = -> { counter += 1 }
3.times(&inc) rescue 3.times { inc.call }
puts counter
def make_counter
  n = 0
  -> { n += 1 }
end
c = make_counter
c.call; c.call
puts c.call
puts c.binding.local_variable_get(:n)
```

**Answer:** 3
3
3

Closures capture and keep the variable alive. Verified by compiling and running it.

**Q46.** What does this print?

```ruby
compose = ->(f, g) { ->(x) { g.(f.(x)) } }
inc = ->(x) { x + 1 }
dbl = ->(x) { x * 2 }
puts compose.(inc, dbl).(5)
puts (inc >> dbl).call(5), (inc << dbl).call(5)
puts %w[a b].each_with_index.map { |s, i| "#{i}#{s}" }.inspect
```

**Answer:** 12
12
11
["0a", "1b"]

&gt;&gt; applies left then right; &lt;&lt; applies right then left. Verified by compiling and running it.

**Q47.** What does method(:puts) return?

**Answer:** A Method object that can be passed around and called

Use it with map(&amp;method(:name)).

**Q48.** What does yield do inside a method?

**Answer:** Calls the block passed to that method

Use block_given? first when the block is optional.

### [03 · Exception Handling](level-2/03-exception-handling.md)

**Q49.** What does this print?

```ruby
def risky(n)
  raise ArgumentError, 'neg' if n < 0
  raise IOError, 'io' if n == 0
  n
rescue ArgumentError => e
  "arg: #{e.message}"
else
  'fine'
ensure
  print 'E '
end
puts risky(-1)
puts risky(1)
begin
  risky(0)
rescue => e
  puts e.class
end
```

**Answer:** E arg: neg
E fine
E IOError

ensure always runs; else runs only when nothing raised; the IOError is not rescued by the ArgumentError clause so it propagates after ensure runs. Verified by compiling and running it.

**Q50.** What is the right way to define a custom error?

**Answer:** class MyError &lt; StandardError; end

Rescuing Exception is too broad.

**Q51.** What does retry do inside a rescue?

**Answer:** Re-runs the begin block

Limit attempts to avoid infinite loops.

**Q52.** Why rescue specific errors instead of bare rescue?

**Answer:** Bare rescue hides unrelated bugs

Rescue only what you can handle.

### [04 · File I/O](level-2/04-file-io.md)

**Q53.** What does this print?

```ruby
require 'tmpdir'
Dir.mktmpdir do |d|
  path = File.join(d, 'a.txt')
  File.write(path, "one\ntwo\n")
  File.open(path, 'a') { |f| f.puts 'three' }
  puts File.readlines(path, chomp: true).inspect
  puts File.foreach(path).count
  puts File.exist?(path), File.basename(path, '.txt')
end
```

**Answer:** ["one", "two", "three"]
3
true
a

Mode 'a' appends; File.open with a block closes the file. Verified by compiling and running it.

**Q54.** Why use File.open with a block?

**Answer:** It closes the file automatically, even on error

Avoid leaking file handles.

**Q55.** Which reads a huge file without loading it all?

**Answer:** File.foreach(path) { |line| ... }

Streaming line by line.

**Q56.** What does File.join('a', 'b') do?

**Answer:** Builds a path using the right separator

Portable paths.

### [05 · Testing with RSpec](level-2/05-testing-rspec.md)

**Q57.** What does RSpec's expect(x).to eq(3) check?

**Answer:** That x == 3

Use be for identity.

**Q58.** What does let do?

**Answer:** Defines a lazily evaluated, memoised helper for each example

let! evaluates eagerly.

**Q59.** What is a test double?

**Answer:** A fake object that stands in for a collaborator

Use doubles to isolate the unit.

**Q60.** Which is a good example name?

**Answer:** 'returns an empty list when there are no todos'

Examples should read as documentation.

### [06 · Working with JSON/APIs](level-2/06-json-apis.md)

**Q61.** What does this print?

```ruby
require 'json'
s = '{"name":"Ana","tags":["a","b"],"n":1.5}'
h = JSON.parse(s)
puts h['name'], h['tags'].size, h['n'].class
puts JSON.parse(s, symbolize_names: true)[:tags].inspect
puts({ a: 1, b: [1, 2] }.to_json)
puts JSON.pretty_generate({ k: 1 })
```

**Answer:** Ana
2
Float
["a", "b"]
{"a":1,"b":[1,2]}
{
  "k": 1
}

JSON.parse returns string keys unless symbolize_names is set. Verified by compiling and running it.

**Q62.** What exception does JSON.parse raise on malformed input?

**Answer:** JSON::ParserError

Rescue it when parsing untrusted data.

**Q63.** Which standard library class makes HTTP requests?

**Answer:** Net::HTTP

Add timeouts.

**Q64.** Why set timeouts on HTTP calls?

**Answer:** A hung server would otherwise block your program indefinitely

open_timeout and read_timeout.

### [07 · Metaprogramming Basics](level-2/07-metaprogramming-basics.md)

**Q65.** What does this print?

```ruby
class Tag
  %w[a b].each { |n| define_method("#{n}?") { @v == n } }
  def initialize(v) = @v = v
  def method_missing(name, *args)
    name.to_s.start_with?('get_') ? name.to_s.sub('get_', '') : super
  end
  def respond_to_missing?(name, priv = false) = name.to_s.start_with?('get_') || super
end
t = Tag.new('a')
puts t.a?, t.b?, t.get_color, t.respond_to?(:get_x)
begin; t.zzz; rescue NoMethodError => e; puts 'NoMethodError'; end
puts t.send(:a?)
```

**Answer:** true
false
color
true
NoMethodError
true

define_method creates methods dynamically; always pair method_missing with respond_to_missing?. Verified by compiling and running it.

**Q66.** What does send do?

**Answer:** Calls a method by name, even a private one

public_send respects visibility.

**Q67.** What is a risk of overusing metaprogramming?

**Answer:** Code becomes hard to read, search and debug

Use it where it removes real duplication.

**Q68.** What does instance_variable_get(:@x) do?

**Answer:** Reads an instance variable by name

Mostly for frameworks and tests.

### [08 · Gem Development Basics](level-2/08-gem-development.md)

**Q69.** What is the conventional gem layout?

**Answer:** lib/ for code, spec/ or test/ for tests, a .gemspec file and a version constant

bundle gem scaffolds it.

**Q70.** What does the .gemspec file contain?

**Answer:** The gem's name, version, authors, files and dependencies

Build with gem build.

**Q71.** What does semantic versioning say about 2.0.0 to 3.0.0?

**Answer:** A major bump signals breaking changes

Patch is fixes, minor is compatible features.

**Q72.** Why add dependencies sparingly to a gem?

**Answer:** Every dependency burdens users with conflicts and updates

Prefer the standard library.

### [09 · Enumerable &amp; Comparable](level-2/09-enumerable-comparable.md)

**Q73.** What does this print?

```ruby
class Bag
  include Enumerable
  def initialize(*x) = @x = x
  def each(&b) = @x.each(&b)
end
b = Bag.new(3, 1, 2, 2)
puts b.sort.inspect, b.min, b.include?(2), b.tally.inspect
puts b.each_cons(2).map { |p, q| q - p }.inspect
puts b.group_by(&:odd?).inspect
puts b.partition { |n| n > 1 }.inspect
```

**Answer:** [1, 2, 2, 3]
1
true
{3 =&gt; 1, 1 =&gt; 1, 2 =&gt; 2}
[-2, 1, 0]
{true =&gt; [3, 1], false =&gt; [2, 2]}
[[3, 2, 2], [1]]

Define each and include Enumerable to get map, sort, min, tally and more. Verified by compiling and running it.

**Q74.** What does sort_by { |x| -x.size } do?

**Answer:** Sorts descending by size using the computed key

sort_by is efficient because it computes each key once.

**Q75.** What is the difference between select and find?

**Answer:** select returns all matches; find returns the first or nil

detect is an alias of find.

**Q76.** What does each_with_object do?

**Answer:** Iterates while passing a memo object you build up, returning it

A clean alternative to inject for hashes.

### [10 · Project — Weather CLI](level-2/10-project-weather-cli.md)

**Q77.** In a weather CLI, where should parsing and formatting logic live?

**Answer:** In small classes or methods separate from the HTTP call

So you can test without the network.

**Q78.** How would you test the CLI's API layer without hitting the real API?

**Answer:** Stub or inject the HTTP client and return canned JSON

Deterministic tests.

**Q79.** What should the CLI do on a network failure?

**Answer:** Print a clear message and exit with a non-zero status

exit 1.

**Q80.** Why cache API responses briefly?

**Answer:** To reduce calls, rate-limit risk and latency

Respect API terms.

## Level 3 · Advanced

### [01 · Building Web Apps with Sinatra](level-3/01-sinatra.md)

**Q81.** What is Sinatra?

**Answer:** A lightweight Ruby web framework built on Rack

get '/' do ... end.

**Q82.** What does Rack provide?

**Answer:** A common interface between Ruby web servers and applications

An app is an object responding to call(env).

**Q83.** In Sinatra, what does params contain?

**Answer:** Query string, form and route parameters (as strings)

Validate and convert them.

**Q84.** How do you return JSON?

**Answer:** Set content_type :json and return a JSON string

to_json.

### [02 · Databases with ActiveRecord](level-3/02-activerecord.md)

**Q85.** What does ActiveRecord implement?

**Answer:** The pattern that maps a database table to a class and rows to objects

Active Record pattern.

**Q86.** What is the N+1 query problem?

**Answer:** Loading a list then querying again per item; fix with includes (eager loading)

Use includes or preload.

**Q87.** What does a migration do?

**Answer:** Describes a versioned change to the database schema

Run with db:migrate.

**Q88.** Why use validations and constraints together?

**Answer:** Model validations give friendly errors; database constraints guarantee integrity

Races can bypass validations.

### [03 · Testing Advanced](level-3/03-testing-advanced.md)

**Q89.** What is a test factory (e.g. FactoryBot)?

**Answer:** A helper that builds valid test records with overridable attributes

Avoid brittle fixtures.

**Q90.** Why keep tests independent?

**Answer:** Order and shared state must not affect results

Reset state between examples.

**Q91.** What does stubbing Time or the clock help with?

**Answer:** Making time-dependent tests deterministic

Freeze time.

**Q92.** What is code coverage not?

**Answer:** Proof that behaviour is correct

100% coverage can still miss bugs.

### [04 · Metaprogramming Advanced](level-3/04-metaprogramming-advanced.md)

**Q93.** What does this print?

```ruby
class Model
  def self.fields(*names)
    names.each do |n|
      define_method(n) { @data[n] }
      define_method("#{n}=") { |v| @data[n] = v }
    end
  end
  def initialize = @data = {}
end
class User < Model
  fields :name, :age
end
u = User.new
u.name = 'Ana'; u.age = 30
puts u.name, u.age
puts User.instance_methods(false).sort.inspect
```

**Answer:** Ana
30
[:age, :age=, :name, :name=]

A class macro defines methods at load time. Verified by compiling and running it.

**Q94.** What does instance_eval do?

**Answer:** Evaluates a block with self set to the receiver, the basis of many DSLs

It enables clean configuration blocks.

**Q95.** What does Module#prepend do?

**Answer:** Puts the module before the class in the method lookup so it can wrap methods

Useful for decorators and instrumentation.

**Q96.** What is a safer alternative to monkey patching core classes?

**Answer:** Refinements or wrapper objects

Limit the scope of change.

### [05 · Design Patterns in Ruby](level-3/05-design-patterns.md)

**Q97.** Which pattern lets you swap algorithms at runtime?

**Answer:** Strategy

Ruby often uses blocks or duck typing.

**Q98.** What does the Observer pattern do?

**Answer:** Notifies subscribers when an object changes

Ruby ships an Observable module in older versions.

**Q99.** What does duck typing mean?

**Answer:** Care about what an object can do (respond to), not its class

If it quacks.

**Q100.** When is a Singleton a smell?

**Answer:** When it hides global mutable state that makes testing hard

Pass dependencies instead.

### [06 · Performance &amp; Profiling](level-3/06-performance-profiling.md)

**Q101.** Which is the first step to speed up slow Ruby code?

**Answer:** Profile or benchmark to find the real bottleneck

Measure first.

**Q102.** What does this print?

```ruby
require 'benchmark'
r = Benchmark.realtime { 100_000.times.map { |i| i * 2 } }
puts r.class
puts (1..5).sum == 15
puts %w[a b c].each_with_index.to_a.flatten.size
```

**Answer:** Float
true
6

Benchmark.realtime returns seconds as a Float. Verified by compiling and running it.

**Q103.** Why does building a string with += in a loop cost more than &lt;&lt; ?

**Answer:** += creates a new string each time; &lt;&lt; appends in place

Use &lt;&lt; or an array and join.

**Q104.** What reduces memory use in large collections?

**Answer:** Lazy enumerators and streaming instead of building big arrays

(1..Float::INFINITY).lazy.

### [07 · Concurrency in Ruby](level-3/07-concurrency.md)

**Q105.** Which Ruby implementation has a Global VM Lock (GVL) limiting parallel Ruby code in threads?

**Answer:** MRI (CRuby)

Threads still help with I/O waits.

**Q106.** What does this print?

```ruby
m = Mutex.new
count = 0
ts = 5.times.map { Thread.new { 1000.times { m.synchronize { count += 1 } } } }
ts.each(&:join)
puts count
q = Queue.new
Th = Thread.new { 3.times { |i| q << i }; q << :done }
out = []
while (v = q.pop) != :done; out << v; end
puts out.inspect
```

**Answer:** 5000
[0, 1, 2]

A Mutex protects the shared counter; Queue is thread-safe. Verified by compiling and running it.

**Q107.** What do Ractors (or processes) offer over threads?

**Answer:** True parallelism for CPU-bound work without sharing mutable state

Choose by workload.

**Q108.** What is a race condition?

**Answer:** A bug where the result depends on unpredictable timing of concurrent access

Protect shared state.

### [08 · Working with Rake Tasks](level-3/08-rake-tasks.md)

**Q109.** What is Rake?

**Answer:** A Ruby build and task runner defined in a Rakefile

rake -T lists tasks.

**Q110.** How do you declare a task dependency?

**Answer:** task deploy: [:test] do ... end

Prerequisites run first.

**Q111.** What does a namespace do in Rake?

**Answer:** Groups tasks, for example db:migrate

Keeps names tidy.

**Q112.** How do you pass arguments to a task?

**Answer:** task :greet, [:name] do |t, args| ... end

rake greet[Ana].

### [09 · Building CLIs with Thor or OptionParser](level-3/09-cli-thor-optparse.md)

**Q113.** What does OptionParser do?

**Answer:** Parses command-line flags and generates help

Part of the standard library.

**Q114.** What does Thor add over OptionParser?

**Answer:** Subcommands, descriptions and a class-based CLI structure

Like git add / git commit.

**Q115.** What exit code signals failure?

**Answer:** Non-zero

exit 1.

**Q116.** Where should error messages go?

**Answer:** Standard error ($stderr)

So pipes of stdout stay clean.

### [10 · Project — REST API Service](level-3/10-project-rest-api.md)

**Q117.** In the REST API project, which HTTP status means 'created'?

**Answer:** 201

Return Location header too.

**Q118.** Which status for invalid input?

**Answer:** 422 (or 400)

Return error details.

**Q119.** What does idempotent mean for PUT and DELETE?

**Answer:** Repeating the request has the same effect as doing it once

Design retries around it.

**Q120.** Why version your API?

**Answer:** So breaking changes do not break existing clients

/v1 or headers.

## Level 4 · Master

### [01 · Rails/Production Patterns](level-4/01-production-patterns.md)

**Q121.** What belongs in environment variables rather than code?

**Answer:** Secrets and environment-specific configuration

Twelve-factor style.

**Q122.** What is structured logging?

**Answer:** Writing machine-readable (JSON) log lines with fields

Easier to search.

**Q123.** Why add health-check endpoints?

**Answer:** Orchestrators and load balancers can tell if the app is up

Separate liveness and readiness.

**Q124.** What is graceful shutdown?

**Answer:** Finishing in-flight work on SIGTERM before exiting

Avoid dropped requests.

### [02 · Background Jobs (Sidekiq)](level-4/02-background-jobs.md)

**Q125.** Why run work in background jobs?

**Answer:** Slow work (emails, reports) should not block web requests

Sidekiq and similar.

**Q126.** Why must jobs be idempotent?

**Answer:** Retries and duplicate deliveries mean a job can run more than once

Use unique keys.

**Q127.** What should jobs receive as arguments?

**Answer:** Small serialisable IDs, not whole objects

Reload fresh data inside.

**Q128.** What does a retry with backoff do?

**Answer:** Waits longer between attempts to avoid hammering a failing service

Cap retries and dead-letter.

### [03 · Security Best Practices](level-4/03-security-best-practices.md)

**Q129.** What does SQL injection exploit?

**Answer:** Untrusted input concatenated into SQL; use parameters instead

Use placeholders.

**Q130.** What is mass assignment risk?

**Answer:** Letting users set attributes they should not, such as admin; use strong parameters

Whitelist fields.

**Q131.** How should you store passwords?

**Answer:** Hashed with a slow salted algorithm such as bcrypt

Never encrypt reversibly.

**Q132.** What do you do about vulnerable gems?

**Answer:** Run bundler-audit or similar and update

Automate in CI.

### [04 · Testing at Scale &amp; CI](level-4/04-testing-at-scale-ci.md)

**Q133.** What makes CI test runs reliable?

**Answer:** Isolated, deterministic tests run in a clean environment

Flaky tests erode trust.

**Q134.** How can you speed up a large test suite?

**Answer:** Parallelise, cache dependencies and run affected tests first

Keep feedback fast.

**Q135.** What should you do with a flaky test?

**Answer:** Fix or quarantine it and track the root cause

Time, ordering and external services are common causes.

**Q136.** What does a test pyramid recommend?

**Answer:** Many fast unit tests, fewer integration tests, few end-to-end tests

Balance cost and confidence.

### [05 · Deployment (Docker)](level-4/05-deployment-docker.md)

**Q137.** Why use a multi-stage Docker build for a Ruby app?

**Answer:** To keep build tools out of the final, smaller image

Smaller attack surface.

**Q138.** Why copy the Gemfile before the rest of the code in a Dockerfile?

**Answer:** So the bundle install layer is cached until dependencies change

Layer caching.

**Q139.** Which user should the container run as?

**Answer:** A non-root user

Least privilege.

**Q140.** Where do secrets go for a container?

**Answer:** Injected at runtime via environment or secret store, not baked into the image

Images are shareable artefacts.

### [06 · Performance at Scale](level-4/06-performance-at-scale.md)

**Q141.** What is usually the first scaling bottleneck of a Rails/Ruby app?

**Answer:** The database (queries and N+1)

Fix queries first.

**Q142.** What does HTTP caching (ETag) save?

**Answer:** Bandwidth and work when the resource has not changed

304 Not Modified.

**Q143.** Why use connection pooling?

**Answer:** Opening database connections is expensive and limited

Size the pool to threads.

**Q144.** Why measure p95/p99 latency, not just the average?

**Answer:** Tail latency is what unhappy users experience

Averages hide outliers.

### [07 · API Design &amp; Versioning](level-4/07-api-design-versioning.md)

**Q145.** How should an API communicate breaking changes?

**Answer:** New version (path or header) with a deprecation period

Give clients time.

**Q146.** What is pagination for?

**Answer:** Returning large collections in pages to limit size and cost

Cursor pagination is stable.

**Q147.** What is a good error response?

**Answer:** A consistent structure with a code, message and details, and the right status

Never leak internals.

**Q148.** Why rate-limit an API?

**Answer:** To protect against abuse and overload

Return 429 with Retry-After.

### [08 · Message Queues](level-4/08-message-queues.md)

**Q149.** What do message queues decouple?

**Answer:** Producers from consumers in time and rate

Work buffers when consumers are slow.

**Q150.** What does at-least-once delivery imply?

**Answer:** Consumers must handle duplicates

Idempotency again.

**Q151.** What is a dead-letter queue?

**Answer:** Where messages go after repeated failures for inspection

Alert on it.

**Q152.** Why ack a message only after processing succeeds?

**Answer:** So a crash redelivers it instead of losing it

Ack late.

### [09 · Code Quality &amp; RuboCop](level-4/09-code-quality-rubocop.md)

**Q153.** What does RuboCop do?

**Answer:** Checks style and common mistakes against configurable cops

Run in CI.

**Q154.** What is auto-correct (-a) good for?

**Answer:** Safe style fixes; review the diff

-A includes unsafe cops.

**Q155.** Why agree a team style config?

**Answer:** Consistent code reduces review noise

Commit .rubocop.yml.

**Q156.** Should you disable cops everywhere that annoy you?

**Answer:** Disable narrowly with a reason, preferably in config

Avoid blanket rubocop:disable.

### [10 · Capstone Project](level-4/10-capstone-project.md)

**Q157.** What is the capstone's aim?

**Answer:** Combine API, persistence, tests, jobs, security and deployment into one production-style project

Prove readiness.

**Q158.** What should the README include?

**Answer:** Setup, run, test, deploy steps and design decisions

Reviewers start there.

**Q159.** What is a good definition of done?

**Answer:** Tests pass in CI, lint is clean and it deploys from a clean checkout

Reproducible.

**Q160.** Why add monitoring and logs to the capstone?

**Answer:** To see failures and performance in production

You cannot fix what you cannot see.

## Mixed questions across all levels

**Q161.** What does this print?

```ruby
h = Hash.new(0)
'banana'.each_char { |c| h[c] += 1 }
puts h.inspect
puts h.max_by { |_, v| v }.first
```

**Answer:** {"b" =&gt; 1, "a" =&gt; 3, "n" =&gt; 2}
a

Hash.new(0) supplies a default of 0 for missing keys; insertion order is kept. Verified by compiling and running it.

**Q162.** What does this print?

```ruby
puts [1, 2, 3].each_with_object([]) { |x, memo| memo << x * x }.inspect
puts (1..4).inject(:*)
puts [[1, 2], [3, 4]].to_h { |a, b| [a, b * 2] }.inspect
```

**Answer:** [1, 4, 9]
24
{1 =&gt; 4, 3 =&gt; 8}

each_with_object returns the memo; inject(:*) multiplies; to_h with a block builds a hash. Verified by compiling and running it.

**Q163.** What does this print?

```ruby
def f
  yield 1, 2
end
f { |a| puts a.inspect }
f { |a, b| puts [a, b].inspect }
f { |*all| puts all.inspect }
pr = proc { |a, b| [a, b] }
puts pr.call([5, 6]).inspect
```

**Answer:** 1
[1, 2]
[1, 2]
[5, 6]

Blocks and procs are lenient: extra args are dropped and a single array is destructured. Verified by compiling and running it.

**Q164.** What does this print?

```ruby
module Loud
  def speak = super.upcase
end
class Animal
  def speak = 'hi'
end
class Dog < Animal
  prepend Loud
  def speak = 'woof'
end
puts Dog.new.speak
puts Dog.ancestors.first(3).inspect
```

**Answer:** WOOF
[Loud, Dog, Animal]

prepend puts the module in front of the class so it wraps the class's own method. Verified by compiling and running it.

**Q165.** What does this print?

```ruby
class Money
  include Comparable
  attr_reader :cents
  def initialize(c) = @cents = c
  def <=>(o) = cents <=> o.cents
  def to_s = format('$%.2f', cents / 100.0)
end
a, b = Money.new(500), Money.new(1250)
puts a < b, [b, a].min, a.clamp(b, b), a == Money.new(500)
```

**Answer:** true
$5.00
$12.50
true

Comparable derives &lt;, min, clamp and == from &lt;=&gt;. Verified by compiling and running it.

**Q166.** A method has a bare `rescue` and a typo inside the begin block. What goes wrong?

**Answer:** The NameError or NoMethodError is swallowed and hidden; rescue specific errors

A bare rescue catches StandardError, which includes typos.

**Q167.** Why should a custom error inherit from StandardError, not Exception?

**Answer:** A bare rescue only catches StandardError; Exception also covers exit and interrupts

Rescuing Exception can trap Ctrl-C.

**Q168.** What does Bundler's Gemfile.lock guarantee?

**Answer:** Every machine installs the same gem versions

Commit it for applications.

**Q169.** What is the N+1 query problem and its Rails fix?

**Answer:** One query for a list plus one per item; use includes/preload

Eager loading.

**Q170.** How do you store passwords in a Ruby app?

**Answer:** bcrypt (or argon2) hashes with salt, never reversible encryption

has_secure_password uses bcrypt.

**Q171.** What is the safe way to put user input in an SQL query?

**Answer:** Bind parameters or the ORM's placeholders, never string interpolation

Prevents SQL injection.

**Q172.** Why make background jobs idempotent?

**Answer:** Retries and duplicates mean a job may run more than once

Use unique keys or check state first.

**Q173.** What does GVL mean for CRuby threads?

**Answer:** Only one thread runs Ruby code at a time, though threads can overlap I/O waits

Use processes or Ractors for CPU parallelism.

**Q174.** Which structure protects a shared counter from a race condition?

**Answer:** A Mutex around the read-modify-write

synchronize.

**Q175.** Why run containers as a non-root user?

**Answer:** It limits damage if the app is compromised

Least privilege.

**Q176.** What HTTP status fits 'validation failed' in an API?

**Answer:** 422 (or 400) with error details

Be consistent.

**Q177.** What does RuboCop do in CI?

**Answer:** Fails the build on style and common-mistake violations

Keeps reviews about design.

**Q178.** What does at-least-once message delivery require of a consumer?

**Answer:** Idempotent handling of duplicates

Ack after processing.

**Q179.** Which is a good definition of done for the capstone?

**Answer:** Tests pass in CI, lint is clean and it deploys from a clean checkout

Reproducibility is the proof.

## Practical coding questions

### Word frequency report

Implement words, top and report so the script prints a stop-word-filtered frequency chart. Ties are broken alphabetically and an empty text must not crash.

Starter code:

```
TEXT = <<~T
  The quick brown fox jumps over the lazy dog. The dog barks; the fox runs!
  A quick reply: the fox is quick, the dog is lazy.
T
STOP = %w[the a is over].freeze

# TODO: words(text)   -> lowercase words (letters and apostrophes only), stop words removed
# TODO: top(text, n)  -> the n most frequent [word, count] pairs, ties broken alphabetically
# TODO: report(text, n) -> lines like "dog   ### 3": word padded to the widest word, one # per count, then the count

puts "words: #{words(TEXT).size}, unique: #{words(TEXT).uniq.size}"
puts report(TEXT, 4)
puts top('', 3).inspect
```

Expected output:

```text
words: 16, unique: 9
dog   ### 3
fox   ### 3
quick ### 3
lazy  ## 2
[]
```

??? success "Show a solution"

    ```
    TEXT = <<~T
      The quick brown fox jumps over the lazy dog. The dog barks; the fox runs!
      A quick reply: the fox is quick, the dog is lazy.
    T
    STOP = %w[the a is over].freeze

    def words(text)
      text.downcase.scan(/[a-z']+/).reject { |w| STOP.include?(w) }
    end

    def top(text, n)
      words(text).tally.sort_by { |w, c| [-c, w] }.first(n)
    end

    def report(text, n)
      rows = top(text, n)
      width = rows.map { |w, _| w.length }.max
      rows.map { |w, c| format("%-#{width}s %s %d", w, '#' * c, c) }
    end

    puts "words: #{words(TEXT).size}, unique: #{words(TEXT).uniq.size}"
    puts report(TEXT, 4)
    puts top('', 3).inspect
    ```

    scan with a regex extracts words, tally counts them and sort_by with [-count, word] gives a stable ranking. Verified by running the solution with Ruby 4.

### A Retrier with exponential backoff

Implement Retrier so transient failures are retried with exponentially growing waits, permanent failures are not retried and the log always records the number of attempts. The sleeper is injected so no real time passes.

Starter code:

```
class TransientError < StandardError; end
class PermanentError < StandardError; end

# TODO: class Retrier with attr_reader :log
#   initialize(max: 4, base: 0.5, sleeper: lambda that logs "sleep <seconds>")
#   run { |attempt| ... } retries only TransientError, waiting base * 2**(attempt-1) between tries,
#   re-raises after max attempts, logs PermanentError ("permanent: msg") without retrying,
#   and always logs "done after <attempts>" in ensure. Logs "attempt N failed: msg" and "ok on attempt N".

r = Retrier.new
v = r.run { |n| n < 3 ? raise(TransientError, "boom#{n}") : "value" }
puts v, r.log
r2 = Retrier.new(max: 2)
begin
  r2.run { |n| raise TransientError, "down" }
rescue TransientError => e
  puts "gave up: #{e.message}"
end
puts r2.log
r3 = Retrier.new
begin
  r3.run { raise PermanentError, "bad input" }
rescue PermanentError
  puts r3.log
end
```

Expected output:

```text
value
attempt 1 failed: boom1
sleep 0.5
attempt 2 failed: boom2
sleep 1.0
ok on attempt 3
done after 3
gave up: down
attempt 1 failed: down
sleep 0.5
attempt 2 failed: down
done after 2
permanent: bad input
done after 1
```

??? success "Show a solution"

    ```
    class TransientError < StandardError; end
    class PermanentError < StandardError; end

    class Retrier
      attr_reader :log

      def initialize(max: 4, base: 0.5, sleeper: ->(s) { @log << "sleep #{s}" })
        @max, @base, @sleeper, @log = max, base, sleeper, []
      end

      def run
        attempt = 0
        begin
          attempt += 1
          result = yield attempt
          @log << "ok on attempt #{attempt}"
          result
        rescue TransientError => e
          @log << "attempt #{attempt} failed: #{e.message}"
          raise if attempt >= @max
          @sleeper.call(@base * 2**(attempt - 1))
          retry
        rescue PermanentError => e
          @log << "permanent: #{e.message}"
          raise
        ensure
          @log << "done after #{attempt}"
        end
      end
    end

    r = Retrier.new
    v = r.run { |n| n < 3 ? raise(TransientError, "boom#{n}") : "value" }
    puts v, r.log
    r2 = Retrier.new(max: 2)
    begin
      r2.run { |n| raise TransientError, "down" }
    rescue TransientError => e
      puts "gave up: #{e.message}"
    end
    puts r2.log
    r3 = Retrier.new
    begin
      r3.run { raise PermanentError, "bad input" }
    rescue PermanentError
      puts r3.log
    end
    ```

    retry re-runs the begin block, the injected sleeper keeps the test deterministic, and ensure records the final attempt count once. Verified by running the solution with Ruby 4.

---

More free practice: the [Ruby Mastery Path](index.md) curriculum, the [Interview Prep Pack](https://learnmasterypath.com/interview-pack/) and the [free resume ATS check](https://learnmasterypath.com/resume/).
