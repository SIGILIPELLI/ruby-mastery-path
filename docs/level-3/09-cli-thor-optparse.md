# 09 · Building CLIs with Thor or OptionParser

Ruby has two common ways to build a real command-line tool with flags and
subcommands: `OptionParser` from the standard library (zero dependencies,
good for a single-command script) and `Thor` (a gem, better for a tool
with multiple subcommands, like `rails` or `bundle` themselves).

## OptionParser — standard library, single command

```ruby
require 'optparse'

options = { verbose: false, count: 1 }

parser = OptionParser.new do |opts|
  opts.banner = "Usage: greet.rb [options] NAME"
  opts.on("-cN", "--count=N", Integer, "Number of greetings") { |n| options[:count] = n }
  opts.on("-v", "--verbose", "Verbose output") { options[:verbose] = true }
  opts.on("-h", "--help", "Show help") { puts opts; exit }
end
parser.parse!(ARGV)

name = ARGV[0] || "World"
options[:count].times do
  puts options[:verbose] ? "[greeting] Hello, #{name}!" : "Hello, #{name}!"
end
```

```text
$ ruby greet.rb --count=2 -v Ruby
[greeting] Hello, Ruby!
[greeting] Hello, Ruby!

$ ruby greet.rb Sam
Hello, Sam!
```

`opts.on` declares one flag, its long form, an optional type
(`Integer` auto-converts and validates), and its help text. `parse!`
mutates `ARGV` in place, stripping out recognized flags and leaving
positional arguments (`ARGV[0]` here) behind for you to read normally.
The `-h`/`--help` block calling `puts opts` prints the auto-generated
usage text built from every `opts.on` line's description.

## Thor — subcommands with declarative options

Thor turns each public method on a class into a CLI subcommand, using
`desc` and `method_option` to declare help text and flags per-command —
the same pattern `rails generate`, `rails db`, etc. use internally:

```ruby
require 'thor'

class Greet < Thor
  desc "hello NAME", "greet NAME"
  method_option :shout, type: :boolean, default: false, aliases: "-s"
  def hello(name)
    msg = "Hello, #{name}!"
    msg = msg.upcase if options[:shout]
    puts msg
  end

  desc "add A B", "add two numbers"
  def add(a, b)
    puts(a.to_i + b.to_i)
  end
end

Greet.start(ARGV)
```

```text
$ ruby greet.rb hello Ruby --shout
HELLO, RUBY!

$ ruby greet.rb add 3 4
7

$ ruby greet.rb help
Commands:
  greet.rb add A B         # add two numbers
  greet.rb hello NAME      # greet NAME
  greet.rb help [COMMAND]  # Describe available commands or one specific c...
  greet.rb tree            # Print a tree of all available commands
```

`method_option` before a method declares a flag scoped to that one
subcommand; inside the method, `options` (a hash) holds the parsed
values. `Greet.start(ARGV)` is the single line that turns `ARGV` into a
method dispatch — Thor figures out which subcommand was named and calls
the matching method with the remaining positional arguments.

## OptionParser vs. Thor — when to use which

- **OptionParser**: one script, one job, a handful of flags — a `bin/`
  script for a single task, no subcommand structure needed, zero gem
  dependencies to install.
- **Thor**: a tool with multiple distinct actions (`mytool build`,
  `mytool deploy`, `mytool status`) that each want their own flags and
  help text — the moment you'd otherwise hand-roll subcommand dispatch
  with a `case ARGV[0]`, Thor is doing that for you plus auto-generated
  help.

## CLI-specific traps

- **`OptionParser#parse!` vs `#parse`** — `parse!` mutates `ARGV`
  destructively, removing recognized flags so leftover positional
  arguments are easy to read afterward; `parse` (no bang) returns a new
  array and leaves `ARGV` untouched, which is easy to forget and then
  wonder why flags are still showing up in `ARGV[0]`.
- **Short flag clustering.** `-cN` in the `opts.on` declaration
  means `-c5` (no space) works, but writing `opts.on("-c", "--count=N")`
  instead requires a space or `=`: `-c 5` or `--count=5`, not `-c5`. The
  exact declaration syntax controls what the parser will accept.
- **Thor method arity must match the CLI call exactly** — `def hello(name)`
  requires exactly one positional argument; calling `ruby greet.rb hello`
  with no name raises `ArgumentError`, not a friendly "missing argument"
  CLI message, unless you give the parameter a default (`def hello(name = "World")`).
- **`Thor#options` is read-only inside the method** and reflects only the
  flags declared for that specific subcommand — flags declared on a
  different subcommand's `method_option` are not visible in a sibling
  method.
- **Forgetting `Type: :boolean`'s default false-vs-nil distinction** —
  a boolean `method_option` without `default: false` returns `nil` when
  not passed rather than `false`, which mostly behaves the same in an
  `if` check but prints differently if you ever inspect `options[:shout]`
  directly.

## Cheat sheet

| Task | OptionParser | Thor |
|---|---|---|
| Declare a flag | `opts.on("-v", "--verbose") { ... }` | `method_option :verbose, type: :boolean` |
| Flag with a value | `opts.on("--count=N", Integer) { \|n\| ... }` | `method_option :count, type: :numeric` |
| Read parsed value | block argument / local hash | `options[:count]` inside the method |
| Positional args | leftover `ARGV` after `parse!` | method parameters |
| Auto-generated help | `opts.banner` + `-h` block | `desc` + built-in `help` command |
| Multiple subcommands | manual `case ARGV[0]` dispatch | one method per subcommand, automatic |
| Entry point | `parser.parse!(ARGV)` | `MyClass.start(ARGV)` |

## Exercise

Build a small CLI tool called `taskcli` with **both** approaches, to feel
the difference directly:

1. As `OptionParser`: a single-command script `bin/task_note.rb` that
   takes a `--priority=LOW|MED|HIGH` flag (default `MED`) and a
   positional task description, printing `"[PRIORITY] description"`.
2. As `Thor`: a `TaskCLI < Thor` class with two subcommands — `add TEXT`
   (appends `TEXT` to an in-memory array and prints a confirmation) and
   `list` (prints every added task, numbered) — note that since each
   invocation is a fresh process, `list` right after `add` in the same
   command won't see it; make `list` accept a `--from` option
   demonstrating how you'd wire real persistence (a file) if you were to
   add it, without actually implementing storage.
3. Run `ruby task_cli.rb help` and paste the generated help output.
