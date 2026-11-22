---
title: "Final Quiz"
description: "Test what you have learned across all four levels of Ruby Mastery Path with a 19-question final quiz."
---

# Final Quiz

19 questions drawn from all four levels of this course. You need **70% (14 out of 19)** to pass —
you can retake it as many times as you like. Your best result is saved in this browser.

<div id="mp-quiz-root"></div>

<script type="application/json" id="mp-quiz-data">
[
  {
    "question": "What does this print?\n```ruby\nh = Hash.new(0)\n'banana'.each_char { |c| h[c] += 1 }\nputs h.inspect\nputs h.max_by { |_, v| v }.first\n```\n",
    "options": [
      "{\"b\" => 1, \"a\" => 3, \"n\" => 2}\na",
      "{\"b\" => 1, \"a\" => 3, \"n\" => 2}\nn",
      "{b: 1, a: 3, n: 2}\na",
      "{\"a\" => 3, \"n\" => 2, \"b\" => 1}\na"
    ],
    "correctIndex": 0,
    "explanation": "Hash.new(0) supplies a default of 0 for missing keys; insertion order is kept. Verified by compiling and running it."
  },
  {
    "question": "What does this print?\n```ruby\nputs [1, 2, 3].each_with_object([]) { |x, memo| memo << x * x }.inspect\nputs (1..4).inject(:*)\nputs [[1, 2], [3, 4]].to_h { |a, b| [a, b * 2] }.inspect\n```\n",
    "options": [
      "[1, 4, 9]\n24\n[[1, 4], [3, 8]]",
      "[1, 2, 3]\n24\n{1 => 2, 3 => 4}",
      "[1, 4, 9]\n24\n{1 => 4, 3 => 8}",
      "[1, 4, 9]\n10\n{1 => 4, 3 => 8}"
    ],
    "correctIndex": 2,
    "explanation": "each_with_object returns the memo; inject(:*) multiplies; to_h with a block builds a hash. Verified by compiling and running it."
  },
  {
    "question": "What does this print?\n```ruby\ndef f\n  yield 1, 2\nend\nf { |a| puts a.inspect }\nf { |a, b| puts [a, b].inspect }\nf { |*all| puts all.inspect }\npr = proc { |a, b| [a, b] }\nputs pr.call([5, 6]).inspect\n```\n",
    "options": [
      "[1, 2]\n[1, 2]\n[1, 2]\n[5, 6]",
      "1\n[1, 2]\n[1, 2]\n[[5, 6], nil]",
      "1\n[1, 2]\n[1, 2]\n[5, 6]",
      "1\n[1, 2]\n[[1, 2]]\n[5, 6]"
    ],
    "correctIndex": 2,
    "explanation": "Blocks and procs are lenient: extra args are dropped and a single array is destructured. Verified by compiling and running it."
  },
  {
    "question": "What does this print?\n```ruby\nmodule Loud\n  def speak = super.upcase\nend\nclass Animal\n  def speak = 'hi'\nend\nclass Dog < Animal\n  prepend Loud\n  def speak = 'woof'\nend\nputs Dog.new.speak\nputs Dog.ancestors.first(3).inspect\n```\n",
    "options": [
      "HI\n[Loud, Dog, Animal]",
      "WOOF\n[Loud, Dog, Animal]",
      "woof\n[Dog, Loud, Animal]",
      "WOOF\n[Dog, Loud, Animal]"
    ],
    "correctIndex": 1,
    "explanation": "prepend puts the module in front of the class so it wraps the class's own method. Verified by compiling and running it."
  },
  {
    "question": "What does this print?\n```ruby\nclass Money\n  include Comparable\n  attr_reader :cents\n  def initialize(c) = @cents = c\n  def <=>(o) = cents <=> o.cents\n  def to_s = format('$%.2f', cents / 100.0)\nend\na, b = Money.new(500), Money.new(1250)\nputs a < b, [b, a].min, a.clamp(b, b), a == Money.new(500)\n```\n",
    "options": [
      "true\n$5.00\n$12.50\ntrue",
      "true\n$12.50\n$12.50\ntrue",
      "true\n$5.00\n$5.00\nfalse",
      "false\n$5.00\n$12.50\ntrue"
    ],
    "correctIndex": 0,
    "explanation": "Comparable derives <, min, clamp and == from <=>. Verified by compiling and running it."
  },
  {
    "question": "A method has a bare `rescue` and a typo inside the begin block. What goes wrong?",
    "options": [
      "It is faster",
      "The NameError or NoMethodError is swallowed and hidden; rescue specific errors",
      "Nothing",
      "Ruby warns"
    ],
    "correctIndex": 1,
    "explanation": "A bare rescue catches StandardError, which includes typos."
  },
  {
    "question": "Why should a custom error inherit from StandardError, not Exception?",
    "options": [
      "Law",
      "Style",
      "A bare rescue only catches StandardError; Exception also covers exit and interrupts",
      "Speed"
    ],
    "correctIndex": 2,
    "explanation": "Rescuing Exception can trap Ctrl-C."
  },
  {
    "question": "What does Bundler's Gemfile.lock guarantee?",
    "options": [
      "Faster Ruby",
      "Every machine installs the same gem versions",
      "Newest versions",
      "No dependencies"
    ],
    "correctIndex": 1,
    "explanation": "Commit it for applications."
  },
  {
    "question": "What is the N+1 query problem and its Rails fix?",
    "options": [
      "Deadlock; lock",
      "Slow index; add one",
      "Big payload; paginate",
      "One query for a list plus one per item; use includes/preload"
    ],
    "correctIndex": 3,
    "explanation": "Eager loading."
  },
  {
    "question": "How do you store passwords in a Ruby app?",
    "options": [
      "Plain text",
      "bcrypt (or argon2) hashes with salt, never reversible encryption",
      "Base64",
      "MD5"
    ],
    "correctIndex": 1,
    "explanation": "has_secure_password uses bcrypt."
  },
  {
    "question": "What is the safe way to put user input in an SQL query?",
    "options": [
      "Trust it",
      "Interpolate",
      "Bind parameters or the ORM's placeholders, never string interpolation",
      "Escape manually"
    ],
    "correctIndex": 2,
    "explanation": "Prevents SQL injection."
  },
  {
    "question": "Why make background jobs idempotent?",
    "options": [
      "Retries and duplicates mean a job may run more than once",
      "Speed",
      "Law",
      "Style"
    ],
    "correctIndex": 0,
    "explanation": "Use unique keys or check state first."
  },
  {
    "question": "What does GVL mean for CRuby threads?",
    "options": [
      "All parallel",
      "Threads crash",
      "No threads",
      "Only one thread runs Ruby code at a time, though threads can overlap I/O waits"
    ],
    "correctIndex": 3,
    "explanation": "Use processes or Ractors for CPU parallelism."
  },
  {
    "question": "Which structure protects a shared counter from a race condition?",
    "options": [
      "Nothing",
      "A Mutex around the read-modify-write",
      "sleep",
      "A string"
    ],
    "correctIndex": 1,
    "explanation": "synchronize."
  },
  {
    "question": "Why run containers as a non-root user?",
    "options": [
      "It limits damage if the app is compromised",
      "Speed",
      "Style",
      "Law"
    ],
    "correctIndex": 0,
    "explanation": "Least privilege."
  },
  {
    "question": "What HTTP status fits 'validation failed' in an API?",
    "options": [
      "200",
      "500",
      "404",
      "422 (or 400) with error details"
    ],
    "correctIndex": 3,
    "explanation": "Be consistent."
  },
  {
    "question": "What does RuboCop do in CI?",
    "options": [
      "Runs tests",
      "Fails the build on style and common-mistake violations",
      "Deploys",
      "Compiles"
    ],
    "correctIndex": 1,
    "explanation": "Keeps reviews about design."
  },
  {
    "question": "What does at-least-once message delivery require of a consumer?",
    "options": [
      "Single thread",
      "Idempotent handling of duplicates",
      "Nothing",
      "No acks"
    ],
    "correctIndex": 1,
    "explanation": "Ack after processing."
  },
  {
    "question": "Which is a good definition of done for the capstone?",
    "options": [
      "Pushed",
      "Compiles",
      "Works locally",
      "Tests pass in CI, lint is clean and it deploys from a clean checkout"
    ],
    "correctIndex": 3,
    "explanation": "Reproducibility is the proof."
  }
]
</script>

## Coding challenges

Try these on your own computer, then compare your output. Nothing you write is sent anywhere.

### Word frequency report

Implement words, top and report so the script prints a stop-word-filtered frequency chart. Ties are broken alphabetically and an empty text must not crash.

**Starter code**

```ruby
TEXT = <<~T
  The quick brown fox jumps over the lazy dog. The dog barks; the fox runs!
  A quick reply: the fox is quick, the dog is lazy.
T
STOP = %w[the a is over].freeze

# TODO: words(text)   -> lowercase words (letters and apostrophes only), stop words removed
# TODO: top(text, n)  -> the n most frequent [word, count] pairs, ties broken alphabetically
# TODO: report(text, n) -> lines like "dog   ### 3": word padded to the widest word, one # per count, then the count

puts "words: #{words(TEXT).size}, unique: #{words(TEXT).uniq.size}"
puts report(TEXT, 4)
puts top('', 3).inspect
```

**Expected output**

```text
words: 16, unique: 9
dog   ### 3
fox   ### 3
quick ### 3
lazy  ## 2
[]
```

<details><summary>Show solution and explanation</summary>

<pre><code>TEXT = &lt;&lt;~T
  The quick brown fox jumps over the lazy dog. The dog barks; the fox runs!
  A quick reply: the fox is quick, the dog is lazy.
T
STOP = %w[the a is over].freeze

def words(text)
  text.downcase.scan(/[a-z&#x27;]+/).reject { |w| STOP.include?(w) }
end

def top(text, n)
  words(text).tally.sort_by { |w, c| [-c, w] }.first(n)
end

def report(text, n)
  rows = top(text, n)
  width = rows.map { |w, _| w.length }.max
  rows.map { |w, c| format(&quot;%-#{width}s %s %d&quot;, w, &#x27;#&#x27; * c, c) }
end

puts &quot;words: #{words(TEXT).size}, unique: #{words(TEXT).uniq.size}&quot;
puts report(TEXT, 4)
puts top(&#x27;&#x27;, 3).inspect
</code></pre>

<p>scan with a regex extracts words, tally counts them and sort_by with [-count, word] gives a stable ranking. Verified by running the solution with Ruby 4.</p>
</details>

### A Retrier with exponential backoff

Implement Retrier so transient failures are retried with exponentially growing waits, permanent failures are not retried and the log always records the number of attempts. The sleeper is injected so no real time passes.

**Starter code**

```ruby
class TransientError < StandardError; end
class PermanentError < StandardError; end

# TODO: class Retrier with attr_reader :log
#   initialize(max: 4, base: 0.5, sleeper: lambda that logs "sleep <seconds>")
#   run { |attempt| ... } retries only TransientError, waiting base * 2**(attempt-1) between tries,
#   re-raises after max attempts, logs PermanentError ("permanent: msg") without retrying,
#   and always logs "done after <attempts>" in ensure. Logs "attempt N failed: msg" and "ok on attempt N".

r = Retrier.new
v = r.run { |n| n < 3 ? raise(TransientError, "boom#{n}") : "value" }
puts v, r.log
r2 = Retrier.new(max: 2)
begin
  r2.run { |n| raise TransientError, "down" }
rescue TransientError => e
  puts "gave up: #{e.message}"
end
puts r2.log
r3 = Retrier.new
begin
  r3.run { raise PermanentError, "bad input" }
rescue PermanentError
  puts r3.log
end
```

**Expected output**

```text
value
attempt 1 failed: boom1
sleep 0.5
attempt 2 failed: boom2
sleep 1.0
ok on attempt 3
done after 3
gave up: down
attempt 1 failed: down
sleep 0.5
attempt 2 failed: down
done after 2
permanent: bad input
done after 1
```

<details><summary>Show solution and explanation</summary>

<pre><code>class TransientError &lt; StandardError; end
class PermanentError &lt; StandardError; end

class Retrier
  attr_reader :log

  def initialize(max: 4, base: 0.5, sleeper: -&gt;(s) { @log &lt;&lt; &quot;sleep #{s}&quot; })
    @max, @base, @sleeper, @log = max, base, sleeper, []
  end

  def run
    attempt = 0
    begin
      attempt += 1
      result = yield attempt
      @log &lt;&lt; &quot;ok on attempt #{attempt}&quot;
      result
    rescue TransientError =&gt; e
      @log &lt;&lt; &quot;attempt #{attempt} failed: #{e.message}&quot;
      raise if attempt &gt;= @max
      @sleeper.call(@base * 2**(attempt - 1))
      retry
    rescue PermanentError =&gt; e
      @log &lt;&lt; &quot;permanent: #{e.message}&quot;
      raise
    ensure
      @log &lt;&lt; &quot;done after #{attempt}&quot;
    end
  end
end

r = Retrier.new
v = r.run { |n| n &lt; 3 ? raise(TransientError, &quot;boom#{n}&quot;) : &quot;value&quot; }
puts v, r.log
r2 = Retrier.new(max: 2)
begin
  r2.run { |n| raise TransientError, &quot;down&quot; }
rescue TransientError =&gt; e
  puts &quot;gave up: #{e.message}&quot;
end
puts r2.log
r3 = Retrier.new
begin
  r3.run { raise PermanentError, &quot;bad input&quot; }
rescue PermanentError
  puts r3.log
end
</code></pre>

<p>retry re-runs the begin block, the injected sleeper keeps the test deterministic, and ensure records the final attempt count once. Verified by running the solution with Ruby 4.</p>
</details>
