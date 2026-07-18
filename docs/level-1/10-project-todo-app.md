# 10 · Project — CLI To-Do App

A small end-to-end project combining everything from Level 1: classes,
methods, arrays/hashes, string interpolation, blocks, and file persistence.

## What you'll build

A command-line to-do list that:

- Adds tasks
- Lists tasks (with done/pending status)
- Marks tasks done
- Deletes tasks
- Persists everything to a JSON file between runs

## Project layout

```text
todo_app/
    storage.rb
    todo.rb
```

## storage.rb — persistence layer

```ruby
# storage.rb
require "json"

class TaskStorage
  DB_PATH = "tasks.json"

  def self.load
    return [] unless File.exist?(DB_PATH)

    JSON.parse(File.read(DB_PATH), symbolize_names: true)
  rescue JSON::ParserError
    []
  end

  def self.save(tasks)
    File.write(DB_PATH, JSON.pretty_generate(tasks))
  end
end
```

## todo.rb — CLI logic

```ruby
# todo.rb
require_relative "storage"

def add_task(tasks, description)
  tasks << { description: description, done: false }
  puts "Added: #{description}"
end

def list_tasks(tasks)
  if tasks.empty?
    puts "No tasks yet."
    return
  end

  tasks.each_with_index do |task, i|
    status = task[:done] ? "x" : " "
    puts "[#{status}] #{i + 1}. #{task[:description]}"
  end
end

def complete_task(tasks, index)
  task = tasks[index - 1]
  if task.nil?
    puts "No task with number #{index}"
    return
  end
  task[:done] = true
  puts "Marked task #{index} done."
end

def delete_task(tasks, index)
  if index < 1 || index > tasks.length
    puts "No task with number #{index}"
    return
  end
  removed = tasks.delete_at(index - 1)
  puts "Deleted: #{removed[:description]}"
end

tasks = TaskStorage.load
command, *rest = ARGV

if command.nil?
  puts "Usage: ruby todo.rb [add <text> | list | done <n> | delete <n>]"
  exit
end

case command
when "add"
  add_task(tasks, rest.join(" "))
  TaskStorage.save(tasks)
when "list"
  list_tasks(tasks)
when "done"
  complete_task(tasks, rest[0].to_i)
  TaskStorage.save(tasks)
when "delete"
  delete_task(tasks, rest[0].to_i)
  TaskStorage.save(tasks)
else
  puts "Unknown command: #{command}"
end
```

## Running it

```bash
ruby todo.rb add "Write Level 1 exercises"
ruby todo.rb add "Review Level 2 outline"
ruby todo.rb list
# [ ] 1. Write Level 1 exercises
# [ ] 2. Review Level 2 outline

ruby todo.rb done 1
ruby todo.rb list
# [x] 1. Write Level 1 exercises
# [ ] 2. Review Level 2 outline
```

## Stretch goals

- Add a `priority` field (`"low"`/`"medium"`/`"high"`) and sort the list by
  it before printing.
- Add due dates using Ruby's `Date` class, and highlight overdue tasks.
- Add an RSpec test for `TaskStorage` (you'll formalize this properly with
  RSpec in [Level 2](../level-2/05-testing-rspec.md)).

Completing this project means you're ready for **Level 2 · Intermediate**.
