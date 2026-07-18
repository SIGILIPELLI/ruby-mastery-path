# 09 · Gems & Bundler Basics

A **gem** is a Ruby package — a library someone else wrote that you can
install and use in your own code. **Bundler** manages exactly which gems (and
versions) a project depends on.

## Installing a gem directly

```bash
gem install colorize
```

```ruby
require "colorize"

puts "Success!".green
puts "Warning!".yellow
puts "Error!".red
```

`require` loads a library — either from Ruby's standard library (like
`json` or `date`) or from an installed gem.

## Why Bundler exists

Installing gems globally with `gem install` works for quick experiments, but
real projects need **reproducible** dependencies — the exact same gem
versions on every machine that runs the code. Bundler solves this with a
`Gemfile` and a `Gemfile.lock`.

## Gemfile

```ruby
# Gemfile
source "https://rubygems.org"

gem "colorize"
gem "httparty", "~> 0.21"     # pessimistic version constraint
gem "rspec", group: :test       # only installed in the :test group
```

```bash
bundle install
```

Running `bundle install` reads the `Gemfile`, resolves compatible versions
for every gem, installs them, and writes the exact resolved versions to
`Gemfile.lock` — that lock file is what guarantees everyone on the team (and
your production server) gets identical versions.

## Version constraint syntax

| Syntax | Meaning |
|--------|---------|
| `gem "foo"` | Any version |
| `gem "foo", "1.2.3"` | Exactly this version |
| `gem "foo", ">= 1.2"` | This version or newer |
| `gem "foo", "~> 1.2"` | `>= 1.2`, `< 2.0` (pessimistic — allows patch/minor updates only) |

`~>` ("twiddle-wakka") is the most common in practice — it allows safe patch
and minor updates while protecting against unexpected breaking major-version
bumps.

## Running code with Bundler

```bash
bundle exec ruby my_script.rb
```

`bundle exec` ensures your script uses the exact gem versions locked in
`Gemfile.lock`, rather than whatever happens to be installed globally on the
system — always prefer this over a bare `ruby` command once a `Gemfile`
exists.

## Requiring gems in your code

```ruby
# my_script.rb
require "bundler/setup"   # restricts loading to only Gemfile-listed gems
require "colorize"
require "httparty"

puts "Ready!".green

response = HTTParty.get("https://api.github.com")
puts response.code
```

## Gem groups (e.g., test-only dependencies)

```ruby
# Gemfile
gem "rails"

group :test do
  gem "rspec"
  gem "factory_bot"
end
```

```bash
bundle install --without test   # skip the :test group, e.g. in production
```

## Cheat sheet

| Task | Command |
|------|---------|
| Install a gem globally | `gem install name` |
| Install project dependencies | `bundle install` |
| Run a script with locked versions | `bundle exec ruby script.rb` |
| Add a gem | Add a line to `Gemfile`, then `bundle install` |
| Pessimistic version constraint | `gem "name", "~> 1.2"` |
| Group gems (e.g. test-only) | `group :test do ... end` |

## Exercise

Create a `Gemfile` requiring the `json` gem (part of the standard library,
but a good practice example) with a `~>` version constraint. Run `bundle
install` to generate `Gemfile.lock`. Then write a script that uses
`require "json"` to parse a JSON string like `'{"name": "Ada", "age": 30}'`
into a Ruby hash and print each key/value pair.
