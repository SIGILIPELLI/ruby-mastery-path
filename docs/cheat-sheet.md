---
title: "Ruby Cheat Sheet: Quick Reference, Beginner to Advanced"
description: "A free Ruby cheat sheet: 35 quick-reference tables from every level of the Ruby Mastery Path, each linked to its full lesson."
---

# Ruby Cheat Sheet

Quick-reference tables from **35 lessons** of the free Ruby Mastery Path, in course order from beginner to advanced. Each section links to the full lesson with explanations and runnable examples.

## Level 1 · Entry

### [02 · Variables &amp; Types](level-1/02-variables-types.md)

| Concept | Example |
|---------|---------|
| Falsy values | `nil`, `false` — that's the entire list |
| Integer division | `7 / 2` → `3` (use `.to_f` for float division) |
| String conversion | `.to_s`, `.to_i`, `.to_f` |
| Symbol | `:name` — lightweight, immutable identifier |
| Constant | `UPPER_SNAKE_CASE`, warns (not errors) on reassignment |

### [03 · Control Flow](level-1/03-control-flow.md)

| Construct | Use when |
|-----------|----------|
| `if / elsif / else` | General branching |
| `unless` | Branching on a negative condition, reads more naturally |
| Trailing `if`/`unless` | Short single-line guard clauses |
| `case / when` | Comparing one value against several possibilities/ranges |
| `while` / `until` | Loop while/until a condition holds |
| `each` | Idiomatic iteration (preferred over `for`) |
| `break` | Exit the loop entirely |
| `next` | Skip to the next iteration |

### [04 · Methods](level-1/04-methods.md)

| Feature | Syntax |
|---------|--------|
| Basic method | `def name(a, b) ... end` |
| Default argument | `def name(a, b = 10) ... end` |
| Keyword argument | `def name(a:, b: 10) ... end` |
| Splat (variadic) | `def name(*args) ... end` |
| Double splat | `def name(**kwargs) ... end` |
| Implicit return | Last expression's value, or explicit `return` |
| Predicate convention | `name?` returns true/false |
| Dangerous/mutating convention | `name!` |

### [05 · Arrays &amp; Hashes](level-1/05-arrays-hashes.md)

| Task | Method |
|------|--------|
| Append | `arr << x` or `arr.push(x)` |
| Transform each element | `arr.map { \|x\| ... }` |
| Keep matching elements | `arr.select { \|x\| ... }` |
| Remove matching elements | `arr.reject { \|x\| ... }` |
| Sum/combine | `arr.reduce(:+)` or `arr.sum` |
| First match | `arr.find { \|x\| ... }` |
| Safe hash lookup | `hash.fetch(:key, default)` |
| Iterate a hash | `hash.each { \|k, v\| ... }` |

### [06 · Strings &amp; String Methods](level-1/06-strings.md)

| Task | Method |
|------|--------|
| Interpolate a value | `"text #{expr} text"` |
| Format a float | `"%.2f" % value` or `format("%.2f", value)` |
| Multi-line literal text | `<<~LABEL ... LABEL` (heredoc) |
| Check substring | `.include?`, `.start_with?`, `.end_with?` |
| Regex match | `.match?(/pattern/)` |
| Mutate in place | Bang methods: `.upcase!`, `.strip!`, etc. |

### [07 · Classes &amp; Objects Basics](level-1/07-classes-objects.md)

| Feature | Syntax |
|---------|--------|
| Define a class | `class Name ... end` |
| Constructor | `def initialize(args) ... end` |
| Instance variable | `@name` |
| Getter + setter | `attr_accessor :name` |
| Getter only | `attr_reader :name` |
| Class method | `def self.method_name ... end` |
| Inheritance | `class Sub < Base ... end` |
| Call parent method | `super(args)` |

### [08 · Blocks, Procs &amp; Lambdas Introduction](level-1/08-blocks-procs-lambdas.md)

| Feature | Syntax |
|---------|--------|
| Block (multi-line) | `method do \|arg\| ... end` |
| Block (single-line) | `method { \|arg\| ... }` |
| Invoke a passed block | `yield(args)` |
| Check for a block | `block_given?` |
| Named parameter for a block | `def m(&block)` |
| Create a Proc | `Proc.new { \|args\| ... }` |
| Create a lambda | `->(args) { ... }` or `lambda { \|args\| ... }` |
| Call a Proc/lambda | `.call(args)`, `.(args)`, or `[args]` |

### [09 · Gems &amp; Bundler Basics](level-1/09-gems-bundler.md)

| Task | Command |
|------|---------|
| Install a gem globally | `gem install name` |
| Install project dependencies | `bundle install` |
| Run a script with locked versions | `bundle exec ruby script.rb` |
| Add a gem | Add a line to `Gemfile`, then `bundle install` |
| Pessimistic version constraint | `gem "name", "~> 1.2"` |
| Group gems (e.g. test-only) | `group :test do ... end` |

## Level 2 · Intermediate

### [01 · OOP Deep Dive](level-2/01-oop-deep-dive.md)

| Goal | Tool |
|------|------|
| Share instance methods across unrelated classes | `module M; end` + `include M` |
| Add class-level (or singleton) methods | `extend M` |
| Wrap/override a method and still call the original | `prepend M` + `super` |
| Inspect method lookup order | `SomeClass.ancestors` |
| Handle undefined method calls dynamically | `method_missing` + `respond_to_missing?` |
| Add methods to an existing class | reopen it (monkey-patch) — use sparingly |

### [02 · Blocks/Procs/Lambdas Advanced](level-2/02-blocks-procs-lambdas-advanced.md)

| Feature | Syntax | Notes |
|---|---|---|
| Capture the block as a Proc | `def m(&block)` | costs a Proc allocation |
| Convert Proc/lambda back to a block | `method(&my_proc)` | |
| Symbol shorthand block | `&:method_name` | via `Symbol#to_proc` |
| Extract a method as a callable | `method(:name)` | returns a `Method` object |
| Partially apply a lambda | `my_lambda.curry` | |
| Closures capture by reference | shared local variables | each closure-creating call gets fresh locals |

### [03 · Exception Handling](level-2/03-exception-handling.md)

| Keyword | Purpose |
|---|---|
| `begin ... rescue ... end` | catch exceptions raised inside `begin` |
| `rescue SomeError => e` | catch a specific class (and its subclasses) |
| `else` | runs only if no exception was raised |
| `ensure` | always runs — cleanup code |
| `retry` | jump back to the top of `begin` and try again |
| `raise` (no args, inside rescue) | re-raise the current exception |
| `raise SomeError, "message"` | raise a specific custom exception |

### [04 · File I/O](level-2/04-file-io.md)

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

### [05 · Testing with RSpec](level-2/05-testing-rspec.md)

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

### [06 · Working with JSON/APIs](level-2/06-json-apis.md)

| Task | Code |
|---|---|
| Parse a JSON string | `JSON.parse(str)` |
| Parse with symbol keys | `JSON.parse(str, symbolize_names: true)` |
| Convert an object to JSON | `obj.to_json` |
| Pretty-print JSON | `JSON.pretty_generate(obj)` |
| Simple GET request | `Net::HTTP.get_response(uri)` |
| Check for a successful response | `response.is_a?(Net::HTTPSuccess)` |
| POST with a body | `Net::HTTP::Post.new(uri)` + `request.body = ...` |
| Set timeouts | `http.open_timeout =`, `http.read_timeout =` |

### [07 · Metaprogramming Basics](level-2/07-metaprogramming-basics.md)

| Tool | What it does |
|---|---|
| `define_method(name) { ... }` | defines a real method dynamically |
| `method_missing` | intercepts calls to undefined methods |
| `respond_to_missing?` | keeps `respond_to?` honest alongside `method_missing` |
| `instance_variable_get/set` | read/write an ivar by name |
| `send` / `public_send` | call a method by name; `send` bypasses `private` |
| `define_singleton_method` | defines a method on one specific object |
| `class_eval` | runs a block as if inside the class body (reopens it) |

### [08 · Gem Development Basics](level-2/08-gem-development.md)

| Task | Command / File |
|---|---|
| Scaffold a new gem | `bundle gem my_gem` |
| Gem metadata | `my_gem.gemspec` |
| Namespace convention | `module MyGem; ... end` around all code |
| Build a `.gem` file | `gem build my_gem.gemspec` |
| Install a local build | `gem install ./my_gem-VERSION.gem` |
| Publish publicly | `gem push my_gem-VERSION.gem` |
| Versioning scheme | semantic versioning: `MAJOR.MINOR.PATCH` |

### [09 · Enumerable &amp; Comparable](level-2/09-enumerable-comparable.md)

| To get... | Implement... | Then you get for free |
|---|---|---|
| Iteration methods (`map`, `select`, `reduce`, ...) | `each` + `include Enumerable` | `to_a`, `count`, `sort`, `min`, `max`, `include?`, `first`, `sum`, ... |
| Comparison operators | `<=>` + `include Comparable` | `<`, `<=`, `==`, `>=`, `>`, `between?`, `clamp` |
| Sort by a computed key efficiently | n/a | `sort_by { \|x\| key }` over `sort { \|a,b\| ... }` |

## Level 3 · Advanced

### [01 · Building Web Apps with Sinatra](level-3/01-sinatra.md)

| Task | Sinatra code |
|---|---|
| Define a GET route | `get('/path') { ... }` |
| Named URL parameter | `get('/x/:id') { params[:id] }` |
| Read form/query param | `params[:field]` |
| Render a template | `erb :template_name` |
| Run before every route | `before { ... }` |
| Stop early with a status | `halt 404, "Not found"` |
| Redirect | `redirect '/somewhere'` |
| Set a response header | `headers['X-Custom'] = 'value'` |
| Modular app base class | `class App < Sinatra::Base` |
| Test in-process | `include Rack::Test::Methods` |

### [02 · Databases with ActiveRecord](level-3/02-activerecord.md)

| Task | ActiveRecord code |
|---|---|
| Connect to a database | `ActiveRecord::Base.establish_connection(...)` |
| Find by primary key | `Book.find(1)` |
| Find one matching row | `Book.find_by(title: "...")` |
| Filter rows | `Book.where(year: 1969)` |
| Order results | `Book.order(year: :desc)` |
| Fetch only certain columns | `Book.pluck(:title)` |
| Raise on invalid save | `book.save!` / `Book.create!(...)` |
| Check validity without saving | `book.valid?` |
| See validation errors | `book.errors.full_messages` |
| Avoid N+1 queries | `Book.includes(:author)` |
| Destroy with callbacks | `book.destroy` |
| Delete without callbacks | `book.delete` |

### [03 · Testing Advanced](level-3/03-testing-advanced.md)

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

### [04 · Metaprogramming Advanced](level-3/04-metaprogramming-advanced.md)

| Task | Code |
|---|---|
| Reopen a class and add a method | `SomeClass.class_eval { def m; end }` |
| Run a block with a different `self` | `obj.instance_eval { ... }` |
| Intercept unknown method calls | `def method_missing(name, *args); ...; end` |
| Keep `respond_to?` honest | `def respond_to_missing?(name, priv); ...; end` |
| Generate a method from a name | `define_method(:name) { ... }` |
| Generate a method with args | `define_method(:name) { \|x\| ... }` |
| Call a private method externally | `obj.send(:private_method)` |
| Check defined methods | `SomeClass.instance_methods(false)` |

### [05 · Design Patterns in Ruby](level-3/05-design-patterns.md)

| Pattern | Ruby idiom |
|---|---|
| Singleton | `include Singleton`, then `Klass.instance` |
| Observer | `include Observable`, `changed` + `notify_observers` |
| Factory | Class method returning different classes by argument |
| Decorator | `prepend SomeModule` + `super` |
| Strategy | Pass a block/lambda instead of a class hierarchy |
| Adapter (bonus) | Wrap an incompatible object, forward calls via `method_missing` or explicit delegation |
| Template method (bonus) | Base class defines the skeleton, calls `NotImplementedError`-raising hook methods subclasses override |

### [06 · Performance &amp; Profiling](level-3/06-performance-profiling.md)

| Task | Code |
|---|---|
| Time a block once | `Benchmark.measure { ... }` |
| Compare a few blocks, one run each | `Benchmark.bm(20) { \|x\| x.report("a") { ... } }` |
| Compare with statistical confidence | `Benchmark.ips { \|x\| x.report("a") { ... }; x.compare! }` |
| Count allocated objects | `GC.stat[:total_allocated_objects]` |
| Force a GC run (rarely needed) | `GC.start` |
| Check current heap size | `GC.stat[:heap_live_slots]` |
| Time a whole script from the shell | `time ruby script.rb` |

### [07 · Concurrency in Ruby](level-3/07-concurrency.md)

| Task | Code |
|---|---|
| Start a thread | `Thread.new { ... }` |
| Wait for a thread to finish | `thread.join` |
| Get a thread's return value | `thread.value` |
| Protect shared state | `mutex = Mutex.new; mutex.synchronize { ... }` |
| Create a fiber | `Fiber.new { ... }` |
| Pause a fiber | `Fiber.yield` |
| Resume a fiber | `fiber.resume` |
| Fork a real OS process | `Process.fork { ... }` |
| Wait for a forked process | `Process.wait` |

### [08 · Working with Rake Tasks](level-3/08-rake-tasks.md)

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

### [09 · Building CLIs with Thor or OptionParser](level-3/09-cli-thor-optparse.md)

| Task | OptionParser | Thor |
|---|---|---|
| Declare a flag | `opts.on("-v", "--verbose") { ... }` | `method_option :verbose, type: :boolean` |
| Flag with a value | `opts.on("--count=N", Integer) { \|n\| ... }` | `method_option :count, type: :numeric` |
| Read parsed value | block argument / local hash | `options[:count]` inside the method |
| Positional args | leftover `ARGV` after `parse!` | method parameters |
| Auto-generated help | `opts.banner` + `-h` block | `desc` + built-in `help` command |
| Multiple subcommands | manual `case ARGV[0]` dispatch | one method per subcommand, automatic |
| Entry point | `parser.parse!(ARGV)` | `MyClass.start(ARGV)` |

## Level 4 · Master

### [01 · Rails/Production Patterns](level-4/01-production-patterns.md)

| Pattern | Responsibility | Typical entry point |
|---|---|---|
| Service object | One business operation, possibly multi-model | `SomeAction.new(...).call` |
| Form object | Multi-model input + validation for one form | `form.valid?`, `form.save` |
| Query object | One reusable, complex query | `SomeQuery.new.call` |
| Presenter/decorator | View-specific formatting over a model | `Presenter.new(model).formatted_x` |
| Result struct | Explicit success/failure + payload/error | `Struct.new(:success?, :error, :data)` |

### [02 · Background Jobs (Sidekiq)](level-4/02-background-jobs.md)

| Task | Sidekiq API |
|---|---|
| Make a class a job | `include Sidekiq::Job` |
| Enqueue a job | `SomeJob.perform_async(arg1, arg2)` |
| Enqueue for later | `SomeJob.perform_in(5.minutes, arg1)` |
| Define the work | `def perform(arg1, arg2); ...; end` |
| Set retry count | `sidekiq_options retry: 5` |
| Assign a queue | `sidekiq_options queue: "critical"` |
| Test without Redis | `Sidekiq::Testing.fake!` |
| Assert enqueued in a test | `expect(SomeJob.jobs.size).to eq(1)` |

### [03 · Security Best Practices](level-4/03-security-best-practices.md)

| Risk | Wrong | Right |
|---|---|---|
| SQL injection | `where("x = '#{v}'")` | `where("x = ?", v)` or `where(x: v)` |
| Mass assignment | `Model.create(raw_params)` | `Model.create(raw_params.slice(*ALLOWED))` |
| Hardcoded secret | `KEY = "sk_live_..."` | `KEY = ENV.fetch("KEY")` |
| Plaintext password | `password == stored_plain` | `BCrypt::Password.new(hash) == password` |
| Unescaped output | `raw(user_input)` | default ERB/Rails auto-escaping |
| Silent missing config | `ENV["KEY"]` | `ENV.fetch("KEY")` |

### [04 · Testing at Scale &amp; CI](level-4/04-testing-at-scale-ci.md)

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

### [05 · Deployment (Docker)](level-4/05-deployment-docker.md)

| Task | Command / directive |
|---|---|
| Base image | `FROM ruby:3.3-slim` |
| Set working directory | `WORKDIR /app` |
| Install gems (cached layer) | `COPY Gemfile* ./` then `RUN bundle install` |
| Copy app code | `COPY . .` |
| Document the listening port | `EXPOSE 4567` |
| Container entry point | `CMD ["ruby", "app.rb", "-o", "0.0.0.0"]` |
| Build an image | `docker build -t name .` |
| Run with a port mapped | `docker run -p 4567:4567 name` |
| Pass an env var at runtime | `docker run -e KEY=value name` |
| Multi-service local dev | `docker-compose up` |
| Ignore files from the build context | `.dockerignore` |

### [06 · Performance at Scale](level-4/06-performance-at-scale.md)

| Pattern | Use when | Ruby idiom |
|---|---|---|
| Memoization | Same input always produces the same output, for this process's lifetime | `@cache[key] ||= compute` |
| TTL cache | Value can go stale, tolerable staleness window | `store expires_at`, check before reuse |
| Shared cache (Redis) | Multiple processes need the same cache | `Rails.cache.fetch(key, expires_in: n) { ... }` |
| Event-based invalidation | Freshness matters more than simplicity | `cache.delete(key)` at the point of mutation |
| HTTP caching | Same response for everyone, cacheable at the edge | `Cache-Control` response header |

### [07 · API Design &amp; Versioning](level-4/07-api-design-versioning.md)

| Task | Convention |
|---|---|
| List resources | `GET /things` |
| Fetch one | `GET /things/:id` |
| Create | `POST /things` |
| Full replace | `PUT /things/:id` |
| Partial update | `PATCH /things/:id` |
| Delete | `DELETE /things/:id` |
| Version in URL | `/api/v1/things` |
| Version via header | `Accept: application/vnd.app.v2+json` |
| Consistent error shape | `{ error: { code:, message: } }` |
| Success status for creation | `201 Created` |
| Success status for deletion | `204 No Content` |

### [08 · Message Queues](level-4/08-message-queues.md)

| Concept | Ruby stand-in used here | Real-world equivalent |
|---|---|---|
| Point-to-point queue | `Queue.new`, `push`/`pop` | SQS, a Sidekiq/Redis queue |
| Pub/sub fan-out | `Topic#subscribe`/`#publish` | RabbitMQ exchange, SNS topic |
| Acknowledgment | (not modeled — `pop` is destructive) | Broker-level `ack`/`nack` |
| Dead-letter queue | (not modeled) | Broker's DLQ feature |
| Idempotent consumer | Guard clause before side effects | Same principle, any broker |

### [09 · Code Quality &amp; RuboCop](level-4/09-code-quality-rubocop.md)

| Task | Command |
|---|---|
| Check a file | `rubocop path/to/file.rb` |
| Check the whole project | `rubocop` |
| Safe auto-fix | `rubocop -a` |
| Full auto-fix (review the diff!) | `rubocop -A` |
| Generate a todo file for existing violations | `rubocop --auto-gen-config` |
| Disable a cop for one line | `# rubocop:disable Cop/Name` ... `# rubocop:enable Cop/Name` |
| Disable a cop project-wide | `Cop/Name: { Enabled: false }` in `.rubocop.yml` |
| List all available cops | `rubocop --show-cops` |

---

Go deeper with the full [Ruby Mastery Path](index.md) curriculum or test yourself with the [interview questions](interview-questions.md).
