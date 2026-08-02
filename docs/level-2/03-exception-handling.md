# 03 · Exception Handling

Things go wrong at runtime: files don't exist, APIs time out, users type
garbage into forms. Ruby's exception handling lets you contain those
failures instead of letting them crash the whole program — and, just as
importantly, lets you guarantee that cleanup code (closing a file, releasing
a lock) runs no matter what happened.

## begin/rescue — catching an exception

```ruby
begin
  1 / 0
rescue ZeroDivisionError => e
  puts "Can't divide by zero: #{e.message}"
end
# Can't divide by zero: divided by 0
```

`rescue` catches the named exception class **and any subclass of it**. If
you omit the class entirely, Ruby rescues `StandardError` (and its
subclasses) by default — not literally everything:

```ruby
begin
  raise "something broke"   # shorthand for raise RuntimeError, "something broke"
rescue => e                   # equivalent to `rescue StandardError => e`
  puts "#{e.class}: #{e.message}"
end
# RuntimeError: something broke
```

## Rescuing multiple, specific exception types

List the most specific exceptions first — Ruby checks `rescue` clauses top
to bottom and uses the first match:

```ruby
def parse_number(text)
  Integer(text)
rescue ArgumentError => e
  puts "Not a valid integer: #{text.inspect}"
  nil
rescue TypeError => e
  puts "Wrong type entirely: #{e.message}"
  nil
end

parse_number("42")     # (no output, returns 42)
parse_number("abc")     # Not a valid integer: "abc"
parse_number(nil)        # Wrong type entirely: can't convert nil into Integer
```

Notice `rescue` can go directly in a `def`/`end` without a matching `begin`
— Ruby methods have an implicit `begin` block around their whole body.

## ensure — always runs, exception or not

```ruby
def read_config(path)
  file = File.open(path)
  file.read
rescue Errno::ENOENT
  puts "Config file missing, using defaults"
  "{}"
ensure
  file&.close   # runs whether the rescue fired or not, or even if `read` raised something else
  puts "Cleanup done"
end

read_config("does_not_exist.json")
# Config file missing, using defaults
# Cleanup done
```

`ensure` runs even if the method returns from inside `begin`/`rescue`, or
even if a *different*, unrescued exception propagates out — it is Ruby's
guarantee for cleanup code, similar to `finally` in other languages.

## else — code that runs only if nothing was raised

```ruby
begin
  result = 10 / 2
rescue ZeroDivisionError
  puts "divide by zero!"
else
  puts "Success: #{result}"   # only runs when NO exception was raised
ensure
  puts "always runs"
end
# Success: 5
# always runs
```

`else` is easy to forget about, but it's the cleanest way to separate
"code that might raise" from "code that should only run on success" —
putting the success-only logic inside `begin` instead would accidentally
get wrapped in the same `rescue`.

## Custom exception classes

Real applications define their own exception hierarchy instead of raising
generic `RuntimeError` everywhere, so callers can rescue precisely the
failures they care about:

```ruby
class ApiError < StandardError; end
class RateLimitedError < ApiError
  def initialize(msg = "Rate limit exceeded, try again later")
    super
  end
end
class NotFoundError < ApiError; end

def fetch_resource(status)
  case status
  when 429 then raise RateLimitedError
  when 404 then raise NotFoundError, "Resource not found"
  else "ok"
  end
end

begin
  fetch_resource(429)
rescue RateLimitedError => e
  puts "Backing off: #{e.message}"
rescue ApiError => e   # catches NotFoundError and any other ApiError subclass
  puts "API problem: #{e.message}"
end
# Backing off: Rate limit exceeded, try again later
```

Subclassing `StandardError` (never `Exception` directly — see below) and
building a small hierarchy like this means callers can rescue narrowly
(`RateLimitedError`) or broadly (`ApiError`) depending on what they need.

## Why StandardError, not Exception

`Exception` is Ruby's root exception class, but it also covers things like
`SyntaxError`, `NoMemoryError`, and `SystemExit` — failures a program
generally should *not* try to catch and recover from. `rescue` with no
class, and virtually every exception you define yourself, should build on
`StandardError`:

```ruby
# Don't do this:
class MyError < Exception; end   # also gets caught by broad `rescue Exception`,
                                    # which can accidentally swallow SystemExit,
                                    # Interrupt (Ctrl-C), etc.

# Do this:
class MyError < StandardError; end
```

## retry — attempting the risky operation again

`retry` jumps back to the top of the `begin` block, which is exactly what
you want for transient failures like a flaky network call. Always cap the
attempts — an unconditional `retry` can loop forever:

```ruby
attempts = 0

begin
  attempts += 1
  raise "simulated timeout" if attempts < 3
  puts "Succeeded on attempt #{attempts}"
rescue => e
  if attempts < 3
    puts "Attempt #{attempts} failed (#{e.message}), retrying..."
    retry
  else
    puts "Giving up after #{attempts} attempts"
  end
end
# Attempt 1 failed (simulated timeout), retrying...
# Attempt 2 failed (simulated timeout), retrying...
# Succeeded on attempt 3
```

## Re-raising and raise with no arguments

Inside a `rescue` block, a bare `raise` re-raises the exception currently
being handled — useful for logging without swallowing the error:

```ruby
def risky_operation
  yield
rescue => e
  puts "Logging error: #{e.message}"
  raise   # re-raises the SAME exception, preserving its original backtrace
end

begin
  risky_operation { raise "boom" }
rescue RuntimeError => e
  puts "Caller saw: #{e.message}"
end
# Logging error: boom
# Caller saw: boom
```

## Cheat sheet

| Keyword | Purpose |
|---|---|
| `begin ... rescue ... end` | catch exceptions raised inside `begin` |
| `rescue SomeError => e` | catch a specific class (and its subclasses) |
| `else` | runs only if no exception was raised |
| `ensure` | always runs — cleanup code |
| `retry` | jump back to the top of `begin` and try again |
| `raise` (no args, inside rescue) | re-raise the current exception |
| `raise SomeError, "message"` | raise a specific custom exception |

## Exercise

Write a method `safe_divide(a, b)` that returns the division result, or
rescues `ZeroDivisionError` and returns `nil` while printing a friendly
message — use `ensure` to always print `"Division attempted."` regardless
of outcome. Then define a small exception hierarchy — `ValidationError <
StandardError`, with subclasses `BlankFieldError` and `TooLongError` — and
write a method `validate_username(name)` that raises the appropriate one
(blank, or longer than 20 characters) so a caller can rescue either
specifically or `ValidationError` broadly.
