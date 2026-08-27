# 04 · Testing at Scale & CI

A suite of 20 specs and one of 2,000 specs need different organizational
tools. Shared examples avoid duplicating the same assertions across
similar classes, tags let you run a subset without touching every file,
and continuous integration makes "did I break anything" an automated
question instead of a manual one.

## Shared examples — testing the same contract across classes

When multiple classes are supposed to satisfy the same behavior (several
implementations of a "stack" interface, several notifier classes that
must all respond to `.send`), `shared_examples` lets you write the
assertions once:

```ruby
RSpec.shared_examples "a stack" do
  it "starts empty" do
    expect(subject.empty?).to eq(true)
  end

  it "pushes and pops in LIFO order" do
    subject.push(1)
    subject.push(2)
    expect(subject.pop).to eq(2)
  end
end

class ArrayStack
  def initialize; @items = []; end
  def empty?; @items.empty?; end
  def push(x); @items.push(x); end
  def pop; @items.pop; end
end

RSpec.describe ArrayStack do
  subject { ArrayStack.new }
  it_behaves_like "a stack"
end
```

Captured output:

```text
..

Finished in 0.00208 seconds (files took 0.04338 seconds to load)
2 examples, 0 failures
```

A second implementation (say `LinkedListStack`) reuses the exact same
`it_behaves_like "a stack"` line — the contract is defined once, and
every implementation claiming to satisfy it gets tested against the same
expectations. This catches a common bug class: an interface's
implementations quietly drifting apart because each one only has its own
bespoke tests.

## Tags — running a meaningful subset

```ruby
RSpec.describe "tagged examples" do
  it "runs a fast unit test", :unit do
    expect(1 + 1).to eq(2)
  end

  it "runs a slow integration test", :slow do
    expect(true).to eq(true)
  end
end
```

```text
$ rspec --tag unit
Run options: include {unit: true}
.

Finished in 0.00064 seconds (files took 0.06764 seconds to load)
1 example, 0 failures
```

`--tag unit` runs only examples tagged `:unit`, skipping the `:slow` one
entirely. In a large suite, this is how you get a fast local feedback
loop (`rspec --tag unit`) distinct from the full CI run (`rspec`, no
filter, everything including slow integration/request specs) — you
don't wait for the slow suite on every save, but CI still runs
everything before merge.

## Organizing a large suite

- **`spec/models/`, `spec/requests/`, `spec/services/`** mirroring your
  `lib`/`app` structure — a spec's location tells you what it's testing
  without opening the file.
- **`spec_helper.rb` vs `rails_helper.rb`** (or an equivalent split in a
  non-Rails app): keep a lean helper for pure unit specs (no database,
  fast to load) separate from one that boots the full app/database stack
  — so a quick unit-only run doesn't pay full boot cost.
- **`before(:suite)`** for one-time expensive setup (seeding reference
  data) versus **`before(:each)`** for per-example state that must not
  leak between examples — using `before(:each)` for something that could
  be `before(:suite)` slows down every single example for no reason.
- **Factories over fixtures** — a factory (via `factory_bot` or a
  hand-rolled builder method) generates exactly the data one test needs
  inline, rather than a large shared fixture file every test implicitly
  depends on and can silently break by editing.

## A minimal CI config (GitHub Actions)

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.3'
          bundler-cache: true
      - name: Run tests
        run: bundle exec rspec
      - name: Run RuboCop
        run: bundle exec rubocop
```

`bundler-cache: true` caches installed gems between runs so CI doesn't
reinstall the whole dependency tree on every push. Running RuboCop
alongside RSpec in the same job means a style violation fails the build
exactly like a failing test does — style debt doesn't silently
accumulate.

## Splitting slow suites in CI

Once a suite takes several minutes, most CI systems support running
specs across parallel jobs (a "test matrix" splitting spec files by
count or historical timing) — e.g. `knapsack` or GitHub Actions' matrix
strategy running 4 shards concurrently, each running roughly a quarter
of the suite, cutting wall-clock CI time by close to 4x without changing
a single test.

## Testing-at-scale-specific traps

- **Shared examples with hidden dependencies on `let`/`subject` names.**
  A shared example referencing `subject` implicitly requires every
  consumer to define one — if `LinkedListStack`'s spec forgets to define
  `subject`, RSpec raises `NotImplementedError` from inside the *shared*
  example, at a location that doesn't obviously point back to the
  missing definition.
- **Global state leaking between examples** (a class variable, a
  singleton, `Time.now` stubbing left in place) causes intermittent,
  order-dependent failures that only reproduce when the full suite runs
  in a specific order — `rspec --seed <N>` re-running with the same
  random order that failed is the standard way to reproduce these
  reliably.
- **A green CI badge that only means "the last push passed."** Flaky
  tests re-run until they pass mask real, intermittent bugs — treat a
  flaky test as a bug in the test (or in the code under test) to fix, not
  something to silently retry away.
- **Tagging everything `:slow` "to be safe"** defeats the purpose of
  tags — the fast/slow split only helps if most specs genuinely stay in
  the fast tier; audit tag usage periodically as the suite grows.
- **CI caching a stale `Gemfile.lock`.** A cache key that doesn't include
  a hash of `Gemfile.lock` can silently keep using old gem versions after
  a real dependency bump, hiding a real incompatibility until it breaks
  in production instead of in CI.

## Cheat sheet

| Task | Syntax |
|---|---|
| Define a reusable contract | `RSpec.shared_examples "name" do ... end` |
| Use it in a spec | `it_behaves_like "name"` |
| Tag an example | `it "...", :slow do ... end` |
| Run only tagged examples | `rspec --tag slow` |
| Exclude tagged examples | `rspec --tag ~slow` |
| One-time expensive setup | `before(:suite) { ... }` |
| Reproduce a flaky failure order | `rspec --seed 1234` |
| Fail fast on first failure | `rspec --fail-fast` |

## Exercise

1. Write `shared_examples "a queue"` covering enqueue/dequeue FIFO
   order, then implement two classes (`ArrayQueue` backed by an array,
   `LinkedQueue` backed by two stacks or a simple linked structure) and
   run the shared example against both.
2. Tag half your specs `:fast` and half `:slow` (arbitrarily, for
   practice), then run `rspec --tag fast` and `rspec --tag ~slow` and
   confirm they select the expected, disjoint example sets.
3. Write a minimal `.github/workflows/ci.yml` for a project with a
   `Gemfile` — include gem caching, running `bundle exec rspec`, and a
   separate matrix strategy step splitting specs across 2 parallel jobs
   by filename glob (`spec/models/*_spec.rb` vs `spec/requests/*_spec.rb`).
