# 10 · Capstone Project

The final project pulls together every Level 4 module into one small but
production-shaped service: a Sinatra + ActiveRecord task API with a
service-object layer, custom typed errors, consistent JSON error
responses, and a full request-spec suite — the same shape you'd deploy
via the Docker setup from module 5 and wire into the CI pipeline from
module 4.

## What you'll build

A `Capstone::App` exposing:

- `GET /tasks` — list all tasks
- `POST /tasks` — create a task (title required, priority validated
  against an allowed list), `201`/`422`
- `GET /tasks/:id` — fetch one, raising a custom `NotFoundError` handled
  as a `404` with a consistent error envelope
- `PATCH /tasks/:id/complete` — mark a task done

## Project layout

```text
capstone/
    Gemfile
    Dockerfile
    .rubocop.yml
    .github/
        workflows/
            ci.yml
    lib/
        errors.rb
        task.rb
        task_service.rb
        app.rb
    spec/
        task_api_spec.rb
```

## lib/errors.rb — a typed domain error

```ruby
module Capstone
  class NotFoundError < StandardError; end
end
```

A dedicated error class, rather than a generic `raise "not found"`, lets
the route layer catch *specifically* this failure mode (via Sinatra's
`error SomeClass do ... end`) without accidentally swallowing unrelated
exceptions.

## lib/task.rb — the model

```ruby
require 'active_record'

class Task < ActiveRecord::Base
  validates :title, presence: true
  validates :priority, inclusion: { in: %w[low medium high] }, allow_nil: true
end
```

## lib/task_service.rb — the service layer (module 1)

```ruby
require_relative 'errors'
require_relative 'task'

module Capstone
  class TaskService
    Result = Struct.new(:success?, :data, :error)

    def self.create(attrs)
      task = Task.new(title: attrs["title"], priority: attrs["priority"])
      if task.save
        Result.new(true, task, nil)
      else
        Result.new(false, nil, task.errors.full_messages.join(", "))
      end
    end

    def self.find!(id)
      Task.find_by(id: id) || raise(NotFoundError, "Task #{id} not found")
    end

    def self.complete(id)
      task = find!(id)
      task.update!(done: true)
      Result.new(true, task, nil)
    rescue NotFoundError => e
      Result.new(false, nil, e.message)
    end
  end
end
```

The route layer never touches ActiveRecord directly for writes — it asks
`TaskService` and branches on an explicit `Result`, exactly the pattern
from module 1.

## lib/app.rb — routes, error handling, and config from ENV

```ruby
require 'sinatra/base'
require 'active_record'
require 'json'
require_relative 'task_service'

ActiveRecord::Base.establish_connection(
  adapter: 'sqlite3',
  database: ENV.fetch('DATABASE_URL', ':memory:')
)

ActiveRecord::Schema.define do
  create_table :tasks, force: false do |t|
    t.string  :title, null: false
    t.string  :priority, default: 'medium'
    t.boolean :done, default: false
    t.timestamps
  end
end unless ActiveRecord::Base.connection.table_exists?(:tasks)

module Capstone
  class App < Sinatra::Base
    set :host_authorization, { permitted_hosts: [] }
    set :show_exceptions, false
    set :raise_errors, false
    before { content_type :json }

    error Capstone::NotFoundError do
      status 404
      { error: { code: "not_found", message: env['sinatra.error'].message } }.to_json
    end

    get '/tasks' do
      Task.all.to_json
    end

    post '/tasks' do
      data = JSON.parse(request.body.read) rescue {}
      result = TaskService.create(data)
      if result.success?
        status 201
        result.data.to_json
      else
        status 422
        { error: { code: "validation_failed", message: result.error } }.to_json
      end
    end

    get '/tasks/:id' do
      TaskService.find!(params[:id]).to_json
    end

    patch '/tasks/:id/complete' do
      result = TaskService.complete(params[:id])
      result.data.to_json
    end
  end
end
```

Three module-4 lessons show up directly here: `ENV.fetch('DATABASE_URL',
':memory:')` reads configuration from the environment (security module)
with a sane local default; `unless
ActiveRecord::Base.connection.table_exists?(:tasks)` avoids re-running
`create_table` against a real persistent database on every boot; and
`error Capstone::NotFoundError do ... end` returns the same
`{ error: { code:, message: } }` envelope shape from the API-design
module for every error case, including validation failures in `POST
/tasks`.

## spec/task_api_spec.rb — the request spec suite

```ruby
require 'rack/test'
require_relative '../lib/app'

RSpec.describe Capstone::App do
  include Rack::Test::Methods
  def app; Capstone::App; end

  before { Task.delete_all }

  def post_json(path, body)
    post path, body.to_json, { 'CONTENT_TYPE' => 'application/json' }
  end

  it "creates a task" do
    post_json '/tasks', { title: "Ship capstone", priority: "high" }
    expect(last_response.status).to eq(201)
    expect(JSON.parse(last_response.body)["title"]).to eq("Ship capstone")
  end

  it "rejects invalid priority" do
    post_json '/tasks', { title: "X", priority: "urgent!" }
    expect(last_response.status).to eq(422)
  end

  it "404s via the custom error handler" do
    get '/tasks/999'
    expect(last_response.status).to eq(404)
    expect(JSON.parse(last_response.body)["error"]["code"]).to eq("not_found")
  end

  it "completes a task" do
    post_json '/tasks', { title: "Finish" }
    id = JSON.parse(last_response.body)["id"]
    patch "/tasks/#{id}/complete"
    expect(JSON.parse(last_response.body)["done"]).to eq(true)
  end
end
```

## Running it

```text
$ ruby spec/task_api_spec.rb
-- create_table(:tasks, {force: false})
   -> 0.0032s
....

Finished in 0.04029 seconds (files took 0.55999 seconds to load)
4 examples, 0 failures
```

All four specs pass: successful creation, a validation rejection, the
custom 404 error envelope, and the completion flow. Note
`set :show_exceptions, false` and `set :raise_errors, false` — Sinatra's
development defaults render its own HTML debug page for unhandled
exceptions; a production-shaped app disables that so your own `error`
handlers (and their JSON envelope) actually run instead.

## Dockerfile (module 5)

```dockerfile
FROM ruby:3.3-slim
WORKDIR /app
COPY Gemfile Gemfile.lock ./
RUN bundle install --without development test
COPY . .
ENV DATABASE_URL=capstone.sqlite3
EXPOSE 4567
CMD ["ruby", "-e", "require './lib/app'; Capstone::App.run! host: '0.0.0.0'"]
```

## .github/workflows/ci.yml (module 4)

```yaml
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
      - run: bundle exec rspec
      - run: bundle exec rubocop
```

## What this project demonstrates from Level 4

- **Module 1**: `TaskService` as a service object with an explicit
  `Result`, keeping business logic out of the route layer.
- **Module 3**: a custom `NotFoundError` used deliberately instead of
  generic exceptions, matching the security module's principle of
  explicit, intentional error paths rather than accidental ones.
- **Module 4**: a request-spec suite structured for CI, plus the
  `ci.yml` running it automatically.
- **Module 5**: a Dockerfile following the layer-caching and
  `ENV.fetch`-based configuration pattern.
- **Module 7**: a consistent `{ error: { code:, message: } }` response
  envelope across every failure mode.
- **Module 9**: a `.rubocop.yml`-governed style pass integrated into the
  same CI job as the test suite.

## Stretch goals

1. Add a `DELETE /tasks/:id` route using `TaskService.find!` (reusing the
   existing `NotFoundError` handling) and a matching request spec.
2. Add API versioning (module 7): a `GET /api/v2/tasks/:id` returning
   `priority_label` (a human-readable string like `"High Priority"`)
   alongside the existing fields, without breaking the existing `v1`
   shape.
3. Wrap task creation in a background job (module 2) that sends a
   (simulated, printed) notification — enqueue it from
   `TaskService.create` on success, using the `InlineQueue` pattern from
   that module so the spec suite doesn't need real Sidekiq/Redis.
4. Add a `Rakefile` (Level 3, module 8) with a `db:reset` task that
   drops and recreates the `tasks` table, and a `spec` task that shells
   out to run the whole RSpec suite — so `rake spec` becomes the one
   command CI and local development both use.
