# 10 · Project — REST API Service

This capstone pulls together everything from Level 3: Sinatra routing,
ActiveRecord as a standalone ORM, and RSpec request specs with
`rack-test`. You'll build a small JSON REST API for managing tasks, with
full CRUD, validation, proper HTTP status codes, and a test suite that
runs with no external database server (SQLite in-memory).

## What you'll build

A `TaskAPI` Sinatra app exposing:

- `GET /tasks` — list all tasks as JSON
- `POST /tasks` — create a task, `201` on success, `422` with error
  details on invalid input
- `GET /tasks/:id` — fetch one task, `404` if it doesn't exist
- `PATCH /tasks/:id` — update a task's `done` flag
- `DELETE /tasks/:id` — remove a task, `204` on success

## Project layout

```text
task_api/
    Gemfile
    lib/
        task_api.rb
    spec/
        task_api_spec.rb
```

## Gemfile

```ruby
# Gemfile
source "https://rubygems.org"

gem "sinatra"
gem "activerecord"
gem "sqlite3"
gem "rack-test"
gem "rspec"
```

## lib/task_api.rb — the whole app

```ruby
require 'sinatra/base'
require 'active_record'
require 'json'

ActiveRecord::Base.establish_connection(adapter: 'sqlite3', database: ':memory:')

ActiveRecord::Schema.define do
  create_table :tasks do |t|
    t.string  :title, null: false
    t.boolean :done, default: false
    t.timestamps
  end
end

class Task < ActiveRecord::Base
  validates :title, presence: true
end

class TaskAPI < Sinatra::Base
  set :host_authorization, { permitted_hosts: [] }

  before do
    content_type :json
  end

  get '/tasks' do
    Task.all.to_json
  end

  post '/tasks' do
    data = JSON.parse(request.body.read) rescue {}
    task = Task.new(title: data['title'])
    if task.save
      status 201
      task.to_json
    else
      status 422
      { errors: task.errors.full_messages }.to_json
    end
  end

  get '/tasks/:id' do
    task = Task.find_by(id: params[:id])
    halt 404, { error: "not found" }.to_json unless task
    task.to_json
  end

  patch '/tasks/:id' do
    task = Task.find_by(id: params[:id])
    halt 404, { error: "not found" }.to_json unless task
    data = JSON.parse(request.body.read) rescue {}
    task.done = data['done'] if data.key?('done')
    task.save
    task.to_json
  end

  delete '/tasks/:id' do
    task = Task.find_by(id: params[:id])
    halt 404, { error: "not found" }.to_json unless task
    task.destroy
    status 204
  end
end
```

A few design choices worth calling out:

- **`before { content_type :json }`** sets the response header once for
  every route instead of repeating it five times.
- **`request.body.read` wrapped in a bare `rescue`** guards against a
  request with an empty or malformed body — `data` falls back to `{}`
  instead of the route raising a 500.
- **`halt 404, ...` before continuing** stops the block immediately, so
  the rest of the route never runs against a `nil` task.
- **`Task.new(title: data['title'])` then `.save`** relies on the model's
  own `validates :title, presence: true` — the route doesn't duplicate
  validation logic, it just checks the boolean result and reports
  `task.errors` on failure.

## spec/task_api_spec.rb — the request spec suite

```ruby
require 'rack/test'
require_relative '../lib/task_api'

RSpec.describe TaskAPI do
  include Rack::Test::Methods
  def app; TaskAPI; end

  before { Task.delete_all }

  def post_json(path, body)
    post path, body.to_json, { 'CONTENT_TYPE' => 'application/json' }
  end

  def patch_json(path, body)
    patch path, body.to_json, { 'CONTENT_TYPE' => 'application/json' }
  end

  it "creates and lists tasks" do
    post_json '/tasks', { title: "Write specs" }
    expect(last_response.status).to eq(201)

    get '/tasks'
    body = JSON.parse(last_response.body)
    expect(body.length).to eq(1)
    expect(body[0]["title"]).to eq("Write specs")
  end

  it "rejects a task with no title" do
    post_json '/tasks', {}
    expect(last_response.status).to eq(422)
  end

  it "marks a task done via PATCH" do
    post_json '/tasks', { title: "Ship it" }
    id = JSON.parse(last_response.body)["id"]

    patch_json "/tasks/#{id}", { done: true }
    expect(JSON.parse(last_response.body)["done"]).to eq(true)
  end

  it "404s for a missing task" do
    get '/tasks/999'
    expect(last_response.status).to eq(404)
  end

  it "deletes a task" do
    post_json '/tasks', { title: "Temp" }
    id = JSON.parse(last_response.body)["id"]
    delete "/tasks/#{id}"
    expect(last_response.status).to eq(204)
    get "/tasks/#{id}"
    expect(last_response.status).to eq(404)
  end
end
```

The `post_json`/`patch_json` helpers exist because `rack-test`'s default
`post`/`patch` send `application/x-www-form-urlencoded` bodies — sending
a raw JSON string without an explicit `CONTENT_TYPE` header is a common
mismatch that makes `request.body.read` return valid-looking JSON that
your app never actually parses correctly on the *client* side of the
test (Sinatra itself doesn't care about the header for `request.body`,
but it's good practice to send the header your route will document as
required, and some frameworks *do* branch on it).

## Running it

```text
$ ruby spec/task_api_spec.rb
-- create_table(:tasks)
   -> 0.0034s
.....

Finished in 0.02152 seconds (files took 0.42981 seconds to load)
5 examples, 0 failures
```

All five specs pass: list, create, validation-rejection, PATCH, and
DELETE-then-404. `before { Task.delete_all }` resets the in-memory table
between examples so specs don't leak state into each other — without it,
"creates and lists tasks" would see tasks left behind by earlier
examples once the suite grows.

## What this project demonstrates from Level 3

- **Module 1 (Sinatra)**: routes, `params`, `halt`, modular `Sinatra::Base`
  style (needed here specifically so `rack-test` can target `TaskAPI`
  rather than the single global `Sinatra::Application`).
- **Module 2 (ActiveRecord)**: a model with validations, `find_by`
  returning `nil` instead of raising, `.to_json` for serialization.
- **Module 3 (Testing Advanced)**: a full request-spec suite exercising
  the HTTP layer end-to-end rather than mocking pieces out.

## Stretch goals

1. Add a `GET /tasks?done=true` filter that returns only completed tasks,
   with a request spec covering both `true` and `false` values.
2. Add pagination: `GET /tasks?page=2&per_page=5`, returning a JSON
   envelope `{ "tasks": [...], "page": 2, "total": N }` instead of a bare
   array — update every existing spec's body-parsing accordingly.
3. Extract the JSON error-handling boilerplate (`halt 404, { error:
   ... }.to_json`) into a small helper method shared across routes, and
   add a Sinatra `error 500` handler that returns `{ "error": "internal
   server error" }.to_json` instead of Sinatra's default HTML error
   page.
4. Swap the in-memory SQLite database for a file-based one
   (`database: "tasks.db"`) and add a request spec proving data survives
   across two separately-instantiated `TaskAPI` requests within the same
   process — then explain in a comment why this still wouldn't survive
   across separate `ruby` process runs without also removing
   `ActiveRecord::Schema.define` re-running `create_table` on every
   boot (hint: use `create_table ... unless ActiveRecord::Base.connection.table_exists?(:tasks)`).
