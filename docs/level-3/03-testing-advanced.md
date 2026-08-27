# 03 · Testing Advanced

Level 2 covered RSpec basics — `describe`, `it`, `expect(...).to eq(...)`.
Real test suites also need to isolate a unit from its collaborators
(mocks/stubs/doubles) and to exercise a whole HTTP layer end-to-end
(request specs). Both rely on the same RSpec you already know; this
module adds the vocabulary for talking to fakes instead of the real
thing.

## Why fake collaborators at all

Suppose `Notifier` sends a welcome email through some `mailer` object:

```ruby
class Notifier
  def initialize(mailer)
    @mailer = mailer
  end

  def notify(user)
    @mailer.send_email(user, "Welcome!")
  end
end
```

Testing this for real would mean actually sending email in your test
suite — slow, flaky, and annoying to run 500 times a day. Instead, you
hand `Notifier` a **double**: a fake object that stands in for the real
mailer and only needs to respond the way this one test cares about.

## Doubles and message expectations

```ruby
RSpec.describe Notifier do
  it "calls send_email on the mailer with a double" do
    mailer = double("mailer")
    expect(mailer).to receive(:send_email).with("alice", "Welcome!")
    Notifier.new(mailer).notify("alice")
  end
end
```

`double("mailer")` creates an object that responds to nothing by default.
`expect(mailer).to receive(:send_email)` declares, before anything runs,
that `send_email` **must** be called with exactly those arguments — the
test fails if it's never called, or called with different arguments.
This is different from a normal `expect(...).to eq(...)`: it's a promise
about behavior, checked after the example runs.

## Stubbing return values

```ruby
it "stubs a return value" do
  mailer = double("mailer", send_email: "sent!")
  result = Notifier.new(mailer).notify("bob")
  expect(result).to eq("sent!")
end
```

`double("mailer", send_email: "sent!")` is shorthand for a double that
responds to `send_email` and always returns `"sent!"`, with no
expectation about whether or how many times it's called — a **stub**,
not an **expectation**. Use a stub when you only care about the return
value; use `expect(...).to receive` when the call itself is the thing
under test.

## instance_double — verified doubles

Plain `double` will happily let you stub a method that doesn't exist on
the real object — a typo like `send_emial` fails silently at test time
and only breaks in production. `instance_double` checks the method
actually exists (and its arity) on the named class:

```ruby
it "spies on a real object" do
  mailer = instance_double("RealMailer", send_email: true)
  Notifier.new(mailer).notify("carol")
  expect(mailer).to have_received(:send_email).with("carol", "Welcome!")
end
```

`have_received` is the "spy" style: let the stub record calls, then
assert afterward instead of declaring the expectation up front. Prefer
`instance_double` over plain `double` whenever the real class is defined
somewhere in your codebase — it catches drift between your test's fake
and the real interface.

Captured output running all three examples together:

```text
$ rspec mocks_spec.rb
...

Finished in 0.00937 seconds (files took 0.05666 seconds to load)
3 examples, 0 failures
```

## Request specs — testing a whole route

A request spec exercises your app the way an HTTP client would: send a
request, assert on the response. Sinatra apps use `rack-test` for this
(Rails calls the equivalent an integration/request spec built on the same
idea):

```ruby
require 'rack/test'
require_relative 'app'

RSpec.describe "Notes API", type: :request do
  include Rack::Test::Methods
  def app; Sinatra::Application; end

  it "GET / returns greeting" do
    get '/'
    expect(last_response.status).to eq(200)
    expect(last_response.body).to eq("Hello, Sinatra!")
  end

  it "GET /greet/:name interpolates the name" do
    get '/greet/Ruby'
    expect(last_response.body).to eq("Hello, Ruby!")
  end
end
```

Captured output:

```text
$ rspec request_spec.rb
..

Finished in 0.00846 seconds (files took 0.22293 seconds to load)
2 examples, 0 failures
```

Request specs don't know or care how a route is implemented internally —
only what goes in (the request) and what comes out (status, headers,
body). That makes them resilient to refactoring the route's internals,
unlike a unit test that mocks internal collaborators.

## Unit vs. request spec — when to use which

- **Unit spec + doubles**: testing one class's logic in isolation,
  fast, pinpoints exactly which class broke.
- **Request spec**: testing that routing, params parsing, and your
  actual handler code all wire together correctly — catches integration
  bugs a unit test can't see (wrong route path, forgotten `require`,
  middleware misconfiguration).

A healthy suite has many unit specs and a smaller number of request specs
covering the critical paths — not the other way around, since request
specs are slower and touch more code per failure.

## Testing-specific traps

- **A `double` with no expectation set never fails on a missed call** —
  if `Notifier` never calls `send_email` at all, a `double("mailer",
  send_email: "sent!")` stub doesn't notice. Use `expect(...).to
  receive` when the *call happening* is what you're verifying.
- **`instance_double` needs the real class loaded** at the time the spec
  runs, or it raises `RSpec::Mocks::MockExpectationError` complaining it
  can't find the class — a common cause of specs that pass locally but
  fail in a differently-ordered CI run.
- **Over-mocking couples the test to implementation, not behavior.**
  Stubbing every single collaborator method means the test only proves
  "the code calls things in the order I stubbed," not "the code produces
  the right result" — refactor the internals and the mock-heavy test
  breaks even though behavior didn't change.
- **`rack-test`'s `last_response` is only set after a request method**
  (`get`/`post`/etc.) — calling it before any request raises `NoMethodError:
  undefined method 'status' for nil`, a confusing error if you forget the
  request line.
- **Request specs re-require the whole app file.** If `app.rb` has
  top-level side effects (like an eager database connection), every
  request spec file that requires it pays that cost — keep expensive
  setup behind lazy initialization or `before(:suite)`.

## Cheat sheet

| Goal | RSpec syntax |
|---|---|
| Create a bare double | `double("name")` |
| Double with stubbed methods | `double("name", method: value)` |
| Verified double against a real class | `instance_double("ClassName", method: value)` |
| Expect a message will be sent | `expect(obj).to receive(:method)` |
| Expect specific arguments | `.with(arg1, arg2)` |
| Assert a message was already sent (spy style) | `expect(obj).to have_received(:method)` |
| Stub without checking it's called | `allow(obj).to receive(:method).and_return(value)` |
| Send a fake HTTP request | `get '/path'`, `post '/path', params` |
| Inspect the fake response | `last_response.status`, `last_response.body` |

## Exercise

Extend the `Notifier`/`app.rb` examples:

1. Write a unit spec for a new `PasswordValidator` class that calls out
   to an injected `dictionary_checker` collaborator — use a double to
   assert it's called with the candidate password, and stub two return
   values (`true`/`false`) to test both branches of your validator.
2. Write a request spec against the `POST /echo` route from module 1,
   asserting the response body echoes back the submitted `message`.
3. Deliberately break one assertion in each spec, run `rspec`, and
   paste the failure output — then fix it and paste the passing output.
