# 08 · Working with Rake Tasks

Rake ("Ruby Make") is Ruby's build-tool DSL — the thing running behind
`rails db:migrate`, `rake spec`, and countless custom project chores. A
`Rakefile` is just Ruby: `task` is a plain method call that registers a
named block you can invoke from the command line as `rake <name>`.

## A first Rakefile

```ruby
# Rakefile
task :greet do
  puts "Hello from Rake!"
end
```

```text
$ rake greet
Hello from Rake!
```

`rake -T` lists every task that has a `desc` above it (undocumented tasks
run fine but are hidden from this summary — a convention for keeping
internal/helper tasks out of the public task list).

## Task dependencies

A task can depend on other tasks, which run first, in declaration order,
each exactly once even if depended on from multiple places:

```ruby
task :build => :greet do
  puts "Building..."
end
```

```text
$ rake build
Hello from Rake!
Building...
```

`:greet` runs to completion before `:build`'s own block starts. This is
how `rake assets:precompile` style multi-step pipelines are put
together — small tasks composed via dependencies instead of one giant
method.

## Namespaces

`namespace` groups related tasks under a prefix, exactly like a module
groups classes:

```ruby
namespace :db do
  task :migrate do
    puts "Running migrations..."
  end

  task :seed => :migrate do
    puts "Seeding data..."
  end
end
```

```text
$ rake db:seed
Running migrations...
Seeding data...
```

`db:seed` depends on `db:migrate` *within the same namespace* — you refer
to it as plain `:migrate` inside the namespace block, and Rake resolves
it to `db:migrate` automatically.

## File tasks — only rebuild when needed

Regular `task` always runs its block. `file` tasks are Rake's answer to
Make's original purpose: only run if the target doesn't exist yet, or is
older than its prerequisite:

```ruby
file "output.txt" => "input.txt" do
  content = File.read("input.txt")
  File.write("output.txt", content.upcase)
  puts "Generated output.txt"
end
```

```text
$ rake output.txt
Generated output.txt

$ cat output.txt
HELLO WORLD
```

Running `rake output.txt` again immediately does nothing and prints
nothing, because `output.txt` now exists and is newer than
`input.txt` — Rake compares file modification times and skips work it
doesn't need to redo. Touch `input.txt` (or edit it) and `output.txt`
becomes stale again, triggering a rebuild on the next run.

## desc — documenting tasks

```ruby
desc "Say goodbye"
task :bye do
  puts "Goodbye!"
end
```

```text
$ rake -T
rake bye  # Say goodbye
```

Only `:bye` shows up because it's the only task with a `desc` line
directly above it in this Rakefile — `:greet`, `:build`, and the `db`
namespace tasks above were left undocumented on purpose to demonstrate
the difference.

## Tasks that take arguments

```ruby
task :greet_name, [:name] do |t, args|
  puts "Hello, #{args[:name] || 'stranger'}!"
end
```

```text
$ rake "greet_name[Ruby]"
Hello, Ruby!
```

The quotes around the whole invocation matter in most shells — `[` and
`]` are glob-special characters that the shell would otherwise try to
expand.

## Rake-specific traps

- **Task blocks always run when invoked directly via `task`**, even if
  nothing actually changed — only `file` tasks get the "skip if
  up-to-date" behavior. Using plain `task` for something that's
  expensive to redo (like a full data import) wastes time on every
  invocation.
- **A dependency runs once per `rake` invocation, not once ever.**
  `rake a b` where both `a` and `b` depend on `c` runs `c` exactly once,
  but running `rake a` and then `rake b` separately runs `c` twice — Rake
  doesn't remember completed work across separate process invocations.
- **Namespaced task dependencies need the full path from outside the
  namespace.** `task :other => "db:migrate"` (as a string, with the
  namespace prefix) from outside `namespace :db do ... end`, versus the
  bare `:migrate` symbol used *inside* the namespace block.
- **Rakefiles execute top-to-bottom at load time**, before any task
  runs — a `puts` at the top level of a Rakefile (outside any task
  block) prints on every single `rake` invocation, including `rake -T`,
  which surprises people expecting task-scoped output only.
- **`file` tasks compare modification time, not content.** Touching a
  file without changing its content (`touch input.txt`) still counts as
  "newer" and triggers a rebuild — useful to know when a build seems to
  rerun for no visible reason.

## Cheat sheet

| Task | Rake code |
|---|---|
| Define a task | `task :name do ... end` |
| Document a task | `desc "..."` above the `task` line |
| Depend on another task | `task :b => :a do ... end` |
| Multiple dependencies | `task :c => [:a, :b]` |
| Group under a namespace | `namespace :db do ... end` |
| Reference inside the namespace | `task :seed => :migrate` |
| Reference from outside | `task :other => "db:migrate"` |
| Rebuild only when stale | `file "out" => "in" do ... end` |
| Accept CLI arguments | `task :t, [:arg] do \|t, args\| ... end` |
| List documented tasks | `rake -T` |
| Set a default task | `task :default => :spec` |

## Exercise

Build a `Rakefile` for a small project:

1. A `:clean` task that removes any `*.log` file in the current
   directory (use `FileList["*.log"].each { \|f\| File.delete(f) }`).
2. A `namespace :report do ... end` with a `generate` task depending on
   `clean` that writes a `report.log` file with today's date, and a
   `show` task that prints the contents of `report.log` (or a friendly
   message if it doesn't exist yet).
3. A `desc`-documented default task (`task :default => "report:generate"`)
   so plain `rake` with no arguments runs the whole pipeline.
4. Run `rake -T`, then `rake`, then `rake report:show`, and paste the
   output of each.
