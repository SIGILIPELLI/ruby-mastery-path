# 01 · Setup & First Program

## Install Ruby

Check whether Ruby is already installed:

```bash
ruby -v
# ruby 3.3.0 (or similar)
```

If not, install it:

```bash
# macOS (Homebrew)
brew install ruby

# Ubuntu/Debian
sudo apt install ruby-full

# Windows: use RubyInstaller from rubyinstaller.org
```

## The REPL — irb

`irb` (Interactive Ruby) is a REPL for quick experiments:

```bash
irb
irb(main):001> 2 + 2
=> 4
irb(main):002> puts "hello"
hello
=> nil
irb(main):003> exit
```

Notice `puts` prints `hello` and then irb also shows `=> nil` — that's the
*return value* of the `puts` call itself (which is always `nil`), separate
from what got printed.

## Your first script

Create `hello.rb`:

```ruby
# hello.rb
def greet(name)
  "Hello, #{name}!"
end

puts greet("world")
```

Run it:

```bash
ruby hello.rb
# Hello, world!
```

`#{...}` inside a double-quoted string is **string interpolation** — it
evaluates the expression inside and inserts the result. Single-quoted
strings do *not* interpolate (`'Hello, #{name}!'` would print literally).

## puts vs print vs p

```ruby
puts "hello"    # prints "hello" followed by a newline
print "hello"   # prints "hello" with NO trailing newline
p "hello"       # prints "hello" (with quotes) -- shows the object's inspect form, useful for debugging

puts [1, 2, 3]     # prints each element on its own line
p [1, 2, 3]        # prints [1, 2, 3] -- the array's inspect form, unambiguous
```

`p` is the one to reach for while debugging — it shows exactly what type of
value you have (a string with quotes, `nil` explicitly, etc.), where `puts`
would print `nil` as an empty line.

## Everything is an object

```ruby
puts 5.class          # Integer
puts "hello".class    # String
puts nil.class         # NilClass
puts true.class        # TrueClass

puts 5.even?           # false
puts 5.respond_to?(:even?)   # true
```

Even numbers, strings, and `nil` are full objects with methods you can call
on them — there's no separate "primitive type" concept like in Java or C.

## Choosing an editor

VS Code (with the Ruby extension) or RubyMine both work well. The editor
matters far less than getting comfortable with `irb` for quick checks and
`ruby file.rb` for running real scripts.

## Exercise

Write a script `greet_many.rb` that defines an array of three names and
prints a greeting for each one using the `greet` method above. Then, in
`irb`, experiment with `p` vs `puts` on a hash like `{name: "Ada", age: 30}`
and note the difference in output.
