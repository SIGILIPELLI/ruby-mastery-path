# 04 · File I/O

Almost every real program eventually reads configuration, writes logs, or
processes data files. Ruby's `File` and `Dir` classes wrap the operating
system's file APIs in a friendly, block-based interface — and, as with
exceptions, using blocks correctly here is what keeps your program from
leaking open file handles.

## Writing a file

```ruby
File.write("notes.txt", "First line\nSecond line\n")
```

`File.write` is the simplest way to write a string to a file in one shot —
it opens the file, writes, and closes it for you. It overwrites the file if
it already exists.

## Reading a whole file at once

```ruby
contents = File.read("notes.txt")
puts contents
# First line
# Second line

lines = File.readlines("notes.txt")
puts lines.inspect
# ["First line\n", "Second line\n"]
```

`File.readlines` keeps the trailing `"\n"` on each line — use `.chomp` (or
`.strip`) if you don't want it: `File.readlines("notes.txt").map(&:chomp)`.

## The File.open block form — why it matters

`File.read`/`File.write` are convenient for one-shot use, but for anything
more involved, open the file explicitly with a block:

```ruby
File.open("notes.txt", "r") do |file|
  file.each_line do |line|
    puts "> #{line.chomp}"
  end
end
# > First line
# > Second line
```

The block form **automatically closes the file** when the block ends, even
if an exception is raised inside it — it's `File.open` wrapped in the
equivalent of a `begin/ensure`. Opening a file *without* a block
(`file = File.open(...)`) means you're responsible for calling
`file.close` yourself, and forgetting to do so leaks file descriptors —
a real problem in long-running programs that open many files.

```ruby
# Risky: if something raises between open and close, the file never closes
file = File.open("notes.txt", "r")
data = file.read
file.close   # skipped entirely if `file.read` had raised

# Safer: always use the block form when you can
File.open("notes.txt", "r") { |f| f.read }
```

## File modes

| Mode | Meaning |
|---|---|
| `"r"` | read only (default), errors if the file doesn't exist |
| `"w"` | write only, truncates (empties) the file first, creates it if missing |
| `"a"` | append only, creates the file if missing, never truncates |
| `"r+"` | read and write, file must already exist |
| `"a+"` | read and append |

```ruby
File.open("log.txt", "a") do |file|
  file.puts "Started at #{Time.now}"
end

File.open("log.txt", "a") do |file|
  file.puts "Finished at #{Time.now}"
end

puts File.read("log.txt").lines.count   # 2 -- "a" mode never overwrote the first line
```

## Streaming large files line-by-line

Loading a huge file with `File.read` pulls the whole thing into memory at
once. `File.foreach` (or `each_line` on an open file) streams it one line
at a time instead:

```ruby
File.write("numbers.txt", (1..5).map { |n| n * n }.join("\n"))

total = 0
File.foreach("numbers.txt") do |line|
  total += line.to_i
end
puts total   # 55
```

For a file with a million lines, `File.foreach` holds only the current
line in memory, while `File.read.each_line` would first materialize the
entire file as one giant string.

## Checking before you touch the filesystem

```ruby
path = "maybe_missing.txt"

if File.exist?(path)
  puts File.read(path)
else
  puts "#{path} does not exist"
end

puts File.exist?("notes.txt")     # true
puts File.directory?(".")           # true
puts File.size("notes.txt")          # size in bytes, e.g. 23
puts File.extname("notes.txt")      # .txt
puts File.basename("/a/b/notes.txt")   # notes.txt
puts File.dirname("/a/b/notes.txt")     # /a/b
```

`File.exist?` avoids raising `Errno::ENOENT` for the common case where a
missing file is an expected possibility, not a bug — reserve
`rescue Errno::ENOENT` (see
[Exception Handling](03-exception-handling.md)) for cases where the
missing-file check itself would introduce a race condition.

## The Dir class — working with directories

```ruby
Dir.mkdir("reports") unless Dir.exist?("reports")

File.write("reports/january.csv", "data")
File.write("reports/february.csv", "data")

puts Dir.entries("reports").sort.inspect
# [".", "..", "february.csv", "january.csv"]

puts Dir.glob("reports/*.csv").sort.inspect
# ["reports/february.csv", "reports/january.csv"]

puts Dir.children("reports").sort.inspect
# ["february.csv", "january.csv"]
```

`Dir.glob` accepts shell-style patterns (`*`, `**`, `?`) and is the
idiomatic way to find files matching a pattern — far more common in real
code than manually filtering `Dir.entries`.

## CSV and structured text — a quick preview

Ruby's standard library also ships a `csv` module for structured data,
which is usually a better fit than hand-parsing comma-separated lines:

```ruby
require "csv"

CSV.open("people.csv", "w") do |csv|
  csv << ["name", "age"]
  csv << ["Ada", 30]
  csv << ["Grace", 34]
end

CSV.foreach("people.csv", headers: true) do |row|
  puts "#{row['name']} is #{row['age']}"
end
# Ada is 30
# Grace is 34
```

## Cheat sheet

| Task | Code |
|---|---|
| Write a string to a file | `File.write(path, content)` |
| Read entire file | `File.read(path)` |
| Read as an array of lines | `File.readlines(path)` |
| Stream line-by-line (memory efficient) | `File.foreach(path) { \|line\| ... }` |
| Open safely (auto-closes) | `File.open(path, mode) { \|f\| ... }` |
| Check existence | `File.exist?(path)` |
| List files matching a pattern | `Dir.glob("*.txt")` |
| Make a directory | `Dir.mkdir(path)` |

## Exercise

Write a method `word_count(path)` that streams a text file with
`File.foreach` (don't load it all with `File.read`) and returns a hash of
`{ "word" => count }` for every word in the file (split on whitespace,
downcase each word). Then write a method `archive_logs(dir)` that uses
`Dir.glob` to find every `*.log` file in `dir`, and appends each one's
contents into a single `archive.txt` using `File.open(..., "a")`.
