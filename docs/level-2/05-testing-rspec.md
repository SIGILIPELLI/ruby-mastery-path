# 05 · Testing with RSpec

RSpec is the most widely used testing framework in the Ruby ecosystem. It
lets you describe your code's expected *behavior* in near-English sentences
that double as living documentation — and, more practically, it's what
catches regressions before they ship. This lesson covers the core DSL:
`describe`/`context`/`it`, matchers, and setup/teardown hooks.

## Installing RSpec

```bash
gem install rspec
# or, in a project with a Gemfile:
bundle add rspec --group test
bundle exec rspec --init   # generates spec/spec_helper.rb and .rspec
```

RSpec expects spec files under `spec/`, named `*_spec.rb`, mirroring the
file they test — a class in `lib/calculator.rb` gets tests in
`spec/calculator_spec.rb`.

## The code under test

```ruby
# lib/calculator.rb
class Calculator
  def add(a, b)
    a + b
  end

  def divide(a, b)
    raise ZeroDivisionError, "Cannot divide by zero" if b.zero?

    a / b.to_f
  end
end
```

## describe, it, and expect — the basic shape

```ruby
# spec/calculator_spec.rb
require_relative "../lib/calculator"

RSpec.describe Calculator do
  it "adds two numbers" do
    calculator = Calculator.new
    result = calculator.add(2, 3)
    expect(result).to eq(5)
  end
end
```

- **`describe`** groups related examples — usually named after the class
  or method being tested.
- **`it`** defines one example ("spec"); its string argument describes the
  expected behavior in plain language.
- **`expect(actual).to matcher(expected)`** is the assertion — RSpec's
  modern syntax, replacing the older `should` syntax you may see in legacy
  code.

Run it with:

```bash
bundle exec rspec spec/calculator_spec.rb
```

```text
Calculator
  adds two numbers

Finished in 0.002 seconds
1 example, 0 failures
```

## context — grouping examples by scenario

`context` is functionally identical to `describe`, but conventionally used
to separate different scenarios or states ("when X" / "with Y"):

```ruby
RSpec.describe Calculator do
  let(:calculator) { Calculator.new }

  describe "#divide" do
    context "when the divisor is not zero" do
      it "returns the quotient as a float" do
        expect(calculator.divide(10, 4)).to eq(2.5)
      end
    end

    context "when the divisor is zero" do
      it "raises a ZeroDivisionError" do
        expect { calculator.divide(10, 0) }.to raise_error(ZeroDivisionError)
      end
    end
  end
end
```

Note `expect { ... }.to raise_error(...)` uses **curly braces**, not
parentheses — the block form is required whenever you're asserting on
*behavior* (raising, changing state) rather than a plain return value.

## let — lazy, memoized test data

`let(:name) { ... }` defines a helper method available in every example
inside that `describe`/`context` block. The block only runs the first time
`name` is called within a given example, and its result is memoized for
the rest of that example — avoiding both repeated setup code and unwanted
shared state between examples:

```ruby
RSpec.describe Calculator do
  let(:calculator) { Calculator.new }   # fresh instance per example

  it "adds" do
    expect(calculator.add(1, 1)).to eq(2)
  end

  it "is a different instance in every example" do
    expect(calculator.add(2, 2)).to eq(4)   # `calculator` here is NOT the same object as above
  end
end
```

## before and after hooks

`before` runs before every example in its scope — commonly used for setup
that several examples need. `after` runs afterward, typically for cleanup:

```ruby
RSpec.describe "File-based cache" do
  before do
    @cache_path = "test_cache.tmp"
    File.write(@cache_path, "cached-value")
  end

  after do
    File.delete(@cache_path) if File.exist?(@cache_path)
  end

  it "reads back what was cached" do
    expect(File.read(@cache_path)).to eq("cached-value")
  end
end
```

Prefer `let` over instance variables in `before` blocks when possible —
`let` is lazy (only computed if used) and reads more clearly, but `before`
is still the right tool for side effects like writing a file or seeding a
database, as shown above.

## Common matchers

```ruby
expect(5).to eq(5)                       # equality (==)
expect(5).to be(5)                          # identity (equal?) -- careful with objects, fine for integers
expect([1, 2, 3]).to include(2)
expect("hello world").to match(/wor.d/)
expect([]).to be_empty
expect(nil).to be_nil
expect(3.14).to be_within(0.01).of(3.15)   # float comparisons -- never use eq with floats
expect { raise "boom" }.to raise_error("boom")
expect(5).not_to eq(6)
```

Floating-point results (like `divide` above) should almost always use
`be_within(...).of(...)` rather than `eq`, since floating-point arithmetic
can produce results like `2.5000000000000004` that fail exact equality.

## Testing collaborating objects — doubles and stubs

RSpec can create fake objects ("doubles") that stand in for real
collaborators, so a unit test doesn't depend on a slow or unreliable real
dependency:

```ruby
RSpec.describe "a class that logs" do
  it "calls the logger with the right message" do
    logger = double("Logger")
    expect(logger).to receive(:info).with("Started")

    logger.info("Started")   # in real code, this call would happen inside the class under test
  end
end
```

`expect(logger).to receive(:info).with("Started")` is a **message
expectation** — the example fails if `logger.info("Started")` is never
called with exactly that argument.

## Cheat sheet

| RSpec construct | Purpose |
|---|---|
| `describe SomeClass do ... end` | group examples for a class/feature |
| `context "when ..." do ... end` | group examples by scenario |
| `it "does something" do ... end` | one test example |
| `let(:name) { ... }` | lazy, memoized helper value |
| `before` / `after` | setup / teardown hooks |
| `expect(x).to eq(y)` | value equality |
| `expect { ... }.to raise_error(Klass)` | assert an exception is raised |
| `double("Name")` | a fake stand-in object |

## Exercise

Write a `StringUtils` class with a method `palindrome?(text)` that returns
`true` if `text` reads the same forwards and backwards (ignore case and
non-letter characters). Then write `spec/string_utils_spec.rb` covering at
least three `context` blocks: a true palindrome (`"racecar"`), a phrase
palindrome (`"A man a plan a canal Panama"`), and a non-palindrome
(`"hello"`) — using `let` for the `StringUtils` instance.
