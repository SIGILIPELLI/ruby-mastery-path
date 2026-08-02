# 08 · Gem Development Basics

A **gem** is Ruby's unit of packaged, shareable code — everything you
`bundle add` or `gem install` (RSpec, JSON, Rails itself) is a gem. Once
you've written something reusable, packaging it as a gem is what lets you
`require` it from other projects, or share it with the world via
[RubyGems.org](https://rubygems.org). This lesson builds a minimal, real
gem from scratch.

## Generating the skeleton

Bundler can scaffold a new gem's directory structure for you:

```bash
bundle gem greeter_kit
```

That produces roughly this layout (a few generated files trimmed for
clarity):

```text
greeter_kit/
    lib/
        greeter_kit.rb
        greeter_kit/
            version.rb
    spec/
        greeter_kit_spec.rb
        spec_helper.rb
    greeter_kit.gemspec
    Gemfile
    README.md
    Rakefile
```

## The gemspec — a gem's manifest

The `.gemspec` file describes the gem's metadata: name, version, files
included, and dependencies. This is what RubyGems reads when you publish:

```ruby
# greeter_kit.gemspec
require_relative "lib/greeter_kit/version"

Gem::Specification.new do |spec|
  spec.name        = "greeter_kit"
  spec.version     = GreeterKit::VERSION
  spec.authors     = ["Your Name"]
  spec.email       = ["you@example.com"]
  spec.summary     = "Small, friendly greeting utilities."
  spec.description = "Generate personalized greetings in several styles."
  spec.homepage    = "https://github.com/you/greeter_kit"
  spec.license     = "MIT"

  spec.files       = Dir["lib/**/*.rb"] + ["README.md", "LICENSE.txt"]
  spec.require_paths = ["lib"]

  spec.required_ruby_version = ">= 2.7.0"

  spec.add_development_dependency "rspec", "~> 3.12"
end
```

`spec.files` controls exactly what ships inside the built `.gem` file —
listing it explicitly (rather than relying on `git ls-files`, which some
generators use by default) keeps the published package free of test
fixtures, CI config, and other development-only cruft.

## The version file — semantic versioning

```ruby
# lib/greeter_kit/version.rb
module GreeterKit
  VERSION = "0.1.0"
end
```

Ruby gems follow [semantic versioning](https://semver.org):
`MAJOR.MINOR.PATCH`. Bump `PATCH` for bug fixes, `MINOR` for new
backward-compatible features, and `MAJOR` for breaking changes — anyone
depending on your gem uses this number to decide whether upgrading is
safe.

## The library code

```ruby
# lib/greeter_kit.rb
require_relative "greeter_kit/version"

module GreeterKit
  class Error < StandardError; end

  class Greeter
    STYLES = {
      formal: ->(name) { "Good day, #{name}." },
      casual: ->(name) { "Hey #{name}!" },
      excited: ->(name) { "#{name}!!! So great to see you!!!" },
    }.freeze

    def initialize(style: :casual)
      raise Error, "Unknown style: #{style}" unless STYLES.key?(style)

      @style = style
    end

    def greet(name)
      STYLES[@style].call(name)
    end
  end
end
```

Note the top-level `module GreeterKit` — this is the gem's **namespace**.
Everything the gem defines should live under it, so `GreeterKit::Greeter`
and `GreeterKit::Error` never collide with a class of the same name from a
different gem someone else has installed.

## Testing the gem

```ruby
# spec/greeter_kit_spec.rb
require "greeter_kit"

RSpec.describe GreeterKit::Greeter do
  it "greets casually by default" do
    greeter = GreeterKit::Greeter.new
    expect(greeter.greet("Ada")).to eq("Hey Ada!")
  end

  it "greets formally when asked" do
    greeter = GreeterKit::Greeter.new(style: :formal)
    expect(greeter.greet("Ada")).to eq("Good day, Ada.")
  end

  it "raises on an unknown style" do
    expect { GreeterKit::Greeter.new(style: :sarcastic) }
      .to raise_error(GreeterKit::Error, /Unknown style/)
  end
end
```

Run the suite with `bundle exec rspec` (see
[Testing with RSpec](05-testing-rspec.md) for the full `describe`/`it`/
matcher rundown).

## Building and installing locally

```bash
gem build greeter_kit.gemspec
# Successfully built RubyGem
# Name: greeter_kit
# Version: 0.1.0
# File: greeter_kit-0.1.0.gem

gem install ./greeter_kit-0.1.0.gem
```

```ruby
require "greeter_kit"
puts GreeterKit::Greeter.new(style: :excited).greet("World")
# World!!! So great to see you!!!
```

Building and installing locally before publishing lets you sanity-check
that `spec.files` actually includes everything needed — a surprisingly
common first-publish bug is a gem that works from the source directory but
breaks once installed, because a file was left out of the gemspec.

## Publishing to RubyGems.org

```bash
gem signin      # one-time, links this machine to your RubyGems.org account
gem push greeter_kit-0.1.0.gem
```

Once pushed, anyone can `gem install greeter_kit` or add
`gem "greeter_kit"` to a `Gemfile`. Publishing is **not easily reversible**
— a version, once pushed, can be "yanked" but the version number can never
be reused — so bump the version and re-push rather than trying to
overwrite a mistake.

## Cheat sheet

| Task | Command / File |
|---|---|
| Scaffold a new gem | `bundle gem my_gem` |
| Gem metadata | `my_gem.gemspec` |
| Namespace convention | `module MyGem; ... end` around all code |
| Build a `.gem` file | `gem build my_gem.gemspec` |
| Install a local build | `gem install ./my_gem-VERSION.gem` |
| Publish publicly | `gem push my_gem-VERSION.gem` |
| Versioning scheme | semantic versioning: `MAJOR.MINOR.PATCH` |

## Exercise

Scaffold a gem called `word_stats` with `bundle gem word_stats`. Give it a
`WordStats.analyze(text)` class method that returns a hash with
`:word_count`, `:char_count` (excluding whitespace), and `:longest_word`.
Write at least three RSpec examples for it, fill in the gemspec's summary
and description, and build it locally with `gem build` to confirm it
packages cleanly — you don't need to actually publish it.
